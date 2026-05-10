+++
title = "Streaming Operations with PyArrow"
date = 2026-05-09
description = "Streaming operations are a great way to process a large data file without holding it all in memory at once. Sometimes you can run into a streaming operation that doesn't have a pre-built implementation in tools like polars. Instead, you can use Apache arrow directly to write your own."
+++

One of my favorite things about [polars' lazy API](https://docs.pola.rs/user-guide/lazy/using/) is how you can minimize the data stored in memory when working with large datasets. If you're careful, you can handle datasets that fit on disk but not in memory, without having to resort to something more complex like distributed computing.

However, polars doesn't support all [streaming operations](https://docs.pola.rs/user-guide/concepts/streaming/) that you might want to do. When you run into one of these more custom cases, do you have to put everything in memory or distribute it across Spark clusters? Fortunately, you can use the same tools as polars: the [Apache Arrow](https://arrow.apache.org/docs/index.html) format, which is the in-memory analogue to the parquet file format.

Here's a quick example: let's say you have some large dataset and you want to extract a subsample that retains part of its structure. Specifically, the dataset consists of many groups, each group has a number of IDs, and there are many rows per ID. You'd like to get ten IDs from each group and all the rows corresponding to those IDs.

There's no clean way to do this in a single, low-memory pass with polars, so let's try using arrow libraries directly.

# Sample Data

Let's start by creating some sample data. Here's a quick Python program to make a moderately-sized parquet file. There are 1,000 groups, 1,000 IDs per group, and 1,000 rows per ID.

```py,name=create_sample.py
import polars as pl

if __name__ == "__main__":
    (
        pl.DataFrame({"group": range(1_000)})
        .join(pl.DataFrame({"id": range(1_000)}), how="cross")
        .join(pl.DataFrame({"row": range(1_000)}), how="cross")
        .write_parquet("data.parquet")
    )
```

Now you should have a parquet file with a billion rows! You can go bigger if you want to push this streaming program further. I chose a relatively small file so that the examples run in a somewhat reasonable amount of time. Note that streaming helps with memory use, but you still have the time cost to process a lot of data!

# Streaming in Python

We'll use [PyArrow](https://arrow.apache.org/docs/python/index.html) to read parquet files and work with Arrow in memory.

```py
from collections import defaultdict

from pyarrow.parquet import ParquetFile, ParquetWriter
from pyarrow import RecordBatch


if __name__ == "__main__":
    data = ParquetFile("data.parquet")
    schema = data.schema_arrow

    first10 = defaultdict(list)

    with ParquetWriter("output.parquet", schema) as writer:
        for batch in data.iter_batches(batch_size=1024):
            groups = []
            ids = []
            rows = []

            for group, id, row in zip(
                batch.column("group").to_pylist(),
                batch.column("id").to_pylist(),
                batch.column("row").to_pylist(),
            ):
                if len(first10[group]) < 10 and id not in first10[group]:
                    first10[group].append(id)

                if id in first10[group]:
                    groups.append(group)
                    ids.append(id)
                    rows.append(row)

            if groups:
                writer.write_batch(
                    RecordBatch.from_pydict(
                        {
                            "group": groups,
                            "id": ids,
                            "row": rows,
                        },
                        schema
                    )
                )
```

This quick program makes use of [`ParquetFile`](https://arrow.apache.org/docs/python/generated/pyarrow.parquet.ParquetFile.html) to only read small batches at a time. We can do whatever we want with them, then write them with the [`ParquetWriter`](https://arrow.apache.org/docs/python/generated/pyarrow.parquet.ParquetWriter.html) before we get the next batch.

Note that the code here isn't super efficient. For one thing, it copies the batch data when it could [filter](https://arrow.apache.org/docs/python/generated/pyarrow.compute.filter.html), although that would require more changes.

## Compression

If you look at file sizes, you'll see that the `output.parquet` is way larger than 1% of `data.parquet`, despite the fact that it should take exactly 1% of the rows, by construction.

Parquet files can get much smaller than an equivalent CSV file through compression. There are several [supported compression algorithms](https://parquet.apache.org/docs/file-format/data-pages/compression/), which can differ in their size reduction and speed, depending on what's most important to you.

You can pass one of these algorithms to `ParquetWriter` to change off the default snappy algorithm.

```py
with ParquetWriter("output.parquet", schema, compression="zstd") as writer:
```

However, we're not seeing the larger file size because we chose the wrong algorithm (or an algorithm that prioritizes speed over size). We've accidentally made a parquet file that's harder to compress, but we'll need to talk more about the internals of a parquet file first.

## Row Groups

Parquet files are usually described as a column-based file format, meaning data is stored next to data from that same column. In contrast, CSV is a row-based format where the entire row is stored together, before the next row.

Despite the ubiquity of the column-based characterization for parquet files, it's **slightly wrong**. Actually, a parquet file contains [row groups](https://parquet.apache.org/docs/concepts/) which are all the data for a set of rows. Only within a row group is the data stored by columns! Most data is stored between other data from the same column, but those columns may be broken into several nonconsecutive chunks, each in their own row group.

The number of row groups in a parquet file is a [configuration choice](https://parquet.apache.org/docs/file-format/configurations/). You can check how many row groups are in a file with [`ParquetFile.num_row_groups`](https://arrow.apache.org/docs/python/generated/pyarrow.parquet.ParquetFile.html#pyarrow.parquet.ParquetFile.num_row_groups).

```py,name=count_row_groups.py
from argparse import ArgumentParser
from pathlib import Path

from pyarrow.parquet import ParquetFile

if __name__ == "__main__":
    parser = ArgumentParser()
    parser.add_argument("file", type=Path, nargs="+")
    args = parser.parse_args()

    for path in args.file:
        data = ParquetFile(path)
        print(f"{path}: {data.num_row_groups}")
```

You can run this for the input and output data, and you should see a large number of row groups on the input data, but a _much_ larger number of row groups for our output data. I ran into 10,750 row groups in the output file, despite only having 10,000,000 rows. Despite having less data in the output, we chose much, much smaller row groups to write.

Row groups provide a limit on compression, since compression happens on the columns _within_ a row group. (Technically on a [finer granularity called "pages"](https://parquet.apache.org/docs/concepts/) that compose the column chunk.) If we have tiny row groups, then the compression algorithm can't do much on each individual page in the column chunk.

## The Fix

We'll have to fix it by building up larger row groups before we write.

```py
from collections import defaultdict

from pyarrow.parquet import ParquetFile, ParquetWriter
from pyarrow import RecordBatch

if __name__ == "__main__":
    data = ParquetFile("data.parquet")
    schema = data.schema_arrow

    first10 = defaultdict(list)

    with ParquetWriter("output.parquet", schema) as writer:
        groups = []
        ids = []
        rows = []

        for batch in data.iter_batches(batch_size=1024):
            for group, id, row in zip(
                batch.column("group").to_pylist(),
                batch.column("id").to_pylist(),
                batch.column("row").to_pylist(),
            ):
                if len(first10[group]) < 10 and id not in first10[group]:
                    first10[group].append(id)

                if id in first10[group]:
                    groups.append(group)
                    ids.append(id)
                    rows.append(row)

            if len(groups) >= 100_000:
                writer.write_batch(
                    RecordBatch.from_pydict(
                        {
                            "group": groups,
                            "id": ids,
                            "row": rows,
                        },
                        schema,
                    )
                )

                groups = []
                ids = []
                rows = []

        if len(groups) > 0:
            writer.write_batch(
                RecordBatch.from_pydict(
                    {
                        "group": groups,
                        "id": ids,
                        "row": rows,
                    },
                    schema,
                )
            )
```

Here, I only write a row group once it reaches a hundred thousand rows. You can reduce that to save memory, at the expense of file size.

# Streaming in Rust

If we really need to go faster, why not try Rust rather than Python? We can use the official [arrow](https://crates.io/crates/arrow) and [parquet](https://crates.io/crates/parquet) crates.

The code is mostly the same, but with all the Rust stuff like types, result handling, and care for the borrow checker.

```rs
use std::collections::HashMap;
use std::fs::File;
use std::sync::Arc;

use arrow::array::{Int64Array, RecordBatch, RecordBatchReader};
use parquet::arrow::ArrowWriter;
use parquet::arrow::arrow_reader::ParquetRecordBatchReaderBuilder;
use parquet::basic::Compression;
use parquet::file::properties::WriterProperties;

const BATCH_SIZE: usize = 1_024;
const FLUSH: usize = 100_000;

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let file = File::open("data.parquet")?;
    let reader = ParquetRecordBatchReaderBuilder::try_new(file)?
        .with_batch_size(BATCH_SIZE)
        .build()?;
    let schema = reader.schema();

    let mut first10: HashMap<i64, Vec<i64>> = HashMap::new();

    let output_file = File::create("output.parquet")?;

    let props = WriterProperties::builder()
        .set_compression(Compression::SNAPPY)
        .build();
    let mut writer = ArrowWriter::try_new(output_file, schema.clone(), Some(props))?;

    let mut groups = Vec::with_capacity(FLUSH + BATCH_SIZE);
    let mut ids = Vec::with_capacity(FLUSH + BATCH_SIZE);
    let mut rows = Vec::with_capacity(FLUSH + BATCH_SIZE);

    for batch in reader {
        let batch = batch?;
        let batch_groups = batch
            .column(0)
            .as_any()
            .downcast_ref::<Int64Array>()
            .ok_or("Missing i64 group column.")?;
        let batch_ids = batch
            .column(1)
            .as_any()
            .downcast_ref::<Int64Array>()
            .ok_or("Missing i64 ID column.")?;
        let batch_rows = batch
            .column(2)
            .as_any()
            .downcast_ref::<Int64Array>()
            .ok_or("Missing i64 row column.")?;

        for ((group, id), row) in batch_groups
            .iter()
            .zip(batch_ids.iter())
            .zip(batch_rows.iter())
        {
            let group = group.unwrap();
            let id = id.unwrap();
            let row = row.unwrap();

            let contained = if let Some(group_ids) = first10.get_mut(&group) {
                if group_ids.contains(&id) {
                    true
                } else if group_ids.len() < 10 {
                    group_ids.push(id);
                    true
                } else {
                    false
                }
            } else {
                first10.insert(group, vec![id]);
                true
            };

            if contained {
                groups.push(group);
                ids.push(id);
                rows.push(row);

                if groups.len() >= FLUSH {
                    writer.write(&RecordBatch::try_new(
                        schema.clone(),
                        vec![
                            Arc::new(Int64Array::from(groups)),
                            Arc::new(Int64Array::from(ids)),
                            Arc::new(Int64Array::from(rows)),
                        ],
                    )?)?;
                    groups = Vec::with_capacity(FLUSH + BATCH_SIZE);
                    ids = Vec::with_capacity(FLUSH + BATCH_SIZE);
                    rows = Vec::with_capacity(FLUSH + BATCH_SIZE);
                }
            }
        }
    }

    if !groups.is_empty() {
        writer.write(&RecordBatch::try_new(
            schema,
            vec![
                Arc::new(Int64Array::from(groups)),
                Arc::new(Int64Array::from(ids)),
                Arc::new(Int64Array::from(rows)),
            ],
        )?)?;
    }

    writer.close()?;

    Ok(())
}
```

Did this program not run much faster than the Python version? Remember to run with the `--release` flag. Otherwise you'll be in debug mode and leave a lot of the optimizations on the table.

```sh
cargo run --release
```

The Rust code should run 50x faster than the equivalent Python version, not counting the compile time. Even with the one-time compile time cost, Rust is still faster than Python! The compile time dwarfs the runtime on this dataset, but it doesn't grow with dataset size. On the much larger datasets where in-memory operations aren't available, the actual runtime will dominate and Rust will look fifty times faster.

So Rust or Python? Python introduces a lot of overhead, both in terms of memory and per-batch processing time. For huge parquet files in limited memory, this may not be a sacrifice worth making.

However, you need to be careful with types in Rust. The code here will exit if the data isn't `i64` when it tries to create an `Int64Array`.

Also worth noting that columns in parquet and arrow are _nullable_: values can be an integer or they can be null. Both programs here assume no nulls, which might not be safe in all contexts. Rust will panic on the `unwrap`s while Python will try to use `None` as a value.
