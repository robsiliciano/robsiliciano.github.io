+++
title = "Match Statements: Rust and Python"
date = 2026-05-03
description = "Both Rust and Python have match statements, but I only really see or use them in Rust code. These notes compare Python's match to Rust's match: how they're similar and a few key differences."
+++

I use `match` statements in Rust all the time, but rarely ever touch them in Python for some reason. Python has had a [`match` statement](https://docs.python.org/3/reference/compound_stmts.html#match) since Python 3.10; that's almost half a decade ago! Overall, it seems like the two languages have broadly similar `match` capabilities, with a few key differences that I've left for the end.

# Basic Syntax

Here are two basic match statements in both languages. If the number is one, the code prints `That's one`. If the number is two, the code prints `That's two`. Otherwise, it prints `That's something`.

```rs
let number = 1;

match number {
    1 => println!("That's one"),
    2 => println!("That's two"),
    _ => println!("That's something"),
}
```

Python's `match` looks broadly similar, except with explicit `case`s instead of the `=>` in Rust. While Python has `case` like a C `switch` statement, it doesn't allow fall through, so there's no need for a `break`.

```py
number = 1

match number:
    case 1:
        print("That's one")
    case 2:
        print("That's two")
    case _:
        print("That's something")
```

## Wildcards

Both Rust and Python use `_` for the wildcard in pattern matching. The match value is discarded, and the code block runs.

You can use a simple `_` pattern at the end to catch all other cases. This catch-all case is a nice-to-have in Python, where it serves a similar purpose to an `else` after an `if`.

However, the `_` pattern shows up a ton in Rust where you need to have your match statements be exhaustive.

# Binding

Both Rust and Python let you bind patterns with variables, then use the variables in the match arm.

```rs
let point = (1, 2);

match point {
    (x, 2) => println!("That's {x} and two"),
    _ => println!("That's something")
}
```

```py
point = (1, 2)

match point:
    case (x, 2):
        print(f"That's {x} and two")
    case _:
        print("That's something")
```

# Combining Patterns

Both Rust and Python let you use `|` to combine multiple patterns into one.

```rs
let number = 1;

match number {
    1 | 2 | 3 => println!("That's one, two, or three"),
    _ => println!("That's something"),
}
```

```py
number = 1

match number:
    case 1 | 2 | 3:
        print("That's one, two, or three")
    case _:
        print("That's something")
```

## Binding in Combinations

If you bind in one pattern, then you have to bind in them all. The following example is okay, since both tuples bind `x`.

```rs
let point = (1, 2);

match point {
    (x, 1) | (x, 2) => println!("That's {x} and one or two"),
    _ => println!("That's something")
}
```

```py
point = (1, 2)

match point:
    case (x, 1) | (x, 2):
        print(f"That's {x} and one or two")
    case _:
        print("That's something")
```

What you _can't_ do is bind `x` in some but not all patterns. If the value matched one of the patterns where `x` was never bound, then any reference to `x` in the code block could run into an undefined variable if the language allowed it. For instance, the following versions would fail:

```rs
let point = (1, 2);

match point {
    (x, 1) | (1, 2) => println!("What's {x}?"),
    _ => println!("That's something")
}
```

```py
point = (1, 2)

match point:
    case (x, 2) | (1, 3):
        print(f"What's {x}?")
    case _:
        print("That's something")
```

# Match Guards

Both Rust and Python allow **match guards**, which are additional conditions for a match following an `if`.

```rs
let number = 1;

match number {
    x if (x > 0) && (x <= 3) => println!("That's one, two, or three"),
    _ => println!("That's something"),
}
```

```py
number = 1

match number:
    case x if 0 < x <= 3:
        print("That's one, two, or three")
    case _:
        print("That's something")
```

Be careful: match guards can have side effects, but might not always be run depending on the order of cases!

## Range Patterns

Rust has a built-in way to specify [range conditions](https://doc.rust-lang.org/book/ch19-03-pattern-syntax.html#matching-ranges-of-values-with-) without using a match guard.

```rs
let number = 1;

match number {
    1..=3 => println!("That's one, two, or three"),
    _ => println!("That's something"),
}
```

You can bind to these range patterns with [the `@` operator](https://doc.rust-lang.org/book/ch19-03-pattern-syntax.html#using--bindings).

```rs
let number = 1;

match number {
    x @ 1..=3 => println!("That's one, two, or three ({x})"),
    _ => println!("That's something"),
}
```

Python doesn't have an equivalent, so you would have to use match guards if you wanted the value in a variable.

# Matching Classes

Matching classes is where Rust and Python start looking a bit different. Python's flexible classes and Rust's strict structs will require different matching behavior.

```rs
struct Point {
    x: u32,
    y: u32,
}

let p = Point { x: 1, y: 2 };

match p {
    Point { x, y } => println!("{x}, {y}"),
}
```

```py
class Point:
    def __init__(self, x, y):
        self.x = x
        self.y = y

p = Point(1, 2)

match p:
    case Point(x=x, y=y):
        print(f"{x}, {y}")
```

Note that Python is matching the awkward `Point(x=x, y=y)`, not the more natural `Point(x, y)`. If you tried that, you'd run into problems.

```py
match p:
    case Point(x, y):
        print(f"{x}, {y}")
```

The above `match` is going to give an error.

```
TypeError: Point() accepts 0 positional sub-patterns (2 given)
```

Python class matching only likes keyword arguments for patterns, not positional ones. Even if the argument is positional in `__init__`, as they were in the `Point` example, Python will refuse to match.

In order to use positional patterns, you need to specify [`__match_args__`](https://docs.python.org/3/reference/datamodel.html#customizing-positional-arguments-in-class-pattern-matching) (automatically generated on dataclasses). Python will use these as the keyword arguments under the hood.

```py
class Point:
    __match_args__ = ("x", "y")

    def __init__(self, x, y):
        self.x = x
        self.y = y

p = Point(1, 2)

match p:
    case Point(x, y):
        print(f"{x}, {y}")
```

However, you should be careful that the `__match_args__` match your intended order in `__init__`. If you switched the order, then you'd get it matched backwards!

```py
class Point:
    __match_args__ = ("y", "x")

    def __init__(self, x, y):
        self.x = x
        self.y = y

p = Point(1, 2)

match p:
    case Point(x, y):
        print(f"{x}, {y}")
```

This code prints `2, 1`, not the `1, 2` that you probably wanted with the match.

The one consequence of Python's keyword-only approach is that you don't need to explicitly put anything in for any fields you don't want to use. If you want to ignore other arguments in Rust, you would [use `..` at the end](https://doc.rust-lang.org/book/ch19-03-pattern-syntax.html#remaining-parts-of-a-value-with-).

```rs
struct Point {
    x: u32,
    y: u32,
}

let p = Point { x: 1, y: 2 };

match p {
    Point { x, .. } => println!("{x}"),
}
```

Python lets you not use all of the `__match_args__`, just taking the first few until it's out of symbols to bind. The following code works just fine, even though `y` was never bound.

```py
class Point:
    __match_args__ = ("x", "y")

    def __init__(self, x, y):
        self.x = x
        self.y = y

p = Point(1, 2)

match p:
    case Point(x):
        print(f"{x}")
```

## Matching Enums

Matching enums is great in Rust, where enums are sum types with their own values to conditionally match.

```rs
enum Point {
    Single { x: u32 },
    Double { x: u32, y: u32 },
}

let p = Point::Double { x: 1, y: 2 };

match p {
    Point::Single { x } => println!("Single: {x}"),
    Point::Double { x, .. } => println!("Double: {x}"),
}
```

Python's `Enum`s don't have fields, so you can only match on the variant.

```py
from enum import Enum

class Point(Enum):
    Single = 1
    Double = 2

p = Point.Double

match p:
    case Point.Single:
        print("Single")
    case Point.Double:
        print("Double")
```

If you wanted to match on values in a sum type, you can do that with duck typing.

```py
class PointSingle:
    __match_args__ = ("x",)

    def __init__(self, x):
        self.x = x

class PointDouble:
    __match_args__ = ("x", "y")

    def __init__(self, x, y):
        self.x = x
        self.y = y

p = PointDouble(1, 2)

match p:
    case PointSingle(x):
        print(f"Single: {x}")
    case PointDouble(x):
        print(f"Double: {x}")
```

In that case, the `Point` type would be `PointSingle | PointDouble`, but that's only for typing purposes.

These enum matches get used a lot in Rust to deal with a `Result`, where you can handle the return value in `Ok` and an error in `Err`. In contrast, Python has separate error handling with `try...except` blocks.

```rs
match function_returning_result() {
    Ok(value) => {
        // Something with the return value
    },
    Err(e) => {
        // Something with the error
    },
}
```

Note: you don't need to do `Result::` here because `Ok` and `Err` are in the prelude.

## Capturing the Value

The `@` operator can also be useful for matching `enum`s in Rust, if you want the whole value not just the items.

```rs
enum Point {
    Single { x: u32 },
    Double { x: u32, y: u32 },
}

let p = Point::Double { x: 1, y: 2 };

match p {
    s @ Point::Single { .. } => {
        // something with s as a Point::Single
    },
    d @ Point::Double { .. } => {
        // something with d as a Point::Double
    },
}
```

Again, Python doesn't have an equivalent. However, you don't need to worry about compilation and types. If you needed to use some method only available on `Double`, then Python's duck typing lets you use it right with `p`.

# Key Differences

## Required Coverage

The big difference between Rust's `match` and Python's `match` is that Rust requires match arms to cover all possible options, but Python doesn't.

The following is okay in Python's `match` syntax, even though it doesn't cover the number three.

```py
number = 1

match number:
    case 1:
        print("That's one")
    case 2:
        print("That's two")
```

However, Rust's `match` statement won't let you skip three or the rest of the numbers.

```rs
let number = 1;

match number {
    1 => println!("That's one"),
    2 => println!("That's two"),
}
```

While Python's flexibility seems nice, the downside is that you're not guaranteed to run an arm. If you're setting a variable or returning a value, you might hit cases where you don't get anything. Rust will tell you during compilation, but Python won't until the unset variable becomes a problem.

## Returning Values

Rust lets `match` happen inline, like when setting a variable. You can have conditional assignment like the code below. Here, the requirement that all cases have an arm is key: if a value didn't match any case, then there wouldn't be a value to return!

```rs
let number = 1;

let text = match number {
    1 => "That's one",
    2 => "That's two",
    _ => "That's something",
};
```

You need a trailing semicolon here, since this is a `let` statement overall.

## Dictionaries

Python has a shorthand for matching keys in dictionaries.

```py
dictionary = {"key1": 1, "key2": 2}

match dictionary:
    case {"key1": 1}:
        print("That's one")
```

With Rust, you have to get the key and do something with it.

```rs
use std::collections::HashMap;

let dictionary = HashMap::from([("key1", 1), ("key2", 2)]);

match dictionary.get("key1") {
    Some(1) => println!("That's one"),
    _ => {},
}
```

The Python syntax also lets you bind other keys.

```py
dictionary = {"key1": 1, "key2": 2}

match dictionary:
    case {"key1": 1, "key2": key2}:
        print(f"key2 is {key2}")
```

You can do this in Rust, but you'd have to do more queries of the hash map.

# Further Links

## Python

 - [PEP 634](https://peps.python.org/pep-0634/) defined Python's `match` syntax.
 - [PEP 636](https://peps.python.org/pep-0636/) is an accompanying tutorial.
 - The [official Python `match` documentation](https://docs.python.org/3/reference/compound_stmts.html#the-match-statement).

## Rust

 - The [Rust documentation on `match` statements](https://doc.rust-lang.org/reference/expressions/match-expr.html).
 - [Rust by Example](https://doc.rust-lang.org/rust-by-example/flow_control/match.html) on `match` statements.
 - [The Rust Book](https://doc.rust-lang.org/book/ch06-02-match.html) on `match` statements.
