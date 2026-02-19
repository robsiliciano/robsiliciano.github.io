+++
title = "A Quick Introduction to lsp_types"
date = 2026-02-18
+++

I'm a big fan of Rust and the Language Server Protocol (LSP) but I haven't found good documentation on how to use LSP with Rust. I wrote these quick notes on using [`lsp_types`](https://docs.rs/lsp-types/latest/lsp_types/), which is a great crate but with almost no documentation.

# LSP Types

The Language Server Protocol has a pretty simple communication protocol for sending messages between the client and server, so simple you could [do it by hand](@/notes/lsp_by_hand.md). 

The richness of the protocol comes from its many types of messages. You could read through [the specification](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/) and be overwhelmed by the amount of parsing to do. There's so many notifications, requests, and responses to keep track of.

Fortunately, the folks behind [Gluon](https://gluon-lang.org) made a crate called `lsp_types` that encodes the specification in Rust. You don't have to write [custom JSON](@/notes/lsp_by_hand.md#initialize) to send an [Initialize message](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#initializeParams); you can just use their [`Initialize` enum](https://docs.rs/lsp-types/latest/lsp_types/request/enum.Initialize.html), which is serializable with serde.

The crate hasn't been updated in a couple years, but [neither has the Language Server Protocol](https://github.com/Microsoft/language-server-protocol/blob/gh-pages/_specifications/lsp/3.17/specification.md#version_3_17_0). The value of the crate is just encoding the types.

# Notifications

The simplest type of message in LSP is a [notification](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#notificationMessage), which doesn't require any response.

If you want to work with these messages, `lsp_types` has a clever [`Notification` trait](https://docs.rs/lsp-types/latest/lsp_types/notification/trait.Notification.html):
```rs
pub trait Notification {
    type Params: DeserializeOwned + Serialize + Send + Sync + 'static;

    const METHOD: &'static str;
}
```

When you want a notification, like say [Initialized](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#initialized), you would use [`Initialized`](https://docs.rs/lsp-types/latest/lsp_types/notification/enum.Initialized.html), which defines both the method name ("initialized") and the params, [`InitializedParams`](https://docs.rs/lsp-types/latest/lsp_types/struct.InitializedParams.html), which is just an empty struct.

The `lsp_types` crate doesn't give you functions to send or receive these messages, but Rust generics then let you write code that works for any notification. You'll probably need code to turn parameters into a message string, and fortunately all notification param types must be serializable!

```rs
fn notification_to_message<N: Notification>(params: N::Params) -> String {
    let json_body = json!(
        {"jsonrpc": "2.0", "method": N::METHOD, "params": params}
    );
    let body = serde_json::to_string(&json_body).unwrap();
    format!("Content-Length: {}\r\n\r\n{body}", body.len())
}
```

You would then send a message specifying the notification in the turbofish and pass it the params:
```rs
notification_to_message::<Initialized>(InitializedParams {})
```

# Requests

Requests are messages that you send that require a message back, called the response. These messages are a bit more complicated to work with, but fortunately `lsp_types` has the same cool trait trick with [the `Request` trait](https://docs.rs/lsp-types/latest/lsp_types/request/trait.Request.html):

```rs
pub trait Request {
    type Params: DeserializeOwned + Serialize + Send + Sync + 'static;
    type Result: DeserializeOwned + Serialize + Send + Sync + 'static;

    const METHOD: &'static str;
}
```

This trait mirrors the `Notification` trait, except there's now also a `Result` type representing what the server would send back after receiving a request.

Creating a message for a result is similar to creating one for a notification, except you need to include an ID to identify the response.

```rs
fn request_to_message<R: Request>(params: R::Params, request_id: i32) -> String {
    let json_body = json!(
        {"jsonrpc": "2.0", "id": request_id, "method": R::METHOD, "params": params}
    );
    let body = serde_json::to_string(&json_body).unwrap();
    format!("Content-Length: {}\r\n\r\n{body}", body.len())
}
```

Say you wanted to send the [Initialize](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#initialize) request. You'd again use the turbofish for the request and pass in the params. In this case, it's [`InitializeParams`](https://docs.rs/lsp-types/latest/lsp_types/struct.InitializeParams.html). There's now all sorts of optional fields, but you can just use the default.

```rs
request_to_message::<Initialize>(
    InitializeParams {
        ..Default::default()
    },
    0,
)
```

Or maybe you want to send some values in the request.

```rs
request_to_message::<Initialize>(
    InitializeParams {
        client_info: ClientInfo {
            name: "Client Name".to_string(),
            version: None,
        },
        ..Default::default()
    },
    0,
)
```

# Responses

The last message category in the Language Server Protocol is responses, but there's no new trait for them. They were already included in the `Request` trait's `Result` type. These are all deserializable, so you can parse them from the JSON response body.

You'll know which type to use from the response's ID, which matches the ID you provided when sending the request.

```rs
fn try_message_to_response<R: Request>(response: Value, request_id: i32) -> Option<R::Result> {
    if response.get("id") == Some(&json!(request_id)) {
        if let Some(result) = response.get("result") {
            return serde_json::from_value::<R::Result>(result.clone()).ok();
        } else {
            return None;
        }
    } else {
        None
    }
}
```

You're then free to access structured data, using the nice type definitions of `lsp_types`!

# Next Steps

You need more than just `lsp_types` to work with LSP in Rust. I've shown some serialization code here, which is pretty simple. The hard part comes with inter-process communication, possibly getting into async Rust. I think there's a lot of ways you could go there, but at least you'll have all the type definitions ready to go!
