+++
title = "RAII for Crossterm"
date = 2026-02-22
+++

Welcome to part 2 in what has become a "how not to screw up your terminal" series for writing text user interfaces (TUIs)!

The first part was for Python and using [context managers for ncurses](@/notes/python_ncurses.md), but what about Rust? There's the great but low-level [`crossterm` crate](https://docs.rs/crossterm/latest/crossterm/), but it took me some time to get error handling set up to safely leave the terminal in the right state.

# The Problem

As a quick review of the problem, TUIs usually need to change terminal settings to work, like not echoing keys to the screen when you press them. When the TUI is done, it's supposed to set everything back to normal, but a crash can cause everything to get stuck in the wrong mode. Unlike a typical program's crash, the TUI's crash leaves you with a mess in your terminal.

If you're feeling dangerous and want to see an example of screwing up your terminal, try this:

```rs,name=main.rs
use crossterm;

fn return_error() -> Result<(), String> {
    Err("Oops, we got an error".to_string())
}

fn main() -> Result<(), String> {
    crossterm::terminal::enable_raw_mode().unwrap();
    return_error()?;
    crossterm::terminal::disable_raw_mode().unwrap();
    Ok(())
}
```

Things should be weird now. Modern terminals can clean up some of this stuff, but you might still see messed up newlines when programs print.

# RAII

The core idea is "Resource Acquisition Is Initialization" or RAII, which is a fancy way to say that we have destructors that are called when something goes out of scope.

Rust has the [`Drop` trait](https://doc.rust-lang.org/std/ops/trait.Drop.html) that provides a method to call when the object is done.

```rs
pub trait Drop {
    fn drop(&mut self);
}
```

Here's an example on a trivial struct that just prints when it's dropped.

```rs,name=main.rs
struct DropExample;

impl Drop for DropExample {
    fn drop(&mut self) {
        println!("DropExample dropped.");
    }
}

fn main() {
    let _example = DropExample;

    println!("Hello, world!");

    // DropExample dropped
}
```

Let's try it out with `cargo run`!

```
Hello, world!
DropExample dropped.
```

We can use this to clean up an object when things go out of scope, whether that's the end of the function or some error or early return.

## Naming

Note that we called the object `_example` rather than `example`. Rust would give us a warning that we didn't use the `example` variable here otherwise. The underscore tells Rust that we don't intend to use it again.

Make sure you don't use the single underscore `_`, or Rust will drop it *right away*. Check out the modified code here.

```rs,name=main.rs
struct DropExample;

impl Drop for DropExample {
    fn drop(&mut self) {
        println!("DropExample dropped.");
    }
}

fn main() {
    let _ = DropExample;

    // DropExample dropped

    println!("Hello, world!");
}
```

Running it now will reverse the order of the print statements! Now `DropExample` is dropped right away.

```
DropExample dropped.
Hello, world!
```

I first found this behavior a bit unintuitive; `_` gets dropped right away but `_example` lasts. You'd usually never care, since both of these examples are contrived and goofy. The one case you do care is for the terminal state management we want! Make sure that you're using the right unused variable names here.

## Multiple Drops

The last thing we'll need is using multiple drops at once. In general, the [drop ordering](https://doc.rust-lang.org/reference/destructors.html) can be pretty complex, but we'll make sure to keep things simple.

```rs,name=main.rs
struct DropExample(u8);

impl Drop for DropExample {
    fn drop(&mut self) {
        println!("_example{} dropped", self.0);
    }
}

fn main() {
    let _example1 = DropExample(1);
    let _example2 = DropExample(2);

    println!("Hello, world!");

    // _example2 dropped
    // _example1 dropped
}
```

Run it and see:

```
Hello, world!
_example2 dropped
_example1 dropped
```

# Cleaning Up After Errors

We can use this RAII/Drop trick to clean up after errors. Now, we automatically call the cleanup when we reach the end of the `main` function, or when we exit early!

```rs,name=main.rs
use crossterm;

struct RawMode;

impl RawMode {
    fn enter() -> std::io::Result<Self> {
        crossterm::terminal::enable_raw_mode()?;
        Ok(RawMode)
    }
}

impl Drop for RawMode {
    fn drop(&mut self) {
        let _ = crossterm::terminal::disable_raw_mode();
    }
}

fn main() {
    let _guard = RawMode::enter().unwrap();

    println!("This won't mess things up afterwards.");
}
```

Of course we expected that to work well, but now let's force an error.

```rs,name=main.rs
use crossterm;

struct RawMode;

impl RawMode {
    fn enter() -> std::io::Result<Self> {
        crossterm::terminal::enable_raw_mode()?;
        Ok(RawMode)
    }
}

impl Drop for RawMode {
    fn drop(&mut self) {
        let _ = crossterm::terminal::disable_raw_mode();
    }
}

fn return_error() -> Result<(), String> {
    Err("Oops, we got an error".to_string())
}

fn main() -> Result<(), String> {
    let _guard = RawMode::enter().unwrap();

    return_error()?;

    println!("No error.");

    Ok(())
}
```

When you run this, you should get an error message but no long-term damage to your terminal!

```
Error: "Oops, we got an error"
```

You should be all safe to crash in raw mode now.

# Alternate Screen Safety

The other thing we need to handle is entering and leaving the alternate screen. The TUI uses this screen for display, then switches back so your shell is visible. If we exit early, we won't be able to return to what we saw before. The following program should cause problems:

```rs,name=main.rs
use crossterm;
use crossterm::ExecutableCommand;
use crossterm::terminal::{EnterAlternateScreen, LeaveAlternateScreen};

fn return_error() -> Result<(), String> {
    Err("Oops, we got an error".to_string())
}

fn main() -> Result<(), String> {
    let mut stdout = std::io::stdout();
    stdout.execute(EnterAlternateScreen).unwrap();
    return_error()?;
    stdout.execute(LeaveAlternateScreen).unwrap();
    Ok(())
}
```

You can get back to the usual shell by running any other program that uses the alternate screen and, unlike the example, cleans up afterwards.

We need to modify the trick because the line we want to use involves `stdout`, which must be mutable. If we gave `stdout` to an unused struct, then we wouldn't be able to use it to make the TUI! We also can't implement `Drop` for `Stdout` ourselves.

Instead, we just have to wrap it in another struct and implement drop there.

```rs,name=main.rs
use crossterm::ExecutableCommand;
use crossterm::terminal::{EnterAlternateScreen, LeaveAlternateScreen};

struct AlternateScreen(std::io::Stdout);

impl AlternateScreen {
    fn enter() -> std::io::Result<Self> {
        let mut stdout = std::io::stdout();
        stdout.execute(EnterAlternateScreen)?;
        Ok(AlternateScreen(stdout))
    }
}

impl Drop for AlternateScreen {
    fn drop(&mut self) {
        let _ = self.0.execute(LeaveAlternateScreen);
    }
}

fn return_error() -> Result<(), String> {
    Err("Oops, we got an error".to_string())
}

fn main() -> Result<(), String> {
    let screen = AlternateScreen::enter().unwrap();
    println!("I can access {:?}", screen.0);
    return_error()?;
    Ok(())
}
```

I used `screen.0` for a useless print, but a real TUI would do all its work with `screen.0` in place of `stdout`. For that full TUI, an `AlternateScreen` would be paired with the `RawMode` guard to handle both clean ups.
