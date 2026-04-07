+++
title = "Getting Started with tree-sitter"
date = 2026-04-06
description = "Getting started with tree-sitter can be hard: it's a super powerful code parsing library, but takes a bit to get going. These are my notes to get a small program off the ground to read and query a syntax tree."
+++

[tree-sitter](https://tree-sitter.github.io/tree-sitter/) is a pretty cool tool: it's a family of syntax parsers that can bind to most languages. Want to work with Julia syntax in Rust? You can. Want to work with Ruby syntax in Python? Yep, still can! Usually, languages only make their syntax tree tools available to their own language, but tree-sitter's common structure lets you easily pull in new language support for your tools.

Did I say easily? Well, mostly. It takes a minute to get set up with the bindings and flow of using tree-sitter. Here's a quick intro to using tree-sitter to read (but not to edit) code. The actual program is a simple linter that calls out when code has reducible math, like using `1 + 1` instead of `2`.

# The Basics

Every use of tree-sitter follows the same basic start:
1. Create a parser for your language.
2. Parse source code into a syntax tree.
3. Do something with that tree.

Here's a basic function that prints the syntax tree for Rust source code.

```rs
use tree_sitter::Parser;

fn main() {
    let code = "let x = 1 + 1;";

    // Create a parser for your language.
    let mut parser = Parser::new();
    parser
        .set_language(&tree_sitter_rust::LANGUAGE.into())
        .unwrap();

    // Parse source code into a syntax tree.
    let tree = parser.parse(code, None).unwrap();
    
    // Do something with that tree.
    println!("{}", tree.root_node().to_sexp());
}
```

Most of your tree-sitter code will come from the [general crate](https://crates.io/crates/tree-sitter), but you'll also need a crate for each language. In this case, it's [`tree_sitter_rust`](https://crates.io/crates/tree-sitter-rust), which just provides the `LANGUAGE` to the parser.

You'd do step 2 slightly differently if you were going to be editing the code, but I won't do that here.

## The Playground

The above example shows you the tree for some source code, which is generally a useful thing to know when building with tree-sitter. You'll need to know the name and structure of different nodes, and I find it a lot easier to learn by looking at real syntax trees than by digging through a tree-sitter language file.

Fortunately, the tree-sitter project has a general version of that function available in their [playground](https://tree-sitter.github.io/tree-sitter/7-playground.html). You can select one of their many officially supported languages, enter code, and get the tree back. 

The full file's tree is quite long, but here's what you would get out of the playground for just the first line, a `use` statement.
```
source_file [0, 0] - [2, 0]
  use_declaration [0, 0] - [0, 24]
    argument: scoped_identifier [0, 4] - [0, 23]
      path: identifier [0, 4] - [0, 15]
      name: identifier [0, 17] - [0, 23]
```

The source file is the root node. You can see I put in a trailing newline since it goes from the first line (the first zero in `[0, 0]`) and ends right before the third line (the two in `[2, 0]`).

The only thing in this code is the `use_declaration`, which spans the first 24 characters in the first line. The use declaration has one _named child_, `argument`, which is the `scoped_identifier` we want to use: `tree_sitter::Parser`.

The scoped identifier starts four characters in the first line, where those first four characters are the `use ` (space included). The scoped identifier also ends one character before the end of the use declaration, which also contains the ending semicolon. The scoped identifier itself has two named children, corresponding to the path of the identifier and its name.

The playground also lets you click on one of the symbols to see it highlighted in the code sample. 

# Queries

A powerful feature of tree-sitter is its [query engine](https://tree-sitter.github.io/tree-sitter/using-parsers/queries/1-syntax.html), which lets you find specific patterns in the syntax tree. Generally, queries will be how you find parts of the code relevant to your use case.

For an example, let's say you wanted to search for any time you added two integers together. You would want the `1 + 1` from the following code.

```rs
1 + 1;
```

You can use the playground to see that this tree is:
```
source_file [0, 0] - [2, 0]
  expression_statement [0, 0] - [0, 6]
    binary_expression [0, 0] - [0, 5]
      left: integer_literal [0, 0] - [0, 1]
      right: integer_literal [0, 4] - [0, 5]
```

What we want is the `binary_expression` node where both the `left` and `right` children are `integer_literal` nodes.

In tree-sitter's query language, we'd describe this as:
```
(binary_expression left: (integer_literal) right: (integer_literal))
```

If you don't know the symbols you want, try out an example in the tree-sitter playground.

## Creating a Query

You'll start your queries with [`Query::new`](https://docs.rs/tree-sitter/0.26.8/tree_sitter/struct.Query.html#method.new), which requires both the language and the query you want. If you wanted to find the sum of two integers, you might start with:

```rs
let query = Query::new(
    &parser.language().unwrap(),
    "(binary_expression left: (integer_literal) right: (integer_literal)) @math"
).unwrap();
```

Note that you need a name for the nodes you want to capture, here the `@math` following the `binary_expression` node. You can name it whatever you want, as long as it follows an `@`.

For multiple patterns in your queries, you can use a multiline string to have each on one line. You can reuse the same name across patterns if you want.
```rs
let query = Query::new(
    &parser.language().unwrap(),
    r#"
    (binary_expression left: (integer_literal) right: (integer_literal)) @math
    (binary_expression left: (float_literal) right: (float_literal)) @math
    "#
);
```

## Playing with Matches

To run a query, you'll use a cursor to get the results through its [`matches` method](https://docs.rs/tree-sitter/0.26.8/tree_sitter/struct.QueryCursor.html#method.matches).

However, the object returned by `matches` is **not** an iterator! Instead, it's a [`StreamingIterator`](https://docs.rs/tree-sitter/0.26.8/tree_sitter/trait.StreamingIterator.html), which does not play well with Rust's `for` loops. Basically, the Rust `tree-sitter` crate sits on top of C code, and that C code doesn't follow Rust's iterator conventions. When the C code is asked for the next match, it mutates the state of the matches object, which wouldn't be allowed for a Rust iterator. The less-useful `StreamingIterator` is based on what the underlying C library allows.

To get started with iterating over matches, you need to bring `StreamingIterator` into scope.
```rs
use tree_sitter::{QueryCursor, StreamingIterator};
```

Then, you have to iterate using a `while let Some()` loop, rather than a `for` loop.

```rs
let mut cursor = QueryCursor::new();
let mut matches = cursor.matches(&query, tree.root_node(), code.as_bytes());
while let Some(m) = matches.next() {
    for capture in m.captures {
        println!("Found: {}", &code[capture.node.byte_range()]);
    }
}
```

The captured node contains the byte range of the code it covers, so you can index in the original string to get the substring back.

Now this code isn't a very fancy linter; it just prints if it finds anything. But from here, you could build much more complex tools.
