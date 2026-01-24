+++
title = "LSP By Hand"
date = "2026-01-24"
+++

# LSP

My favorite change to code editors over the years is probably the Language Server Protocol (LSP). It's made the editing experience better and enabled some great [new](https://helix-editor.com) [editors](https://zed.dev).

In the old days, each editor needed to implement its own language support for each individual language. I've heard it described as an `N * M` situation where `N` editors and `M` languages need `N * M` implementations. Then, LSP came along and defined a "language server" that would provide the key building blocks of language support to an editor. A single language server can power the language features in any editor that supported LSP. Now, you only need `N + M` implementations of LSP, one per editor and one language server for each language. In [some cases](https://rust-analyzer.github.io), those language servers were even built by the people making the language, who presumably can do a thorough job.

I was curious how it all worked, so I tried to take the place of a code editor and use LSP to communicate with a language server by hand.

# Talking to the Server

You can communicate with the language server by just typing directly into your terminal if you really wanted to, but the experience isn't so great. One problem is that standard input and output will be in the same space, so your messages to the server and the server's messages back get messed up.

For a quality-of-life improvement, I'm going to use a pipe to separate the server from the input text. You've probably used a pipe before as `|` between two shell commands. That kind of pipe has the same problem where stdin and stdout are in the same place, but we can instead put a pipe in the filesystem, then use it to pipe between terminals!

You can create a filesystem-based pipe with `mkfifo`.

```sh
mkfifo lsp
```

There's now a special file `lsp` that we'll use to send messages to the server.

In one terminal, you can send a program's output to the pipe (`echo hello > lsp`) and see it in another terminal (`cat lsp`). We'll use the first terminal for typing in LSP messages, and the second terminal for running the server.

# Running the Server

I'll use [Astral's `ty`](https://docs.astral.sh/ty/) but you can use any LSP server that reads from standard input. You can run the server with:
```sh
ty server
```

`ty` will listen to stdin and print its responses to stdout. We're going to want to replace stdin, so use `cat` to send our pipe into the server's standard input.

```sh
cat lsp | ty server
```

# Sending LSP Messages

An [LSP message](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#baseProtocol) has two parts: a header and a JSON body. For the most part, the header is pretty simple: it just says the length of the body.

We could manually add the `Content-Length` header, but to make things easier, I'll just use a [simple `awk` script](https://www.gnu.org/software/gawk/manual/gawk.html) (really a `gawk` script because of the `fflush`). If you were going to type in the whole message directly, note that you also need to type carriage returns (`\r`) in addition to your newlines (`\n`).

```sh
awk '{ printf "Content-Length: %d\r\n\r\n%s", length($0), $0}; fflush("")' > lsp
```

When you run this, you should be able to type in each line, then `awk` will add the header and send it to the LSP.

You can now try typing something, anything! Unless you happened to type the exact correct message, you'll probably see `ty` quit with:
```
ty failed
  Cause: Failed to start server
  Cause: disconnected channel
```

That's okay though. Let's now get into properly sending messages.

# LSP Lifecycle

Any LSP session has a specific [lifecycle](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#lifeCycleMessages) with four required messages: two to start it up and two to shut it down.

## Initialize

The first thing we need to do is tell the server to [initialize](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#initialize). The minimal message we can send looks like:

```
{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"capabilities":{}}}
```

`capabilities` is a required field, but we can leave it empty since we're not going to have any client capabilities here. We also need to have an `id`, which is how the protocol matches our requests with a server response.

After sending this, `ty` should print a response like:

```
Content-Length: 1335

{"jsonrpc":"2.0","id":1,"result":{"capabilities":{"codeActionProvider":{"codeActionKinds":["quickfix"]},"completionProvider":{"triggerCharacters":["."]},"declarationProvider":true,"definitionProvider":true,"diagnosticProvider":{"identifier":"ty","interFileDependencies":true,"workDoneProgress":true,"workspaceDiagnostics":true},"documentHighlightProvider":true,"documentSymbolProvider":true,"executeCommandProvider":{"commands":["ty.printDebugInformation"],"workDoneProgress":false},"hoverProvider":true,"inlayHintProvider":{},"notebookDocumentSync":{"notebookSelector":[{"cells":[{"language":"python"}]}],"save":false},"positionEncoding":"utf-16","referencesProvider":true,"renameProvider":{"prepareProvider":true},"selectionRangeProvider":true,"semanticTokensProvider":{"full":true,"legend":{"tokenModifiers":["definition","readonly","async","documentation"],"tokenTypes":["namespace","class","parameter","selfParameter","clsParameter","variable","property","function","method","keyword","string","number","decorator","builtinConstant","typeParameter"]},"range":true},"signatureHelpProvider":{"retriggerCharacters":[")"],"triggerCharacters":["(",","]},"textDocumentSync":{"change":2,"openClose":true},"typeDefinitionProvider":true,"workspaceSymbolProvider":true},"serverInfo":{"name":"ty","version":"0.0.13 (fc1478bd9 2026-01-21)"}}}
```

`ty` will probably also warn about the lack of a workspace or certain capabilities.

You can see that there's an `id` in the response, which matches the `id` we provided in the message.

## Initialized

Next, we tell the server that we've [initialized](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#initialized). This message body is as simple as possible. 

```
{"jsonrpc":"2.0","method":"initialized","params":{}}
```

Since this is just a notification, not a request, we don't provide any `id` and don't expect a response. At this point, we've got a running language server!

## Shutdown

We're not going to do anything with the language server, so it's time to tell it to [shutdown](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#shutdown). This message is a request, so we need an `id`.

```
{"jsonrpc":"2.0","id":2,"method":"shutdown"}
```

The server will send something back like:
```
{"jsonrpc":"2.0","id":2,"result":null}
```

You'll see that the server hasn't shut down yet! We need one more message to get there.

## Exit

The final part of LSP is to send an [exit](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#exit) notification.

```
{"jsonrpc":"2.0","method":"exit"}
```

After this, `ty` should exit without an error, and you can send an EOF to `awk`.

Congrats! You've just had a complete LSP session by yourself!
