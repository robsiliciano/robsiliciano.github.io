+++
title = "Polars: Chunks and Speed"
date = 2026-03-07
description = "This note looks under the hood of polars dataframes, where data is stored in the Apache Arrow format as an array of data chunks. These chunks connect to polars threading and caching, so operations on the same data can take vastly different amounts of time depending on how many chunks are used."
+++

# Is Polars or NumPy Faster?

What's faster: polars' [`sqrt`](https://docs.pola.rs/api/python/stable/reference/expressions/api/polars.Expr.sqrt.html) or converting to NumPy and back? Here's a quick script to check.

```py
from time import time

import numpy as np
import polars as pl
from polars.testing import assert_frame_equal


def numpy_sqrt(df: pl.DataFrame, col: str):
    return pl.DataFrame({col: np.sqrt(df[col].to_numpy())})


def polars_sqrt(df: pl.DataFrame, col: str):
    return df.select(pl.col(col).sqrt())


if __name__ == "__main__":
    N = 1_000
    df = pl.DataFrame({"x": np.linspace(0, 10_000, 100_000_000)})

    start = time()
    for _ in range(N):
        sqrt_n = numpy_sqrt(df, "x")
    numpy_time = time() - start

    print(f"NumPy:   {numpy_time / N:.1e}s")

    start = time()
    for _ in range(N):
        sqrt_p = polars_sqrt(df, "x")
    polars_time = time() - start

    print(f"Polars:  {polars_time / N:.1e}s")

    assert_frame_equal(sqrt_n, sqrt_p, check_exact=True)
```

Note that I'm using a [testing function](https://docs.pola.rs/api/python/stable/reference/api/polars.testing.assert_frame_equal.html) to make sure neither implementation is cutting corners.

It looks like NumPy is _much_ faster than the direct implementation! Around 3x faster.

```
NumPy:   3.2e-03s
Polars:  9.8e-03s
```

At these times, should we just convert to NumPy for all math operations? I would be shocked, since one of polars' claims to speed was the use of Arrow rather than NumPy under the hood. In fact, pandas has even [done the same](https://pandas.pydata.org/docs/user_guide/pyarrow.html) in recent versions.

# Sorting it Out

Perhaps I chose a weird input dataset or hit some odd bug? The data I chose had the rare property that it comes in order. I know from some previous work on polars that it has optimizations for sorted data; sometimes, polars will maintain [sorted flags](https://docs.pola.rs/api/python/stable/reference/series/api/polars.Series.flags.html) that can enable optimized versions of certain functions. Perhaps I hit a bug in one of these functions.

Instead of checking for a special `sqrt` on sorted data (how weird) I decided to unsort the data by randomly shuffling it. I just added a single line after creating the dataframe:
```py
    df = df.sample(len(df))
```

Now, running the script gave a _different_ surprise!

```
NumPy:   1.2e-02s
Polars:  1.5e-03s
```

Yes, Polars is faster than NumPy, but the really interesting thing is that Polars on the shuffled data is faster than Polars on the ordered data. A bit less interesting is that NumPy is much slower on the shuffled data.

## Why is NumPy Slower on Shuffled Data?

Let's start with the less interesting question: why was NumPy four times slower after I shuffled the data?

The answer comes down to the [NumPy conversion](https://docs.pola.rs/api/python/stable/reference/dataframe/api/polars.DataFrame.to_numpy.html) and whether or not it has to copy data. If possible, polars tries to avoid copying the underlying data in the conversion. If the conditions are met, then polars can just show NumPy a read-only view of the data, and the operation is purely the NumPy square root.

We can quickly check if polars is copying the data by disallowing the copy. If polars needs to copy data for `to_numpy`, it will throw an error instead. Change the `numpy_sqrt` to set `allow_copy=False`.

```py
def numpy_sqrt(df: pl.DataFrame, col: str):
    return pl.DataFrame({col: np.sqrt(df[col].to_numpy(allow_copy=False))})
```

When you run it, the program should crash! Polars had to make a copy to complete the operation, but we didn't allow that. However, if you take out the shuffling line, the program works fine.

NumPy is slower after the shuffle because `to_numpy` has to make a full copy of all our data. Before the shuffle, we could just use the data as it was in memory, skipping this expensive step.

## Chunks!

I didn't fully answer why NumPy was slower: we still need to know why we have to make a copy in one case, but not the other.

Here's [the conditions](https://docs.pola.rs/api/python/stable/reference/dataframe/api/polars.DataFrame.to_numpy.html) for when `to_numpy` can avoid a copy:

> This operation copies data only when necessary. The conversion is zero copy when all of the following hold:
>
> The DataFrame is fully contiguous in memory, with all Series back-to-back and all Series consisting of a single chunk.
> The data type is an integer or float.
> The DataFrame contains no null values.
> The order parameter is set to fortran (default).
> The writable parameter is set to False (default).

In our case, the relevant condition is that the data has to be a single **chunk**, a [concept from Apache Arrow](https://arrow.apache.org/docs/r/reference/ChunkedArray-class.html). Remember, Arrow is the memory format backing polars dataframes. Essentially, Arrow allows a series to be stored in multiple chunks, rather than in one contiguous chunk that's harder to fit in memory.

If the data is in one contiguous chunk, then NumPy can understand that memory and read from it. However, if we've broken our dataframe into chunks, then NumPy won't understand why the data is all over. We _have_ to copy it to one location for NumPy to read it.

You can check how many chunks with [the `n_chunks` method](https://docs.pola.rs/api/python/stable/reference/dataframe/api/polars.DataFrame.n_chunks.html).

```py
import numpy as np
import polars as pl

if __name__ == "__main__":
    df = pl.DataFrame({"x": np.linspace(0, 10_000, 100_000_000)})
    print(f"Starting Chunks: {df.n_chunks()}")

    df = df.sample(len(df))
    print(f"Shuffled Chunks: {df.n_chunks()}")
```

The starting chunks should be only one, but the shuffled chunks should be higher! For instance:
```
Starting Chunks: 1
Shuffled Chunks: 10
```

We've split up the data in memory, which is fine for polars but incomprehensible to NumPy. Our NumPy code requires a costly copy to run.

_Note:_ I configured things up so that polars would default to 10 chunks for this example, but your default number can vary. It's fine if you see something else! I just think it's nice to have a round number for these examples. Later on, you'll see how I chose the number of chunks here to get to 10.

# Why is Polars Faster on Shuffled Data?

Now that we've covered chunks, we can get to why Polars took one sixth as long when the data was shuffled.

First, let's confirm that it's the chunks and not the data. We can always get a dataframe [back to a single chunk](https://docs.pola.rs/api/python/stable/reference/dataframe/api/polars.DataFrame.rechunk.html). If you add the line below, you should have the shuffled data, but only in a single chunk.

```py
    df = pl.DataFrame({"x": np.linspace(0, 10_000, 100_000_000)})
    df = df.sample(len(df))
    df = df.rechunk()
```

Running the script now should give the same times you initially saw, with the slower polars and faster NumPy.

```
NumPy:   3.2e-03s
Polars:  1.0e-02s
```

We could do the opposite as well: take the original data ordering and put it into chunks. One way to get multiple chunks is to [split the data](https://docs.pola.rs/api/python/stable/reference/dataframe/api/polars.DataFrame.iter_slices.html) in order, then tell polars [_not_ to rechunk when concatenating](https://docs.pola.rs/api/python/stable/reference/api/polars.concat.html) it.
```py
    df = pl.DataFrame({"x": np.linspace(0, 10_000, 100_000_000)})
    df = pl.concat(df.iter_slices(len(df) // 10), rechunk=False)
```

Run this, and you'll get back to the fast polars and slow NumPy, but on the original data. We can safely say that it's the _chunks_ that matter for speed, not the data itself.

```
NumPy:   1.2e-02s
Polars:  1.5e-03s
```

## Threading Polars

Polars can go faster by using multiple threads to process chunks. On a multi-CPU machine, polars can process multiple chunks at once. Even if polars is slower on individual chunks, it can make up for it by multitasking!

You can see polars slow down by turning off multithreading. Polars has a maximum number of threads that it will use, and you can check this maximum:
```py
print(f"Polars pool {pl.thread_pool_size()}")
```

Turn off multithreading by setting it to one.

```py
import os

os.environ["POLARS_MAX_THREADS"] = "1"
```

If you take the code with multiple chunks and add this line, you'll see both the slow NumPy and a slow polars (though not _as_ slow)!
```
NumPy:   1.2e-02s
Polars:  3.4e-03s
```

## Caching

Even at one thread, the multi-chunk polars is still noticeably faster than the single-chunk polars. With this memory stuff, you also need to worry about the size of your hardware cache. Sometimes the CPU is very good at processing small chunks of data, but fails when it needs to deal with large chunks and has "cache miss" problems.

As a quick, system-independent test, try varying the number of chunks you have while still forcing a single polars thread.

```py
import os
from time import time

import numpy as np
import polars as pl


def polars_sqrt(df: pl.DataFrame, col: str):
    return df.select(pl.col(col).sqrt())


os.environ["POLARS_MAX_THREADS"] = "1"

if __name__ == "__main__":
    N = 1_000
    df_base = pl.DataFrame({"x": np.linspace(0, 10_000, 104_144_040)})

    for chunks in range(2, 20):
        df = pl.concat(df_base.iter_slices(len(df_base) // (chunks - 1)), rechunk=False)

        start = time()
        for _ in range(N):
            sqrt_p = polars_sqrt(df, "x")
        polars_time = time() - start

        print(f"Chunks: {df.n_chunks()}\n\tTime:  {polars_time / N:.1e}s")
```

_Note_: I switched the data size slightly, increasing it by 4%, so that it is evenly divisible by most of the chunk sizes. I wasn't too careful with this code, so it's possible that `n_chunks` returns a different number than `chunks`. You'd see this at all the primes larger than 17.

```
Chunks: 1
	Time:  1.1e-02s
Chunks: 2
	Time:  1.1e-02s
Chunks: 3
	Time:  1.1e-02s
Chunks: 4
	Time:  1.1e-02s
Chunks: 5
	Time:  1.1e-02s
Chunks: 6
	Time:  1.1e-02s
Chunks: 7
	Time:  1.1e-02s
Chunks: 8
	Time:  1.1e-02s
Chunks: 9
	Time:  1.1e-02s
Chunks: 10
	Time:  3.7e-03s
Chunks: 11
	Time:  3.6e-03s
Chunks: 12
	Time:  3.7e-03s
Chunks: 13
	Time:  3.7e-03s
Chunks: 14
	Time:  3.6e-03s
Chunks: 15
	Time:  3.6e-03s
Chunks: 16
	Time:  3.7e-03s
Chunks: 17
	Time:  3.7e-03s
Chunks: 18
	Time:  3.7e-03s
Chunks: 20
	Time:  3.7e-03s
```

At only one chunk, we have the same slow speed as the first example, and at ten chunks, we have the slightly faster polars above. As the number of chunks increase, there's a **step change**: increasing the number of chunks shows no impact at first, then the run time suddenly flips from the slow polars to the less slow polars, then further chunks show no improvement. _That's_ the cache working!

# Chunking for ~Fun and Profit~ Speed

We can control the chunking to get our super fast polars on the original data. Here's a modified square root function that puts the dataframe into ten chunks before calling.

```py
def polars_sqrt(df: pl.DataFrame, col: str):
    df = pl.concat(df.iter_slices(len(df) // 10), rechunk=False)
    return df.select(pl.col(col).sqrt())
```

Run it and we get our fast NumPy and our _even faster_ polars.

```
NumPy:   3.3e-03s
Polars:  1.5e-03s
```

# Takeaways

Dataframes are an abstraction over the data in memory; polars makes that abstraction pretty good, avoiding copying whenever it can. However, there're still times when memory matters.

1. Take a look at `n_chunks` to know how your data is stored in memory.
2. Think about how many threads you want running and the capacity from your hardware.
3. Check the size of your data against your hardware's cache size.

If you're missing your cache or have CPU cores available, consider more chunks for some faster code.
