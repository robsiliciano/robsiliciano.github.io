+++
title = "Testing in Rust, Coming from Python"
date = 2026-03-18
description = "Writing tests in Rust is moderately different from writing them in Python. That's probably not shocking, given how different the two languages are. Here's my notes on getting started with Rust testing, coming from a Python testing background."
+++

I recently needed to write a Rust program to help me with another project (writing a language server, but that's another story). This utility needed to be robust, as any crashes would throw me out of the loop on my main focus. So I decided to add a bunch of tests.

My testing background is in Python, [`unittest`](https://docs.python.org/3/library/unittest.html) through Django and [`pytest`](https://docs.pytest.org/en/stable/). Python, however, is a very different beast from Rust, and their testing has some notable differences. Here's my notes on how to test in Rust, coming from Python.

I found the biggest thing to be conceptual differences, rather than the more translatable differences like Python's assert methods vs Rust's assert macros (`assert!`/`assert_eq!`/etc.), or Python's tests for raised errors vs Rust's [`#[should_panic]`](https://doc.rust-lang.org/rust-by-example/testing/unit_testing.html#testing-panics) or direct `Result` testing. If you want more details, check out [the Rust Book](https://doc.rust-lang.org/book/ch11-01-writing-tests.html) or [Rust By Example](https://doc.rust-lang.org/rust-by-example/testing.html).

# Test Right There

In Python, you'd usually have tests in separate files from the code you want to test, generally with a filename like `test*.py` or `*test.py`. You _could_ have them in the same file with `unittest`, but it's [not recommended](https://docs.python.org/3/library/unittest.html#organizing-test-code).

Rust goes the opposite direction and wants you to put the tests in the [same file as the code it's testing](https://doc.rust-lang.org/book/ch11-03-test-organization.html#unit-tests). You just mark all the tests with a `#[test]` attribute.

To stop the test code from being put in your final product, put all the tests in a `tests` submodule and use the `#[cfg(test)]` annotation to tell Cargo to _only_ build this as part of `cargo test`, and never as part of `cargo build`.

I think the reasons that Python and Rust go in different directions come down to differences in the languages, one compiled and one interpreted.

## Keeping Test Code Apart

Rust can easily exclude code as a compiled language, so it doesn't have Python's problem putting test code in the same files as the regular code. If you put test code in there, the compiler will ignore it when making binaries. However, Python doesn't have that as an interpreted language, so if you have test code in your Python module, someone could access it. I don't know if that's necessarily the worst thing, but at the minimum, those extra test cases are processed when someone imports your module and increase the size of programs for no reason.

## Testing Private Code

Rust has a public-private distinction, unlike Python where everything is public. Your methods starting with an underscore? Still accessible. So a Python test can access anything in a module to test. There's no cost to putting tests elsewhere! On the other hand, Rust actually has functions or structs that _can't_ be accessed by tests in another file. If you wanted to test private code (and [that's debatable](https://doc.rust-lang.org/book/ch11-03-test-organization.html#private-function-tests)), you can only access it from one place.

# No Mocking

There's no [mock](https://docs.python.org/3/library/unittest.mock.html) in Rust tests, where you can create a fake version of an object, function, or class in a test. I thought I'd miss them, but it turns out that testing in Rust is just fine if you're prepared.

Mocking works great with Python's duck typing, since there's no rule that anything has to be exactly a certain type. Objects just need to have the methods that your code requires, so you can replace a real object with a version with pre-set return values. If you want to test a function that operates on a file pointer, you can make an object that has the same methods as the file pointer object without actually having to test with real files!

Rust, being compiled, won't let you get away with duck typing! If your function requires a [`File`](https://doc.rust-lang.org/std/fs/struct.File.html), then you _have_ to pass it a `File`.

```rs
pub fn your_func(file: &mut File) {
    file.write_all(b"Hello, world").unwrap();
}
```

You can't have some mock that implements `write_all` and tracks the value. If you try to create such a mock for testing and pass it to `your_func`, Rust won't compile your tests to even run them.

## The Right Trait

The trick here is to **not require a `File` in the first place**. Okay, you ask, but the whole point of this function is to use a file! How can you _not_ have a `File` argument?

It comes down to Rust's traits. You don't need a `File`, you just need something that implements the [`Write`](https://doc.rust-lang.org/std/io/trait.Write.html) trait.

```rs
pub fn your_func<W: Write>(file: &mut W) {
    file.write_all(b"Hello, world").unwrap();
}
```

Your main code will still work, since `your_func(file)` will really be `your_func::<File>(file)`, which is the exact same code as you originally had! If you don't like having the generic exposed to everyone, you could always split it in two: the code in a private generic, and the interface in a trivial public function.

```rs
fn your_func_w<W: Write>(file: &mut W) {
    write!(file, "Hello, world").unwrap();
}

pub fn your_func(file: &mut File) {
    your_func_w(file);
}
```

You'd now test `your_func_w` not `your_func`, and feel confident that `your_func` behaves appropriately.

## Duck Typing, Duck Typing, Grey Duck Typing

Okay, so Rust doesn't do duck typing, but it kinda can if you want it to. Python duck typing works by checking what functions your object has, but Rust won't do that.

```rs
struct FileMock;

impl FileMock {
    fn write_all(&mut self, buf: &[u8]) -> Result<()> {
        ...
    }
}
```

The origin of the phrase "duck typing" comes from the quote "if it walks like a duck and it quacks like a duck, then it must be a duck." In Rust, it's more like "if it has the duck walking trait and the duck quacking trait, it must be a duck." All that means is you need to implement the relevant trait, and Rust will let you do it.

```rs
struct FileMock;

impl Write for FileMock {
    fn write(&mut self, buf: &[u8]) -> Result<usize> {
        ...
    }

    fn flush(&mut self) -> Result<()> {
        ...
    }
}
```

Now, `your_func` can take a `FileMock` no problem!

# Testing Binaries

One last note, and this applies to both Python and Rust, is how to test binaries. The standard unit tests are set up to test an interface, so they work better for libraries or small parts whose interface is in the language itself. However, a binary (or script for Python) has its interface outside the language. You'd usually be calling it from the shell!

The [Rust Book](https://doc.rust-lang.org/book/ch11-03-test-organization.html#integration-tests-for-binary-crates) recommends that you stay within the Rust testing system as much as possible, by moving your binary functionality into internal modules. You can test those easily, and have little code in `main.rs` outside the scope of your test submodules.
