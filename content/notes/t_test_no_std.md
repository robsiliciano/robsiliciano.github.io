+++
title = "A \"No Standard\" T test"
date = 2026-06-28
description = "Most Rust statistics crates won't run on embedded targets, because the crates require the standard library but the microcontrollers don't support it. Instead, you'll need no_std stats code, which requires some care to get around missing std code."
[extra]
latex = true
+++

Rust offers a mode called `no_std` where it doesn't use most of the standard library, which is mainly useful if you want to run on an embedded or WASM target that doesn't have the usual type of operating system that backs a lot of the standard library. For example there's no filesystem on a microcontroller, so there's no way `std::fs` could work.

You'd think that `no_std` would be standard on computation crates, or at least an option that leaves out a couple bells and whistles. There's no need for a file system when calculating a probability, right? Not so! It turns out that most of the major stats and math crates don't work without `std`. The standard library provides a lot of the math backbones needed for anything remotely complicated.

Here's what it takes to implement a simple t-test in `no_std` code, which should illustrate why most math crates don't offer `no_std`, and how to get around those limitations if you must.

# Starting a no_std Library

It's pretty easy to start our `no_std` code: we just add `#![no_std]` to the top of the file.

```rs,name=lib.rs
#![no_std]

pub fn t_test(_data: &[f32], _mu: f32) -> f32 {
    todo!()
}
```

Make sure you include the `!` here, which makes the attribute an [inner attribute](https://doc.rust-lang.org/reference/attributes.html#r-attributes.inner) that modifies whatever it's in, which in this case is the module. The usual outer attribute modifies the thing that comes next, but we can't have a line of code before the file starts!

# The T Statistic

We may as well call `no_std` "no standard deviation" because that's where we'll immediately run into problems when trying to calculate the t-statistic. Specifically, Rust's [floating-point square root](https://doc.rust-lang.org/std/primitive.f32.html#method.sqrt) is in the `std` crate, so we lose it in the `no_std` world. We'll need another implementation of it to calculate the sample standard deviation.

Fortunately, there's the Rust `libm` crate, which implements a lot of the platform math that's in [`std`](https://github.com/rust-lang/compiler-builtins/tree/main/libm), including the square root.

```rs
use libm::sqrtf;

fn t_statistic(data: &[f32], mu: f32) -> f32 {
    let n = data.len() as f32;
    let mean = data.iter().sum::<f32>() / n;
    let std_dev = sqrtf(data.iter().map(|x| (x - mean) * (x - mean)).sum::<f32>() / (n - 1.0));
    (mean - mu) * sqrtf(n) / std_dev
}
```

Note the `f` suffix on `sqrtf`. I'm using 32-bit floats rather than 64-bit floats, and `libm` offers versions for both, distinguishing the `f32` versions with the suffix. The `libm` crate has to implement functions rather than methods, which means it has to worry about name collisions. As an external crate, `libm` can't implement methods on `f32` or `f64` (without having to bring a trait in scope).

# Student T Distribution

We need to calculate the CDF of a Student T distribution, which can be written with the regularized [Beta function](https://dlmf.nist.gov/5.12).

\\[ F(t) = 1 - \frac{1}{2 B(v/2, 1/2)} B\left( \frac{v}{t^2 + v}; \frac{v}{2}, \frac{1}{2} \right) \\]

Where the Beta function is:
\\[ B(a, b) = \int_0^1 t^{a - 1} (1-t)^{b - 1} \; dt  \\]

And the [_incomplete_ Beta function](https://dlmf.nist.gov/8.17) is:
\\[ B(x; a, b) = \int_0^x t^{a - 1} (1-t)^{b - 1} \; dt  \\]

Fortunately for us, the Beta function has a nice expression that we can get with `libm` using the Gamma function.

\\[ B(a, b) = \frac{\Gamma(a)\Gamma(b)}{\Gamma(a + b)}  \\]

We have the log of the gamma function, [`lgammaf`](https://docs.rs/libm/latest/libm/fn.lgammaf.html), then can get back to the Beta function with `libm`'s [`expf`](https://docs.rs/libm/latest/libm/fn.expf.html).

The hard part comes from calculating the incomplete Beta function, for which `libm` doesn't offer a convenient function or two.

```rs
use libm::{expf, lgammaf};

fn beta(a: f32, b: f32) -> f32 {
    expf(lgammaf(a) + lgammaf(b) - lgammaf(a + b))
}
```

## The Incomplete Beta Function

We'll have to compute the incomplete Beta function by hand, using the [continued fraction](https://dlmf.nist.gov/8.17#v) for it.

\\[ B(x; a, b) = \frac{x^a (1 - x)^b}{a} \frac{1}{1 + \frac{d_1}{1 + \frac{d_2}{1 + \dots}}} \\]

Where the coefficient formula depends on whether the index is even or odd.
\\[ d_{2m} = \frac{m(b - m)x}{(a + 2m)(a + 2m - 1)} \\]
\\[ d_{2m + 1} = -\frac{(a + m)(a + b + m)x}{(a + 2m + 1)(a + 2m)}  \\]

The continued fraction converges quickly for low levels of \\( x \\).
\\[ x < \frac{a + 1}{a + b + 2} \\]

Otherwise, we can just swap it around using the identity:

\\[ B(x; a, b) = B(a, b) - B(1 - x; b, a) \\]

Note the swapped \\( a \\) and \\( b \\) mean that we'll have the proper condition when we compute that second incomplete Beta.

\\[ 1 - x < 1 - \frac{a + 1}{a + b + 2} = \frac{b + 1}{b + a + 2} \\]

We can write that in Rust using only one function from `libm`.

```rs
use libm::powf;

fn incomplete_beta(x: f32, a: f32, b: f32) -> f32 {
    if x > (a + 1.0) / (a + b + 2.0) {
        beta(a, b) - incomplete_beta(1.0 - x, b, a)
    } else {
        const MAX_M: usize = 5;
        let mut ds = [0.0; 2 * MAX_M];

        for i in 0..MAX_M {
            let m = i as f32;
            ds[2 * i] = m * (b - m) * x
                / ((a + 2.0 * m) * (a + 2.0 * m - 1.0));
            ds[2 * i + 1] = -(a + m) * (a + b + m) * x
                / ((a + 2.0 * m + 1.0) * (a + 2.0 * m));
        }
        ds[0] = 1.0;
        ds.reverse();

        let continued_fraction = ds.into_iter().reduce(|acc, d| d / (1.0 + acc)).unwrap();

        continued_fraction * powf(x, a) * powf(1.0 - x, b) / a
    }
}
```

Note that you could write this more efficiently, like without the `reverse`, but I left it this way for readability.

## Testing the Beta

Okay, so we've written a lot of numerical computation code for a complicated function and could have easily messed something up. We really should add a test for this function.

The good news is that we don't have to make our tests `no_std`! We can test using the standard library without forcing our functions to use it. For example, here's a silly example of adding `std` to the test.

```rs
#[cfg(test)]
extern crate std;

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_incomplete_beta() {
        let a = 5.5;

        assert_eq!(a, 5.5);
    }
}
```

We can test the `no_std` code against other `std`-based statistics libraries, namely [`statrs`](https://github.com/statrs-dev/statrs). First, we'll need to add it as a [dev dependency](https://doc.rust-lang.org/rust-by-example/testing/dev_dependencies.html), so we can use it in testing but not have to have it to build the actual `no_std` code!

```sh
cargo add --dev statrs
```

Now, we're free to use `statrs` in the `tests` module and any test we want. Note that `statrs` gives us the [CDF of the Beta distribution](https://docs.rs/statrs/latest/statrs/distribution/struct.Beta.html), which is the _regularized_ incomplete beta function. We just need to divide by \\( B(a, b) \\). We'll actually use this regularized form too in the Student t distribution, so we'd want to test it anyways.

```rs
#[cfg(test)]
mod tests {
    use super::*;
    use statrs::distribution::{Beta, ContinuousCDF};

    #[test]
    fn test_incomplete_beta() {
        let a: f32 = 5.5;
        let b: f32 = 0.5;

        let beta_std = Beta::new(a as f64, b as f64).unwrap();

        for i in 0..=10 {
            let x = (i as f32) / 10.0;

            let computed = incomplete_beta(x, a, b) / beta(a, b);
            let reference = beta_std.cdf(x as f64) as f32;
            assert!((computed - reference).abs() < 1e-4);
        }
    }
}
```

Run `cargo test` and you should see that this `incomplete_beta` implementation is close.

# Putting it All Together

Now that we have the incomplete Beta working, let's just put everything together for a basic two-sided t test.

```rs
pub fn t_test(data: &[f32], mu: f32) -> f32 {
    let t = t_statistic(data, mu);
    let dof = (data.len() - 1) as f32;
    let x = dof / (t * t + dof);
    incomplete_beta(x, 0.5 * dof, 0.5) / beta(0.5 * dof, 0.5)
}
```

Now you can go ahead and add a simple test of the t-test to make sure it gives the right answer!

```rs
#[test]
fn test_t_test() {
    let data = [1.0, 2.0, 3.0, 4.0];
    let p = t_test(&data, 2.0);
    assert!((p - 0.4950).abs() < 1e-4)
}
```

While `statrs` doesn't have a t-test function, you can either precompute it and copy (as I did here) or use another statistics crate as a dev dependency.
