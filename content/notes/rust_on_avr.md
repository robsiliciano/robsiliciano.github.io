+++
title = "Running Rust on AVR Chips"
date = 2025-12-19
+++

# Rust on AVR

If you want to use Rust to program AVR chips (including Arduinos) then the easiest path is to start from [this template](https://github.com/Rahix/avr-hal-template), which initializes your project with all the correct configurations. All you need is the hardware and core tools (Rust, `avr-gcc`, etc.) and you're good to go.

I was curious about how the process worked, so I took the manual route to run a simple Hello World program on an Arduino Uno board. Here are my notes on how to set it up and why those configurations are needed.

# Project Setup

If you start without the template, you'll just have a minimal `Cargo.toml`.

```sh
cargo init rust_on_arduino
```

If you were to run the project now, you'd get a nice "Hello, World!" message back. That's great, but I want the code running on a different device!

# Configuring Cargo

First, we need to [configure Cargo itself](https://doc.rust-lang.org/cargo/reference/config.html) to build and run our code on different hardware than the current platform. Note that all of Cargo's configuration will go in the `.cargo/config.toml` file, not the project's `Cargo.toml`.

## Cross Compiling

You'll have to tell Cargo to cross compile for your AVR microcontroller. Specifically, Cargo needs to target the [`avr-none`](https://doc.rust-lang.org/beta/rustc/platform-support/avr-none.html) platform, with the [specific chip](https://github.com/llvm/llvm-project/blob/main/clang/lib/Basic/Targets/AVR.cpp) type as provided as a [codegen flag](https://doc.rust-lang.org/beta/rustc/codegen-options/index.html#target-cpu). In the case of an Arduino Uno board, the chip is the ATMega328p, though other boards may have other chips.

```toml
[build]
target = "avr-none"
rustflags = ["-C", "target-cpu=atmega328p"]
```

## The Standard Library

Usually, you wouldn't need to compile the Rust standard library. Rust provides a [pre-compiled binary](https://rust-lang.github.io/rustup/cross-compilation.html) in the `rust-src` component for most host platforms that you'd ever use. However, AVR microcontrollers aren't one of those platforms!

Fortunately, we don't actually want a pre-compiled Rust standard library for such a small microcontroller. We really only want the standard library's [`core` crate](https://doc.rust-lang.org/core/index.html), and we want to [build it ourselves](https://doc.rust-lang.org/cargo/reference/unstable.html#build-std) when we build the rest of our project. This lets us keep our binaries small, which is important on a microcontroller with limited program memory. (We'll add optimization flags later.)

There's also a lot in the Rust standard library that we don't necessarily want. First, we're running on bare metal without an operating system, so a lot of platform-based things for the filesystem or threading don't make sense. What would a filesystem look like for an off-the-shelf Arduino? Second, we don't necessarily want the heap and dynamic memory access, as provided by the standard library's [`alloc` crate](https://doc.rust-lang.org/alloc/). We'd incur overhead to use it, but many microcontroller programs are just fine without it.

To build the standard library's `core`, we'll need the following. Note that it's an unstable feature for binaries, which requires more configuration later on.

```toml
[unstable]
build-std = ["core"]
```

## Ravedude

Once Cargo builds an AVR binary, we still need to move it from your computer to the microcontroller's program memory.

We'll do that with [`ravedude`](https://blog.rahix.de/ravedude/), a tool for running your code on AVR microcontrollers. It integrates [`avrdude`](https://github.com/avrdudes/avrdude), the tool that transfers the binary, into the Cargo workflow so you can use `cargo run` to handle the full process. You might be familiar with `avrdude` if you've used other languages for AVR microcontrollers.

To tell Cargo to use `ravedude`, you'll need the following:

```toml
[target.'cfg(target_arch = "avr")']
runner = "ravedude"
```

You'll also need a `Ravedude.toml` configuration file to tell Ravedude what hardware you have. If you want to open the serial port to print to the console – necessary for Hello, World! – also include the last two lines here.

```toml,name=Ravedude.toml
[general]
board = "uno" # Whatever hardware you're using

# If you want to let your AVR code print to your terminal
open-console = true
serial-baudrate = 57600
```

## The Full File

Overall, the `.cargo/config.toml` file should look like this.

```toml,name=.cargo/config.toml
[build]
target = "avr-none"
rustflags = ["-C", "target-cpu=atmega328p"]

[unstable]
build-std = ["core"]

[target.'cfg(target_arch = "avr")']
runner = "ravedude"
```

# Project Configuration

Now, let's get to project configuration in `Cargo.toml`.

## Dependencies

You'll need three dependencies for a basic Hello, World program.

```sh
cargo add --git https://github.com/rahix/avr-hal.git arduino-hal -F arduino-uno
cargo add panic-halt ufmt
```

### `arduino-hal`

In an AVR project, you'll need a crate that lets you access the hardware. Usually, that will be a hardware abstraction layers (HAL) from [Rahix's `avr-hal` repo](https://github.com/rahix/avr-hal.git), which provides `arduino-hal` for Arduino boards and other HAL crates for specific AVR chips. I'll demo this with an Arduino Uno, but you can use another board or a non-Arduino AVR setup. Just make sure to switch configurations away from the Uno.

If you don't want to use the HAL, you can use [`avr-device`](https://github.com/Rahix/avr-device) instead. Compared to the HAL crates, this crate provides the bare minimum to use AVR hardware. For instance, you would have to use [memory-mapped registers directly](https://docs.rs/avr-device/latest/avr_device/#example-digital-io-port-access) for IO with `avr-device` but the HAL crates provide abstractions for [setting individual pins](https://rahix.github.io/avr-hal/avr_hal_generic/port/struct.Pin.html). The HAL crates also contain implementations for low-level protocols like [I2C](https://rahix.github.io/avr-hal/avr_hal_generic/i2c/struct.I2c.html). I'd recommend the HAL crate unless you're short on space or want to learn the nitty-gritty implementation of something.

The HAL crates aren't available in the crates.io registry, so you'll need to [add them from git](https://doc.rust-lang.org/cargo/reference/specifying-dependencies.html#specifying-dependencies-from-git-repositories).

### `panic-halt`

One consequence of not having the full Rust standard library is that we need to provide a [panic handler](https://docs.rust-embedded.org/book/start/panicking.html) that will run whenever the code `panic!`s.

[`panic-halt`](https://github.com/korken89/panic-halt) is a simple option that just enters an infinite loop if the code panics: the [whole thing](https://github.com/korken89/panic-halt/blob/master/src/lib.rs) is just `loop {}`. You won't get any traceback or error message with this handler, so be prepared for more difficult debugging!

### `ufmt`

Stylized as "micro" rather than "u," [this crate](https://docs.rs/ufmt/latest/ufmt/) is used by the HAL to write to the UART serial bus. It's nice on microcontrollers as an alternative to `core::fmt` since it's optimized for build size and speed and doesn't panic in many places. With `panic-halt` as your handler, any panicking will stop any activity without providing a traceback, so I'd recommend avoiding code that could panic as much as possible.

You don't need `ufmt` in general if you're not doing string processing or writing to the serial bus, but you'll always see it in the lock file as it's currently a dependency of `arduino-hal`.

## Profiles

Finally, we'll need to add two configurations to the build profiles.

First, we need to [abort on a panic](https://doc.rust-lang.org/cargo/reference/profiles.html#panic), since we can't unwind a panic without the standard library.

Second, we'll usually want to set an [optimization level](https://doc.rust-lang.org/cargo/reference/profiles.html#opt-level) to shrink the binary for a microcontroller's limited memory. Both `s` and `z` aim to shrink the size. Without this option, the basic program in these notes is a massive 17858 bytes, rather than only 1184 with size optimization.

Together, a profile might look like:
```toml
[profile.dev]
panic = "abort" # Needed for no_std
opt-level = "s"
```

## The Full File

Here's what a `Cargo.toml` would look like overall.

```toml,name=Cargo.toml
[package]
name = "rust_on_arduino"
version = "0.1.0"
edition = "2024"

[dependencies]
panic-halt = "1.0.0"
ufmt = "0.2.0"
arduino-hal = { git = "https://github.com/rahix/avr-hal.git", features = ["arduino-uno"] }

[profile.dev]
panic = "abort" # Needed for no_std
opt-level = "s"

[profile.release]
panic = "abort" # Needed for no_std
opt-level = "s"
```


# The Rust Toolchain

The last configuration file we need is for the Rust toolchain. Unfortunately, your off-the-shelf stable rust toolchain won't work AVR chips because building the standard library is unstable. Instead, we'll need to use a nightly toolchain.

There's a lot of ways to get a project-specific toolchain, like command line arguments, environmental variables, or project overrides. We'll want to specify it with a `rust-toolchain.toml` file so it's saved in the project. Note that the toolchain file is one of the [later places](https://rust-lang.github.io/rustup/overrides.html) that Rust will check, so make sure you don't have an environmental variable or override active.

The full file would look like:

```toml,name=rust-toolchain.toml
[toolchain]
channel = "nightly-2025-06-01" # A recent version that works with AVR
components = ["rust-src"] # Needed to build the standard library core
```

## Nightly Toolchain

First, this file selects a nightly toolchain and pins the version. Nightly is required for unstable features, and the version should be pinned since the nightly toolchains can have bugs or other changes that clash with the AVR code. For instance, I couldn't produce a working Hello World with later versions, but the June first version was successful.

## Rust Source

The `rust-toolchain.toml` file also says that we want the [Rust standard library source code](https://rust-lang.github.io/rustup/concepts/components.html), which we'll need this to build the `core` of the standard library.

# Hello, World!

Finally, we can write and run some code!

Without the full standard library, we also can't use the usual Rust `main`. By default, Rust would add some setup code that requires the full `std`. Instead, we use [`arduino_hal::entry`](https://rahix.github.io/avr-hal/arduino_hal/attr.entry.html) to designate the main function.

```rust,name=src/main.rs
#![no_std]
#![no_main] // Consequence of no_std

use panic_halt as _;

#[arduino_hal::entry]
fn main() -> ! {
    let dp = arduino_hal::Peripherals::take().unwrap();
    let pins = arduino_hal::pins!(dp);
    let mut serial = arduino_hal::default_serial!(dp, pins, 57600);

    ufmt::uwriteln!(serial, "Hello, world!").unwrap();

    loop {} // There's nothing if we exit
}
```

You can run this as `cargo run` with your Arduino or AVR board plugged in. If everything worked correctly, you should see something like the below output, ending in "Hello, World!"

```
     Running `ravedude target/avr-none/debug/rust_on_arduino.elf`
       Board Arduino Uno
 Programming target/avr-none/debug/rust_on_arduino.elf => /dev/cu.usbmodem1101
Reading 1184 bytes for flash from input file rust_on_arduino.elf
Writing 1184 bytes to flash
Writing | ################################################## | 100% 0.23 s 
Reading | ################################################## | 100% 0.18 s 
1184 bytes of flash verified

Avrdude done.  Thank you.
  Programmed target/avr-none/debug/rust_on_arduino.elf
     Console /dev/cu.usbmodem1101 at 57600 baud
             CTRL+C to exit.

Hello, world!
```
