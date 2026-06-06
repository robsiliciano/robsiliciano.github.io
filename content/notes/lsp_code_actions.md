+++
title = "Getting Started with LSP: Code Actions"
date = "2026-06-06"
description = "Many language servers offer code actions, which are automated code edits like refactoring or importing a required symbol. However, code actions can do much more and offer pretty much arbitrary automated code editing in any editor that supports LSP. Here are my notes on implementing a custom code action."
+++


Modern code editors and IDEs can all make some common code refactors without any AI, like adding missing imports or match arms. These features generally come from language servers providing code actions that any IDE can use. Any time you want to complete match arms in Rust, it's probably `rust-analyzer`, the Rust language server, doing the work. IDEs can all communicate with the server using the same protocol.

Whenever you want to put a feature into every editor, code actions are a good way to start. They're much more flexible than the common actions most people use, and can pretty much cover anything that involves editing the code.

I've been working on a language server that can insert random numbers into code so you don't have to use [low-entropy seeds like `0`, `42`, or the current date](@/notes/random_bits.md). I took these notes on how to get a simple Rust language server with [`lsp-types`](@/notes/intro_to_lsp_types.md) running code actions. My goal was to provide a simple action to insert 128 bits of randomness as a hexadecimal string at the current cursor.

# Getting Started

As with most LSP projects, the first step to support code actions is to add them to the server capabilities. In this case, you provide a [`CodeActionProviderCapability`](https://docs.rs/lsp-types/latest/lsp_types/enum.CodeActionProviderCapability.html) for the `code_action_provider` so the client knows your server has code actions.

```rs
InitializeResult {
    server_info: None,
    capabilities: ServerCapabilities {
        code_action_provider: Some(CodeActionProviderCapability::Simple(true)),
        ..Default::default()
    },
}
```

I've chosen the simple option, but you can provide more [complex capabilities](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#codeActionOptions) if you want.

# The Request

The code action support gets started when the client sends a [code action request](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#textDocument_codeAction), represented by this [type](https://docs.rs/lsp-types/latest/lsp_types/request/enum.CodeActionRequest.html). The client will tell the server two key pieces of information: the current file and the range of code for the actions. (The client may also send some other context, but ignore that for now).

The client expects a list back, containing [edits](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#workspaceEdit) and [custom commands](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#command). I'm mostly interested in the edits, which are executed by the client. The protocol also lets you define commands, which are usually executed by the server to provide functionality not available through the edits. (In some cases, the client may execute commands itself).

In this case, I just want to send back a singleton. Starting with the [code action params](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#codeActionParams) in `params`, the code to get the response looks like this.

```rs
let changes = HashMap::from([(
    params.text_document.uri,
    vec![TextEdit {
        range: params.range,
        new_text,
    }],
)]);

vec![CodeActionOrCommand::CodeAction(CodeAction {
    title: "Insert a random number".to_string(),
    edit: Some(WorkspaceEdit {
        changes: Some(changes),
        ..Default::default()
    }),
    ..Default::default()
})]
```

The protocol supports multiple edits across multiple files at once, but I only want a single insertion of some `new_text` at the given range in the given document.

# Randomness in Rust

The Rust standard library doesn't have strong random number support, but fortunately the [`rand` crate](https://crates.io/crates/rand) covers a wide range of algorithms and outputs. You can see [their book](https://rust-random.github.io/book/intro.html) for the full feature list, but I'll just use a small slice for this project.

The [`getrandom` crate provides the minimal interface](https://docs.rs/getrandom/latest/getrandom/fn.fill.html) for random numbers, pulled from the OS random number source. If we're lucky, this OS source is hardware randomness, but it's not guaranteed. Whatever the source is, we can just fill a buffer with random bytes, then convert as needed. I'm using 16 bytes here since that corresponds to 128 random bits.

```rs
let mut buf = [0u8; 16];
getrandom::fill(&mut buf)?;
```

You can convert these bytes to anything you want. For instance, you could create a `u128` if you pick the [endianness](https://doc.rust-lang.org/std/primitive.u128.html#method.from_be_bytes). In this case, I need a hexadecimal `String`, so we'll `write!` each byte into two characters. Note that the byte may need [zero padding](https://doc.rust-lang.org/std/fmt/#sign0), as we want to render `0x01` as `01` not as `1`, which would mess with randomness by reducing the state space!

```rs
let mut new_text = String::with_capacity(32);
for b in buf {
    write!(&mut new_text, "{:02x}", b)?;
}
```

Make sure to have [`std::fmt::Write`](https://doc.rust-lang.org/std/fmt/trait.Write.html) in scope anytime you use the `write!` macro with a `String`.

Anyways, we've got a string to insert, and can get back to the LSP of it all and run some code actions.

# Trying it Out

I find it easiest to test my language servers with Neovim, where a short addition to `~/.config/nvim/init.lua` can run the language server for Rust files.

```lua
vim.lsp.config['language-server'] = {
	cmd = { '/path/to/language/server' },
	filetypes = { 'rust' },
}
vim.lsp.enable('language-server')
```

If everything worked out, the code action will be available. Check with `:lua vim.lsp.buf.code_action()` and you should see some new text inserted at your cursor. My code action inserts random hexadecimals, but you can add any text or even make more complicated edits!
