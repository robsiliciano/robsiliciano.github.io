+++
title = "Getting Started with LSP: Syntax Highlighting"
date = "2026-03-14"
description = "Want to write your own functional language server? Here's the bare minimum needed for a language server to provide syntax highlighting through semantic tokens, usable in any modern editor or IDE. Performance isn't great, but hey, it works!"
+++

One of my favorite features of a language server is that it can provide consistent syntax highlighting across any editor that supports the Language Server Protocol (LSP). These days, your syntax highlighting usually comes from either LSP or [tree-sitter](https://tree-sitter.github.io/tree-sitter/), whether you're using [Zed](https://zed.dev/docs/semantic-tokens), [VS Code](https://code.visualstudio.com/api/language-extensions/semantic-highlight-guide) or its friends, or [Neovim](https://neovim.io/doc/user/lsp/#lsp-semantic-highlight).

I think semantic tokens are a great place to begin when writing a language server. It only requires minimal work to get something off the ground: you have to handle LSP messaging and document management in any case, so the only additional task is syntax highlighting.

Well, good semantic tokens need more work, but for a first language server, I think it's fine to take some performance shortcuts. Here're my notes on going from zero to one on a language server.

I'll skip over the boilerplate, but the architecture of the server can be really simple: You can use [`lsp-types`](@/notes/intro_to_lsp_types.md) for data structure, and the messaging protocol is [so simple you could type it by hand](@/notes/lsp_by_hand.md)! I've chosen enough simplifications that you don't have to worry about async calls or much state management. Just keep one `HashMap<Uri, String>` to track documents and process messages one at a time as they come.

# Semantic Tokens

Let's get to the meat of syntax highlighting: providing [semantic tokens](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#textDocument_semanticTokens) to the LSP client. The tokens are just the location in the document and the type of token you want to provide.

I'll assume you have a syntax-highlighting function ready to go.
```rs
fn syntax_highlighter(text: &str) -> Vec<SemanticToken> {
    ...
}
```

I'll talk about what `SemanticToken` means later.

## Legends

First, your server tells the client (the editor) what tokens it can handle. We don't even have a document open yet, but we need to spell out all the options in the [legend](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#semanticTokensLegend). The semantic token legend is basically just a list of token names. There's also possibly a list of token modifiers, but let's ignore that for now.

We'll refer to tokens using their index in the legend rather than by their name. If your legend is `["comment", "struct"]` then you'll refer to some `"struct"` code by the index `1`.

With the [`lsp-types` crate](@/notes/intro_to_lsp_types.md), you'd create a legend like so:

```rs
SemanticTokensLegend {
    token_types: vec![SemanticTokenType::COMMENT, SemanticTokenType::STRUCT],
    token_modifiers: vec![],
}
```

## Actually Highlighting

Now that the client knows what we mean when we say `1`, we can actually get to providing those semantic tokens.

The protocol has a couple ways to return tokens, but the easiest one is just to provide [all the tokens](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#semanticTokensLegend) for the document at a time with the `textDocument/semanticTokens/full` request. LSP has more efficient ways to provide tokens than doing everything all at once, but that's beyond my scope here.

You'll need to tell the client that your language server only supports [full text](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#semanticTokensOptions) semantic token queries, right where you also have to provide your legend.

```rs
SemanticTokensServerCapabilities::SemanticTokensOptions(
    SemanticTokensOptions {
        legend: SemanticTokensLegend {
            token_types: vec![SemanticTokenType::COMMENT, SemanticTokenType::STRUCT],
            token_modifiers: vec![],
        },
        full: Some(SemanticTokensFullOptions::Bool(true)),
        ..Default::default()
    },
)
```

Now, the client will send you a [document URI](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#semanticTokensParams), and you can send back tokens!

## What are Tokens Actually?

When it comes to the token array you need to send back, the protocol doesn't have its usual level of polish. Fortunately `lsp-types` provides the [`SemanticToken`](https://docs.rs/lsp-types/latest/lsp_types/struct.SemanticToken.html) struct.

```rs
pub struct SemanticToken {
    pub delta_line: u32,
    pub delta_start: u32,
    pub length: u32,
    pub token_type: u32,
    pub token_modifiers_bitset: u32,
}
```

(In the base protocol, the tokens list is just a chaotic array with everything back to back, from one token's `token_modifiers_bitset` to the next token's `delta_line`.)

I find LSP's representation of tokens a bit unintuitive, because it uses relative position rather than the absolute position in the document. Really, you need four things for a token:
1. Where it starts
2. How far it goes
3. The legend index 
4. Any modifiers (out of scope)

LSP stores the start (1) _relative_ to the previous token, and this delta start is encoded in two numbers: the number of lines since the previous token and the number of characters on the line. A token five characters down on the same line would be `(0, 5)` while a token at the start of a line five lines later would be `(5, 0)`. If you instead gave the absolute position in a document (as I did once) you'll get some weird highlighting that fans out as the gaps between tokens get larger and larger.

Let's say you wanted to highlight comments in the following text:
```c
/* 1 */
int x; /* 2 */ /* 3 */
/* 4 */
```

You'd want four tokens:
1. Starts at line 0, character 0. There's no previous token, so the delta is just the absolute position: `delta_line: 0, delta_start: 0`. Easy!
2. Starts at line 1, character 7. We reset the character count on each line, so the delta is `delta_line: 1, delta_start: 7`.  
3. Starts at line 1, character 15. Even though LSP doesn't allow overlapping tokens, we need to provide the delta from the previous token's _start_, not its _end_. The correct delta is `delta_line: 0, delta_start: 8`.  
4. Starts at line 2, character 0. Remember, it's the number of lines since the last token, which is only one line. You want `delta_line: 1, delta_start: 0`.

In total, you'd want to respond with:
```rs
Some(SemanticTokensResult::Tokens(SemanticTokens {
    result_id: None,
    data: vec![
        SemanticToken {
            delta_line: 0,
            delta_start: 0,
            length: 7,
            token_type: 0,
            token_modifiers_bitset: 0,
        },
        SemanticToken {
            delta_line: 1,
            delta_start: 7,
            length: 7,
            token_type: 0,
            token_modifiers_bitset: 0,
        },
        SemanticToken {
            delta_line: 0,
            delta_start: 8,
            length: 7,
            token_type: 0,
            token_modifiers_bitset: 0,
        },
        SemanticToken {
            delta_line: 1,
            delta_start: 0,
            length: 7,
            token_type: 0,
            token_modifiers_bitset: 0,
        },
    ],
}))
```

A token type of zero means a comment, as defined by our token legend. If you want other tokens, you'd replace the `token_type` with the index in your vector. Also note that it's `Some` because the response type is `Option<SemanticTokensResult>` not just `SemanticTokensResult`.

# Getting Documents

You'll get a document URI with the semantic token request, but **don't** try to read from it! That approach will work the first time an editor asks for tokens in a file, but try editing it and just insert a newline. You'll see that the syntax highlighting doesn't update! If the editor doesn't invalidate the old tokens, you'll see the syntax highlighting shift up a line with no regards for the actual characters. Weird, right?

Once the user changes the file in the editor, we no longer care about syntax highlighting for the file on disk! When the client asks you for semantic tokens for a document, it means the version of the document in its buffer, not the version on disk.

How are you supposed to know what's in the editor's buffer? Usually, it'd be impossible to find, but the LSP provides messages so the server can stay up to date with the client. For the simplest case, we only need to care about three notifications:
1. [`textDocument/didOpen`](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#textDocument_didOpen): sent when the document is first open to tell the server the URI and _full text_.
2. [`textDocument/didChange`](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#textDocument_didChange): sent when edits are made to update the language server.
3. [`textDocument/didClose`](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#textDocument_didClose): sent when the file is closed and the server can stop tracking it.

All these messages are just notifications, so the language server doesn't need to send a response back. All it needs to do is keep an up-to-date version of the document for when the client requests semantic tokens. For the simplest language server, you just need a few lines of code.

```rs
fn did_open(documents: &mut HashMap<Uri, String>, params: DidOpenTextDocumentParams) {
    documents.insert(params.text_document.uri, params.text_document.text);
}

fn did_close(documents: &mut HashMap<Uri, String>, params: DidCloseTextDocumentParams) {
    documents.remove(&params.text_document.uri);
}

fn did_change(documents: &mut HashMap<Uri, String>, params: DidChangeTextDocumentParams) {
    documents.insert(
        params.text_document.uri,
        params.content_changes[0].text.clone(),
    );
}
```

I guess you don't even need `did_close` for the minimal implementation, if you don't worry about clogging up memory with closed documents.

## Requesting Full Text

The simplest way to keep track of the document is to have the client send the whole thing over every time an edit is made. Not efficient, but simple. The other option is to have the client send just the changes, in which case we'd have to care a lot about the `params.content_changes` vec in `did_change`. For a real language server, you'll want to support this, but I'm just focused on the easiest path to a working syntax highlighter now.

You can get the full text configuration by sending the ["full" text document sync kind](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#textDocumentSyncKind) in your [server capabilities](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#serverCapabilities).

```rs
TextDocumentSyncCapability::Kind(
    TextDocumentSyncKind::FULL,
)
```

# The Lifecycle Messages

The last thing you need to handle is the [lifecycle messages](@/notes/lsp_by_hand.md), though all of the tricky stuff is in the first one, [the `Initialize` request](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#initialize). Here's where you have to set up the capabilities I mentioned earlier: the semantic token provider ([the legend and the full semantic token option](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#semanticTokensOptions)) and the ["full" text document sync kind](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#textDocumentSyncKind).

```rs
InitializeResult {
    capabilities: ServerCapabilities {
        semantic_tokens_provider: Some(
            SemanticTokensServerCapabilities::SemanticTokensOptions(
                SemanticTokensOptions {
                    legend: SemanticTokensLegend {
                        token_types: vec![SemanticTokenType::COMMENT],
                        token_modifiers: vec![],
                    },
                    full: Some(SemanticTokensFullOptions::Bool(true)),
                    ..Default::default()
                },
            ),
        ),
        text_document_sync: Some(TextDocumentSyncCapability::Kind(
            TextDocumentSyncKind::FULL,
        )),
        ..Default::default()
    },
    server_info: Some(ServerInfo {
        name: "Your language server name".to_string(),
        version: None,
    }),
}
```

Fortunately, nothing else is that hard to handle. You'll get the `Initialized` and `Exit` notifications, which don't require a response, and the [`Shutdown` request](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#shutdown), which only requires a blank response. If you mess this one up, what's the worst that'll happen? The editor wants the server to close down anyways.

# Testing it Out

Once you've got your language server, you'll want to test it in a real, live editor! I prefer Neovim for testing, since [its language server setup is so fast](https://neovim.io/doc/user/lsp/#lsp-quickstart). Some other editors need a wrapping extension to talk to your language server. Neovim only needs a few lines in `init.lua`:
```lua,name=~/.config/nvim/init.lua
vim.lsp.config['your_ls'] = {
  cmd = { '/path/to/your/language_server' },
  filetypes = { 'filetype' },
  root_markers = { '.git' }
}
vim.lsp.enable('your_ls')
```

Now, you can open a file of your filetype and see some highlighting! You can check on your language server with `:checkhealth vim.lsp`.

Be careful here: the syntax highlighting might not be from your language server. Neovim has other ways to highlight, like tree-sitter or the legacy patterns. I'd recommend trying to open your filetype once before adding your language server to see how much is highlighted already.

You can also use the `:Inspect` command to see information about who is highlighting under the cursor. Your language server should show up in the "Semantic Tokens" category, but watch out for other language servers, tree-sitter, or the legacy syntax highlighting. If you're also seeing these, then it's hard to tell what highlighting is due to your LSP. Fortunately, Neovim has options to turn off these other highlights; for example, you can turn off legacy highlighting with `:set syntax=off`.
