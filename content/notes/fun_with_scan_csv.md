+++
title = "Fun with scan_csv"
date = 2026-01-04
+++

This year, I decided to do [Advent of Code](https://adventofcode.com/2025/day/12) using only [polars](https://pola.rs), the dataframe library. While polars is great for many things, it's not meant to do weird things like Advent of Code puzzles.

The hackiest part was always reading the puzzle inputs, which are just puzzle-specific text. polars, as a dataframe library, only focuses on reading data files like parquets and CSVs. While fitting random text into a parquet format was out of the question, I was able to hack all the inputs using CSV processing. For better, but mostly for worse, CSV file formatting is a very flexible and ill-defined in ways that needs a lot of options to read.

These are my notes on how I used [`scan_csv`](https://docs.pola.rs/api/python/stable/reference/api/polars.scan_csv.html) to process data that was never intended to go into data frames. Please do not use these hacks in a serious setting!

# Head(er)less

A well-formed CSV file should have a header row with the names of all the columns. The CSV reader would, by default, expect a header and take the first row of our input as column names. Since the input is nowhere near a CSV, we'll have to tell the reader that it shouldn't expect a header with the `has_header=False` option.

A couple days, 8 and 9 to be exact, otherwise passed in nice, comma-separated data with a fixed number of columns. polars will name these columns `column_1`, `column_2`, etc. if you don't have a header. Otherwise, you can give specific names with the `new_columns` argument.

# Going Beyond Commas

Sometimes, inputs had pieces of information separated by things other than commas. For instance, day 11 has input rows like `input: outputs` where two pieces of information are separated by a colon. You can change the separate from commas to any single byte using the `separator` argument.

My ugliest use of the custom separator was on day 2, where the input was a comma-separated list of ranges like `10-20,30-40`. I could read this as a headerless CSV, but then I'd have a single row, a large and unknown number of columns, and string values that couldn't be converted to a numeric primative. Here, I got a much nicer read by switching the separator to `-` and telling the parser to interpret commas as newlines with `eol_char=","`. The resulting data was a nice shape, with two numeric columns, one for the range start and one for the range end, where each row represented an interval.

# Grids

A lot of Advent of Code inputs are grids, which are oddly weird for data frames. You'd expect that the rectangular shape would fit well, but the variable number of columns are awkward and columns aren't the easiest to use for spatial operations. Instead, I preferred a long-format dataframe where I had three columns: the x coordinate, the y coordinate, and the value.

I read grid data in four steps:

1. Read as a headerless CSV, making sure that commas don't appear in the input. You'll get one column in total, and each row will represent one line of the input.
2. Assign a y coordinate to each row with [`int_range`](https://docs.pola.rs/api/python/dev/reference/expressions/api/polars.int_range.html), [`cum_count`](https://docs.pola.rs/api/python/dev/reference/expressions/api/polars.Expr.cum_count.html) or the unstable [`row_index`](https://docs.pola.rs/api/python/dev/reference/expressions/api/polars.row_index.html).
3. Split the row into a list of each column with `pl.col("column_1").str.split("")`. The splitting function ['str.split'](https://docs.pola.rs/api/python/dev/reference/expressions/api/polars.Expr.str.split.html) can take an empty string, which tells it to split each character.
4. Switch from rows of lists to have each row be a single element from the list with the [`explode`](https://docs.pola.rs/api/python/dev/reference/dataframe/api/polars.DataFrame.explode.html) operation. Any unexploded columns, like the y coordinate, will be copied to each element in the row.
5. Add x coordinates by the same `int_range`/`cum_count`/`row_index` approach as for y coordinates, expect this time as a window function over the y coordinate.

The full code might look like:
```py
df = (
    pl.scan_csv(puzle_input, has_header=False)
    .select(pl.col("column_1").str.split(""), y=pl.row_index())
    .explode("column_1")
    .with_columns(x=pl.row_index().over("y"))
)
```

Parsing a column as a list then exploding was a good trick for dealing with variable-length data in several other days.

# Empty Rows, Short Rows, and Nulls

The last major challenge reading input is when the puzzle input contains two distinct sections. Day 5 is a great example, where the input has a first section of ranges, then a blank line, and finally single numbers. The first part could be nicely read as a two-column dataframe with a `separator="-"`, and the second part could be read as a one-column dataframe. However, I had to read both at once, while also handling the blank line.

Fortunately, `scan_csv` will put in null values for any missing columns in a row. If you have an empty row, then it'll be read as nulls for each of your columns. I read the input as a CSV with `-` as the separator, so the first section came out as nice rows with two non-null values, the empty line came out as a row with two null values, and the final section came out as rows with only one non-null value. Filtering on the null columns separated out the two sections into separate `LazyFrame`s.
