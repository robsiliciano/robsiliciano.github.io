+++
title = "Deriving Proc Macros in Rust"
date = "2026-04-03"
description = "Rust's proc macros are a powerful, but complicated, way to implement traits. When done well, you can just derive traits on any eligible struct you want! These are my notes getting started with proc macros for deriving traits and some of the traps to avoid."
+++

If I had to give a tagline for Rust procedural macros, it'd be "generic trait implementations for thrill seekers!" (Call me, Rust maintainers.) Proc macros, as they're called, power Rust's ability to `#[derive()]` implementations on your structs. While generics can fill in the implementation, they're limited by what you can fit into the generics system. Proc macros are much more flexible, but that also makes them much more complex!

For a practical intro, let's implement a silly trait that couldn't be implemented with generics.

# The Taxicab Distance

The [taxicab distance](https://en.wikipedia.org/wiki/Taxicab_geometry) is a simple metric on multidimensional spaces, where the distance is just the sum of distances in each dimension. The name comes from a taxi driving on a city grid, where it has to follow the roads and not travel as the crow flies.

Let's start with a trait that has a taxicab method.

```rs
pub trait Taxicab {
    fn taxicab(&self, other: &Self) -> f64;
}
```

Ideally, we could have this for all sorts of structs that only contain numbers.

```rs
#[derive(Debug)]
struct Point2 {
    x: f64,
    y: f64,
}

impl Taxicab for Point2 {
    fn taxicab(&self, other: &Self) -> f64 {
        (self.x - other.x).abs() + (self.y - other.y).abs()
    }
}

#[derive(Debug)]
struct Point3 {
    x: f64,
    y: f64,
    z: f64,
}

impl Taxicab for Point3 {
    fn taxicab(&self, other: &Self) -> f64 {
        (self.x - other.x).abs() + (self.y - other.y).abs() + (self.z - other.z).abs()
    }
}
```

Okay, but this is a pain if we have to implement it for every single struct! It's all the same basic logic, so can't we take a shortcut?

# Derive with Proc Macros

You've surely used `#[derive()]` for a struct; the examples above both derived `Debug`! Some other common targets of `derive` are `Clone`, `Copy`, and `PartialOrd`.

Fortunately, we're not restricted to only deriving the standard traits, so we can use `#[derive(Taxicab)]` to automatically generate implementations of our own traits.

## Crate Setup

Unfortunately, you can't have the proc macro that derives `Taxicab` in the same crate where it's used, although the Rust team [hopes to drop](https://doc.rust-lang.org/book/ch20-05-macros.html#procedural-macros-for-generating-code-from-attributes) this requirement in the future. Instead, you need to carefully set up your workspace to include both the regular crate defining `Taxicab` and a special crate just containing the proc macros. Essentially, the proc macros are run _during_ compilation, so they need to be built before any crate that would use them.

To use multiple crates, you'll want to use a [Cargo workspace](https://doc.rust-lang.org/cargo/reference/workspaces.html). For some example workspace setups in projects that need proc macros, check out [serde](https://github.com/serde-rs/serde/blob/master/Cargo.toml) or [polars](https://github.com/pola-rs/polars/blob/py-1.36.1/Cargo.toml).

First, create one binary crate, `taxicab`, that will contain the trait definition and the `main` function. Then, create a second crate, the library `taxicab-derive`, that has the proc macro we need to use `#[derive(Taxicab)]`.
```sh
cargo new taxicab
cd taxicab
cargo new --lib taxicab-derive
```

In the `taxicab` `Cargo.toml`, set up the workspace.
```toml
[workspace]
members = [".", "taxicab-derive"]

[dependencies]
taxicab-derive = { path = "taxicab-derive" }
```

We also have to tell Rust that `taxicab-derive` is going to give us proc macros, with a line in `taxicab-derive`'s `Cargo.toml`. Make sure to put it in the right configuration, since we've got a lot of `Cargo.toml`s floating around now.

```toml
[lib]
proc-macro = true
```

## A First Proc Macro

The barebones proc macro looks pretty short: you just take in a [`TokenStream`](https://doc.rust-lang.org/proc_macro/struct.TokenStream.html), do something to it, and return another `TokenStream`. 

```rs
use proc_macro::TokenStream;

#[proc_macro_derive(Taxicab)]
pub fn taxicab_derive(input: TokenStream) -> TokenStream {
    todo!()
}
```

Filling it in is the hard part!

## Syn

Despite proc macros being a core part of Rust, we pretty much always want two external crates to help fill out the derive function. The first is [syn](https://docs.rs/syn/latest/syn/), which parses the token stream input into a syntax tree that we can traverse and process.

```rs
let ast = syn::parse_macro_input!(input as syn::DeriveInput);
```

## Quote

The second external crate is [quote](https://docs.rs/quote/latest/quote/), which lets us generate token streams by writing Rust-like code with its `quote!` macro. It has a minimal pattern [interpolation](https://docs.rs/quote/latest/quote/macro.quote.html#interpolation) that uses the `#` symbol to identify variables from outside of the scope.

Here's an example of how you'd use it, with [`format_ident!`](https://docs.rs/quote/latest/quote/macro.format_ident.html) to get `y` as a variable not a string.

```rs
fn example() -> TokenStream {
    let x = format_ident!("y");
    let rust = quote!( let #x = "x"; );
    rust.into()
}
```

The example gives you a token stream corresponding to the line below.
```rs
let y = "x";
```

## A First Take

Putting it all together, here's a quick implementation that derives the taxicab method!

```rs
use proc_macro::TokenStream;
use quote::quote;

#[proc_macro_derive(Taxicab)]
pub fn taxicab_derive(input: TokenStream) -> TokenStream {
    let ast = syn::parse_macro_input!(input as syn::DeriveInput);

    // The struct's name: impl Taxicab for ???
    let name = &ast.ident;

    // We need to go through and get all the struct's fields
    let mut sum = vec![];
    if let syn::Data::Struct(data) = &ast.data {
        if let syn::Fields::Named(fields) = &data.fields {
            for field in &fields.named {
                let field_name = field.ident.as_ref().unwrap();
                // A field named x becomes self.x.taxicab(&other.x)
                sum.push(quote! { self.#field_name.taxicab(&other.#field_name) });
            }
        }
    }

    // Putting it all together for the trait implementation
    let implementation = quote! {
        impl Taxicab for #name {
            fn taxicab(&self, other: &Self) -> f64 {
                0.0 #(+ #sum)*
            }
        }
    };
    implementation.into()
}
```

You can derive this on the `Point2` struct above.

```rs
#[derive(Debug, Taxicab)]
struct Point2 {
    x: f64,
    y: f64,
}
```

When you compile, the proc macro generates the following implementation.

```rs
impl Taxicab for Point2 {
    fn taxicab(&self, other: &Self) -> f64 {
        0.0 + self.x.taxicab(&other.x) + self.y.taxicab(&other.y)
    }
}
```

(We haven't implemented `Taxicab` for `f64` yet, but we'll get to it later!)

So most of this is a fairly straightforward traversal of a syntax tree, but I want to talk through how the quoting works. When I go through the struct fields, I collect the field method calls. In the `Point2` case, the `sum` ends up with two quotes like the vec below.
```rs
vec![quote!( self.x.taxicab(&other.x) ), quote!( self.y.taxicab(&other.y) )]
```

We now have to join this into one line, which is what the `0.0 #(+ #sum)*` does. The `#()*` lets you expand a list.
* You can add separators before the asterisk: `#(),*` would use a comma between items.
* You can add other characters in the repeated part, like the "+ " in this case.

We don't need to use "+" as a separator here, since we've got the leading `0.0`. That zero will _usually_ compile away, except in a single degenerate case without any fields.

```rs
#[derive(Taxicab)]
struct Point0;
```

Calling the taxicab method will always return zero.

## A Generic Implementation

We need to implement the taxicab distance for _something_ to be able to use the derived implementation, since it uses the `taxicab` method for the struct's fields. We could just write it for `f64`, but why not do something more general? Ideally we'd cover `f32`, `i32`, and the like. This problem is a great use case for generics!

All we need to do for the primitive fields is subtract two values, take the absolute value, and somehow get it to a `f64`. We can do this with standard library traits! Specifically, [`std::ops::Sub`](https://doc.rust-lang.org/std/ops/trait.Sub.html) covers subtraction and we can use [`Into<f64>`](https://doc.rust-lang.org/std/convert/trait.Into.html) to convert to our final type. We don't need a trait for the absolute value if we convert the difference to a float first.

```rs
use std::ops::Sub;

impl<T> Taxicab for T
where
    for<'a> &'a T: Sub<&'a T>, 
    for<'a> <&'a T as Sub<&'a T>>::Output: Into<f64>,
{
    fn taxicab(&self, other: &Self) -> f64 {
        Into::<f64>::into(self - other).abs()
    }
}
```

The one tricky part is that `Sub` can use all sorts of types! You implement it for the left-hand side type, and the right-hand side is the generic: `Sub<Rhs>`. The [output](https://doc.rust-lang.org/std/ops/trait.Sub.html#associatedtype.Output) is part of the trait. We'll need to require `Into<f64>` for the output type, not the base generic itself.

Also, we have to use a lot of lifetimes here because we're operating on references. We wouldn't want to consume the values when evaluating a taxicab distance!

## Trying It Out

With the trait implemented for a lot of numeric primitives, we can start using the proc macro on our structs.

```rs
#[derive(Taxicab, Debug)]
struct Point {
    x: f64,
    y: f64,
}

#[derive(Taxicab, Debug)]
struct PointMixed {
    x: f64,
    y: i32,
}

#[derive(Taxicab, Debug)]
struct PointParent {
    x: f32,
    y: PointMixed,
}
```

In the last case, `PointParent`, our derived `taxicab` method is itself using a derived `taxicab` method on `PointMixed`.

# Hygiene

There's an important concept in macros called ["hygiene"](https://dl.acm.org/doi/10.1145/319838.319859) that basically refers to how well symbols in the caller's scope bleed into the macro-generated code. Do the identifiers from where the macro was called affect the execution of a macro? For example, this C macro is unhygienic and pulls in `y` from the containing scope, giving different results for the same input. 

```c
#define F(x) x + y

int function1() {
    int y = 1;
    return F(1);
}

int function2() {
    int y = 2;
    return F(1);
}
```

In contrast, functions are hygienic, since you don't need to care about what's in scope when the function is called.

The key thing to know is that [proc macros are unhygienic](https://doc.rust-lang.org/reference/procedural-macros.html#procedural-macro-hygiene)! We're just outputting a token stream that's interpreted in the local context.

We should particularly worry about traits. In the Rust code block above, we generated an `impl Taxicab for ...`, but never specified what `Taxicab` meant!

What if some other trait were in scope as `Taxicab`? For instance, you could (but never would) pull in another trait with that name.
```rs
use std::fmt::Display as Taxicab;
```

Then, you'd get a compilation [error](https://doc.rust-lang.org/error_codes/E0407.html) that your derive proc macro implements the wrong method!
```
error[E0407]: method `taxicab` is not a member of trait `Taxicab`
 --> src/main.rs:5:10
  |
5 | #[derive(Taxicab, Debug)]
  |          ^^^^^^^ not a member of trait `Taxicab`
  |
  = note: this error originates in the derive macro `Taxicab` (in Nightly builds, run with -Z macro-backtrace for more info)
```

Another, probably more realistic scenario, is that you don't have the `Taxicab` trait in scope when you derive it. You wouldn't run into this problem for a small program where you define structs and use the taxicab distance in the same file, since you have to have `Taxicab` in scope whenever you use the `taxicab` method. However, you might have a larger project where all the structs that derive the taxicab distance are in a file that doesn't have `Taxicab` in scope. In that setup, you'll get an [error](https://doc.rust-lang.org/error_codes/E0404.html) that you're trying to implement something that isn't a trait.

The way to get around the hygiene problem is to use the trait's full, legal name: `crate::Taxicab` (for a binary).

```rs
use proc_macro::TokenStream;
use quote::quote;

#[proc_macro_derive(Taxicab)]
pub fn taxicab_derive(input: TokenStream) -> TokenStream {
    let ast = syn::parse_macro_input!(input as syn::DeriveInput);

    let name = &ast.ident;

    let mut sum = vec![];
    if let syn::Data::Struct(data) = &ast.data {
        if let syn::Fields::Named(fields) = &data.fields {
            for field in &fields.named {
                let field_name = field.ident.as_ref().unwrap();
                sum.push(quote! { self.#field_name.taxicab(&other.#field_name) });
            }
        }
    }

    // Using :: here
    let implementation = quote! {
        impl crate::Taxicab for #name {
            fn taxicab(&self, other: &Self) -> f64 {
                0.0 #(+ #sum)*
            }
        }
    };
    implementation.into()
}
```

## Hygiene in Libraries

If you put a trait in a library, then `crate` is going to be whoever is calling your library, not the library itself! That's bad hygiene.

Instead, you can quantify the library as `::taxicab::Taxicab`.

To use the proc macro in your own library, you need to tell Rust that the library name refers to your `crate`.
```rs
extern crate self as taxicab;
```

## Recursive Definitions

In the `Taxicab` case, hygiene is a bit more complicated since we actually need `Taxicab` in scope for the derived implementation! This requirement is _not_ a general need for proc macros, but special to the recursive nature where each struct's `taxicab` method calls the `taxicab` method of its fields.

Again, we should worry about `Taxicab` being in scope when we call the method. We can be safe by putting the full quantifier for the method.

```rs
sum.push(quote! { ::taxicab::Taxicab::taxicab(&self.#field_name, &other.#field_name) });
```

I'll switch to the `::taxicab::Taxicab` version from here out.

# Other Things with Derive

I've only shown derive for normal structs with named fields, but you can throw a `#[derive(Taxicab)]` on a bunch of other things! If you don't take care, like in the code above, you'll end up with weird implementations for them.

## Other Types of Structs

First, your struct might not have any named fields.
```rs
#[derive(Taxicab, Debug)]
struct PointUnnamed(f64, f64);
```

While this type should behave the same as `Point2` above, the taxicab method will _always_ return zero, even when the two values are different! We're silently skipping past an `if let` statement and not handling the other cases.

To handle these structs, let's replace the `if let` with a `match`:

```rs
match &data.fields {
    syn::Fields::Named(fields) => { // Same as before
        for field in &fields.named {
            let field_name = field.ident.as_ref().unwrap();
            sum.push(quote! { ::taxicab::Taxicab::taxicab(&self.#field_name, &other.#field_name) });
        }
    }
    syn::Fields::Unnamed(fields) => { // New to handle tuple structs
        for index in 0..fields.unnamed.len() {
            let index = syn::Index::from(index);
            sum.push(quote! { ::taxicab::Taxicab::taxicab(&self.#index, &other.#index) });
        }
    }
    syn::Fields::Unit => {} // Nothing to do!
}
```

Now, you should get proper taxicab distances on your tuple structs.

You might note that there's also this case for `syn::Fields::Unit`, which corresponds to the simple structs with no fields.

```rs
#[derive(Taxicab)]
struct Point0;
```

The original implementation worked fine here as well, since it defaults to zero.

## Things That Aren't Structs

The last thing to keep in mind is enums and unions. We probably don't want to handle them in the taxicab metric, so we need some way to generate an error. We could `panic!`, but that will only execute when the code is run, _not_ when it's compiled!

Instead, we want the [`compile_error!`](https://doc.rust-lang.org/std/macro.compile_error.html) macro, which is like `panic!` but it crashes when the code is compiled, not when it's run.

```rs
if let syn::Data::Struct(data) = &ast.data {
    // Skipped for brevity
} else {
    return quote! {compile_error!("Can only derive Taxicab on structs.");}.into();
}
```

The trick here is that you need the `compile_error!` **inside** a `quote!` or you won't be able to build the derive crate! Remember that we're actually compiling twice: once for the crate that gives us the proc macro, and once for the program itself. We need the `compile_error!` to appear in the program, which is the second time we compile, not in the proc macro crate.

You could alternatively `panic!` in the proc macro directly, but that crash is a bit awkward for users to parse.

# All Together Now

Putting all those changes together would give you something like the following proc macro.

```rs
use proc_macro::TokenStream;
use quote::quote;

#[proc_macro_derive(Taxicab)]
pub fn taxicab_derive(input: TokenStream) -> TokenStream {
    let ast = syn::parse_macro_input!(input as syn::DeriveInput);

    let name = &ast.ident;

    let mut sum = vec![];
    if let syn::Data::Struct(data) = &ast.data {
        match &data.fields {
            syn::Fields::Named(fields) => {
                for field in &fields.named {
                    let field_name = field.ident.as_ref().unwrap();
                    sum.push(quote! { ::taxicab::Taxicab::taxicab(&self.#field_name, &other.#field_name) });
                }
            }
            syn::Fields::Unnamed(fields) => {
                for index in 0..fields.unnamed.len() {
                    let index = syn::Index::from(index);
                    sum.push(quote! { ::taxicab::Taxicab::taxicab(&self.#index, &other.#index) });
                }
            }
            syn::Fields::Unit => {}
        }
    } else {
        return quote! {compile_error!("Can only derive Taxicab on structs.");}.into();
    }

    let implementation = quote! {
        impl ::taxicab::Taxicab for #name {
            fn taxicab(&self, other: &Self) -> f64 {
                0.0 #(+ #sum)*
            }
        }
    };
    implementation.into()
}
```

# Further Reading

 * The Rust Book on [workspaces](https://doc.rust-lang.org/book/ch14-03-cargo-workspaces.html), [proc macros](https://doc.rust-lang.org/book/ch20-05-macros.html#procedural-macros-for-generating-code-from-attributes), and the standard library's [derivable traits](https://doc.rust-lang.org/book/appendix-03-derivable-traits.html).
 * Lukas Wirth's [the Little Book of Rust Macros](https://lukaswirth.dev/tlborm/proc-macros.html).
 * Tsvetomir Dimitrov's [intro to proc macros](https://petanode.com/posts/rust-proc-macro/).
 * The [Rust reference](https://doc.rust-lang.org/reference/procedural-macros.html#procedural-macro-hygiene) on proc macros.
