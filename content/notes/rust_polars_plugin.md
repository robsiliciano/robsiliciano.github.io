+++
title = "Polars Plugins with Rust"
date = 2026-01-03
+++

# Plugins for Polars

I first started writing Rust extensions for polars during this year's [Advent of Code](https://adventofcode.com), where I challenged myself to exclusively use [polars' lazy API](https://docs.pola.rs/user-guide/concepts/lazy-api/). The restriction wasn't anywhere near as bad as I thought: I was worried about problems that used grids, but just joining two `LazyFrame`s worked for most of them.

I got through most of the days before I ran into my main obstacle, the lack of a `while` loop. I felt like `collect`ing early and evaluating a condition wouldn't count, but I remembered that polars can be [extended in Rust](https://docs.pola.rs/api/python/stable/reference/plugins.html). The Rust-written functions fit in the computation graph and wouldn't require more than the single `collect` at the end of the script. I decided to write try Rust extensions to fill in the few days that I couldn't get in pure `polars`.

## Polars Plugin Guides

The best resource on writing a polars plugin is this [extensive tutorial](https://marcogorelli.github.io/polars-plugins-tutorial/) by Marco Gorelli, a developer of polars and pandas. There's also the [official polars user guide](https://docs.pola.rs/user-guide/plugins/expr_plugins/)'s section on writing a plugin. Otherwise, I didn't find a ton of resources on writing plugins, which seem like an underutilized part of polars.

# Project Setup

I'm going to walk through a plugin to compute factorials. I'll set the project up manually, but the practical way would be through this [template](https://github.com/MarcoGorelli/cookiecutter-polars-plugins).

## Maturin

Any time you want to use Rust code from Python, you're going to use [PyO3](https://pyo3.rs) and its build system, [Maturin](https://www.maturin.rs). You can also use Maturin with binding other than PyO3, but that's not going to be relevant for a polars extension.

Maturin can initialize a Python-Rust project for you, but there's a trick to avoid a Catch-22. Unfortunately, Maturin doesn't have an automated setup [on top of existing projects](https://pyo3.rs/v0.27.2/getting-started.html#adding-to-an-existing-project), so you can't use it with your project's `uv` environment. However, you do need an environment in which to install Maturin, before you can run `maturin new`.  Fortunately, you don't have to do something weird like use use system Python, because you can just run it as a tool with [`uvx`](https://docs.astral.sh/uv/guides/tools/)!

```sh
uvx maturin new --mixed -b pyo3 polars-number
```

There are [a couple ways](https://www.maturin.rs/project_layout.html#mixed-rustpython-project) to structure a Rust and Python project, but I've used `--mixed` to keep the Rust code in `src` and the Python code in `python/polars-number`.

## Maturin and uv

If you try to use `uv` commands now, you'll notice that it's not rebuilding the Rust code! We'll have to tell `uv` to [rebuild when the Rust code changes](https://github.com/PyO3/maturin/issues/2314), though you can use [other options](https://quanttype.net/posts/2025-09-12-uv-and-maturin.html) like manually building the Rust code each time.

To have `uv` monitor the Rust side of things, add the following to your `pyproject.toml`:
```toml,name=pyproject.toml
[tool.uv]
cache-keys = [{file = "pyproject.toml"}, {file = "Cargo.toml"}, {file = "**/*.rs"}]
```

# Getting Started with PyO3

Before getting to the polars plugin, let's just write a function in Rust that can be called with Python.

```rust,name=src/lib.rs
use pyo3::prelude::*;

#[pymodule]
mod polars_number {
    use pyo3::prelude::*;

    #[pyfunction]
    fn factorial(number: u64) -> PyResult<u64> {
        Ok((1..number + 1).fold(1, |result, v| result * v))
    }
}
```

The error handling isn't clean, but this code is just a test.

We can try it out with a simple script:

```py,name=run.py
from polars_number import factorial

if __name__ == "__main__":
    for n in range(10):
        print(f"{n}! = {factorial(n)}")
```

Run it with `uv run run.py` and you should see factorials after some build info.

```
0! = 1
1! = 1
2! = 2
3! = 6
4! = 24
5! = 120
6! = 720
7! = 5040
8! = 40320
9! = 362880
```

# Polars Extensions

To get this code into a polars extension, we'll need two more crates: `polars` to use polars types, and `pyo3-polars` to bind our Rust code as a polars extension.

```sh
cargo add polars
cargo add pyo3-polars --features derive
```

Now that we have the binding for polars, let's change our factorial code from a regular Python binding to a polars binding. We're going to drop the `pymodule`, and swap the `pyfunction` for a `polars_expr`. With this last swap, we no longer know the exact return type from the Rust return type, since polars' `Series` could hold many things. We'll have to set `output_type=UInt64`, but be careful: this is not going to be verified by the Rust compiler.

The polars plugin is passed series, and we'll just apply the factorial function on each element, converting into a chunked array then back into a series.

```rust,name=factorial.rs
use polars::prelude::*;
use pyo3_polars::derive::polars_expr;

fn factorial_helper(number: u64) -> u64 {
    (1u64..number + 1).fold(1u64, |result, v| result * v)
}

#[polars_expr(output_type=UInt64)]
fn factorial(input: &[Series]) -> PolarsResult<Series> {
    let chunks = &input[0].u64()?;
    let s = chunks.apply(|v| Some(factorial_helper(v.unwrap()))).into_series();
    Ok(s)
}
```

We'll also need some Python code to [register this function](https://docs.pola.rs/api/python/stable/reference/api/polars.plugins.register_plugin_function.html) as a plugin. We can provide a path, which polars will search for the Rust-based library. Maturin should compile the library into that directory automatically for us (but it should already be covered by your generated `.gitignore`).

```py,name=python/polars_number/__init__.py
from pathlib import Path

import polars as pl
from polars.plugins import register_plugin_function
from polars._typing import IntoExpr

PLUGIN_PATH = Path(__file__).parent

def factorial(expr: IntoExpr) -> pl.Expr:
    return register_plugin_function(
        plugin_path=PLUGIN_PATH,
        function_name="factorial",
        args=expr,
        is_elementwise=True,
    )
```

Finally, let's update the test script `run.py` to use polars:

```py,name=run.py
import polars as pl

from polars_number import factorial

if __name__ == "__main__":
    s = pl.DataFrame({"n": range(10)}).with_columns(
        pl.col("n").cast(pl.UInt64)
    )

    s = s.with_columns(factorial=factorial("n"))

    print(s)
```

Check out the cast to `UInt64`. The Rust code assumes that the input series has that type and will error with anything else. If we didn't cast, then `"n"` would be a signed integer. You can try casting to `UInt32`, aka `u32`, and the code will also fail.

If all goes well, you should see the same output as before, but now in a polars data frame.

```
shape: (10, 2)
┌─────┬───────────┐
│ n   ┆ factorial │
│ --- ┆ ---       │
│ u64 ┆ u64       │
╞═════╪═══════════╡
│ 0   ┆ 1         │
│ 1   ┆ 1         │
│ 2   ┆ 2         │
│ 3   ┆ 6         │
│ 4   ┆ 24        │
│ 5   ┆ 120       │
│ 6   ┆ 720       │
│ 7   ┆ 5040      │
│ 8   ┆ 40320     │
│ 9   ┆ 362880    │
└─────┴───────────┘
```

# Handling Null Values

In polars, every value could be null, and your plugin should be able to handle them without crashing.

Here's a modified `run.py` script where one of the values is null.

```py
import polars as pl

from polars_number import factorial

if __name__ == "__main__":
    s = pl.DataFrame({"n": range(10)}).with_columns(
        pl.col("n").cast(pl.UInt64).replace(1, None)
    )

    s = s.with_columns(factorial=factorial("n"))

    print(s)
```

If you run this, you should see the script crash, with the plugin panicking! Polars is giving us an `Option<u64>`, not just a `u64`, and we're `unwrap`ing it. That risky behavior was fine in the case when we didn't have any nulls, but will cost us when we do.

Instead of unwrapping, we should have used [`map`](https://doc.rust-lang.org/std/option/enum.Option.html#method.map) to only apply the factorial to the non-null values.

```rust,name=src/factorial.rs
use polars::prelude::*;
use pyo3_polars::derive::polars_expr;

fn factorial_helper(number: u64) -> u64 {
    (1u64..number + 1).fold(1u64, |result, v| result * v)
}

#[polars_expr(output_type=UInt64)]
fn factorial(input: &[Series]) -> PolarsResult<Series> {
    let chunks = &input[0].u64()?;
    let s = chunks.apply(|v_opt| v_opt.map(|v| factorial_helper(v))).into_series();
    Ok(s)
}
```

Now, you should get an output where all the null values are propagated.

```
shape: (10, 2)
┌──────┬───────────┐
│ n    ┆ factorial │
│ ---  ┆ ---       │
│ u64  ┆ u64       │
╞══════╪═══════════╡
│ 0    ┆ 1         │
│ null ┆ null      │
│ 2    ┆ 2         │
│ 3    ┆ 6         │
│ 4    ┆ 24        │
│ 5    ┆ 120       │
│ 6    ┆ 720       │
│ 7    ┆ 5040      │
│ 8    ┆ 40320     │
│ 9    ┆ 362880    │
└──────┴───────────┘
```

# Handling More Types

So far, our code refuses to accept anything but a 64-bit unsigned integer (`UInt64`/`u64`). Anything else, like a lower-precision unsigned integer, will cause a crash.

```py,name=run.py
import polars as pl

from polars_number import factorial

if __name__ == "__main__":
    s = pl.DataFrame(
        {"n": range(10)},
        schema={"n", pl.UInt32},
    )

    s = s.with_columns(factorial=factorial("n"))

    print(s)
```

Fortunately, we can check the series data type before converting and run different logic for each case. If you change the function to the following code, you'll be able to run the example for `UInt32`/`u32`.

```rust,name=factorial.rs
use polars::prelude::*;
use pyo3_polars::derive::polars_expr;

#[polars_expr(output_type=UInt64)]
fn factorial(input: &[Series]) -> PolarsResult<Series> {
    let number = &input[0];

    match number.dtype() {
        DataType::UInt64 => {
            let chunks = &number.u64()?;
            let s = chunks
                .apply(|v_opt| v_opt.map(|v| (1..v + 1).fold(1, |result, i| result * i)))
                .into_series();
            Ok(s)
        }
        DataType::UInt32 => {
            let chunks = &number.u32()?;
            let s = chunks
                .apply(|v_opt| v_opt.map(|v| (1..v + 1).fold(1, |result, i| result * i)))
                .into_series();
            Ok(s)
        }
        DataType::UInt16 => {
            let chunks = &number.u16()?;
            let s = chunks
                .apply(|v_opt| v_opt.map(|v| (1..v + 1).fold(1, |result, i| result * i)))
                .into_series();
            Ok(s)
        }
        DataType::UInt8 => {
            let chunks = &number.u8()?;
            let s = chunks
                .apply(|v_opt| v_opt.map(|v| (1..v + 1).fold(1, |result, i| result * i)))
                .into_series();
            Ok(s)
        }
        dtype => {
            polars_bail!(
                InvalidOperation:
                "Unsupported type `{}` for series with name `{}`",
                dtype.to_string(), number.name()
            )
        }
    }
}
```

Note that I've set an `InvalidOperation` error here, but polars will [convert any type of error](https://github.com/pola-rs/polars/blob/c2412600210a21143835c9dfcb0a9182f462b619/crates/polars-plan/src/plans/aexpr/function_expr/plugin.rs#L137) to a `ComputeError`.

# Concluding Thoughts

## Is It Faster?

Things written in Rust have a reputation for being fast. I tested the Rust implementation against a simple Python implementation of the same algorithm, and in an entirely predictable outcome, Rust was faster.

Though if you really wanted a fast factorial, you'd use more sophisticated methods and cache across elements in the series.

```py
from time import time

import polars as pl

from polars_number import factorial


def py_factorial(n: int) -> int:
    result = 1
    for i in range(n):
        result *= i + 1
    return result


if __name__ == "__main__":
    s = pl.DataFrame(
        {"n": range(10)},
        schema={"n": pl.UInt64},
    )

    N = 1_000_000

    start = time()
    for _ in range(N):
        x = s.with_columns(factorial=factorial("n"))
    rust_time = time() - start
    print(f"Rust:   {rust_time:.1f}us")

    start = time()
    for _ in range(N):
        x = s.with_columns(factorial=pl.col("n").map_elements(py_factorial))
    python_time = time() - start
    print(f"Python: {python_time:.1f}us")
```

## Is This Useful?

The `factorial` function is almost certainly useless on the fixed-size integers usually used in polars. There's only about twenty numbers whose factorials could be represented, so I doubt the code here is of any direct use.

However, Rust plugins were a nice way to handle complicated data problems in Advent of Code, either by simplifying what I would do with polars magic or covering what I couldn't do at all in polars.
