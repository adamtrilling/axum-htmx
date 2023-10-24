# axum-htmx

<!-- markdownlint-disable -->
<div align="right">
<a href="https://crates.io/crates/axum-htmx">
    <img src="https://img.shields.io/crates/v/axum-htmx?style=flat-square" alt="crates.io badge">
</a>
<a href="https://docs.rs/axum-htmx/latest/">
    <img src="https://img.shields.io/docsrs/axum-htmx?style=flat-square" alt="docs.rs badge">
</a>
</div>
<br>
<!-- markdownlint-enable -->

`axum-htmx` is a small extension library providing extractors, responders, and
 request guards for all [htmx](https://htmx.org/) headers within
 [axum](https://github.com/tokio-rs/axum). Additionally, the library exports
 const values for all of the htmx headers, so there's no need to mess with
 strings in your handlers.

## Getting Started

Simply run `cargo add axum-htmx` to add the library to your project.

If you are using the unreleased branch of `axum` from GitHub, you can build
against the `main` version of `axum-htmx` by adding the following to your
`Cargo.toml`:

```toml
[dependencies]
axum-htmx = { git = "https://github.com/robertwayne/axum-htmx" }
```

## Extractors

All of the [htmx request headers](https://htmx.org/reference/#request_headers)
have a supported extractor. Extractors are infallible, meaning they will always
succeed and never return an error. In the case where a header is not present,
the extractor will return `None` or `false` dependant on the expected return
type.

| Header                       | Extractor                 | Value            |
|------------------------------|---------------------------|------------------|
| `HX-Boosted`                 | `HxBoosted`               | `bool`           |
| `HX-Current-URL`             | `HxCurrentUrl`            | `Option<String>` |
| `HX-History-Restore-Request` | `HxHistoryRestoreRequest` | `bool`           |
| `HX-Prompt`                  | `HxPrompt`                | `Option<String>` |
| `HX-Request`                 | `HxRequest`               | `bool`           |
| `HX-Target`                  | `HxTarget`                | `Option<String>` |
| `HX-Trigger-Name`            | `HxTriggerName`           | `Option<String>` |
| `HX-Trigger`                 | `HxTrigger`               | `Option<String>` |

## Responders

All of the [htmx response headers](https://htmx.org/reference/#response_headers)
have a supported responder. A responder is a basic type that implements
`IntoResponseParts`, allowing you to simply and safely apply the HX-* headers to
any of your responses.

| Header                    | Responder                      | Value                                    |
|---------------------------|--------------------------------|------------------------------------------|
| `HX-Location`             | `HxLocation`                   | `axum::http::Uri`                        |
| `HX-Push-Url`             | `HxPushUrl`                    | `axum::http::Uri`                        |
| `HX-Redirect`             | `HxRedirect`                   | `axum::http::Uri`                        |
| `HX-Refresh`              | `HxRefresh`                    | `bool`                                   |
| `HX-Replace-Url`          | `HxReplaceUrl`                 | `axum::http::Uri`                        |
| `HX-Reswap`               | `HxReswap`                     | `axum_htmx::responders::SwapOption`      |
| `HX-Retarget`             | `HxRetarget`                   | `String`                                 |
| `HX-Reselect`             | `HxReselect`                   | `String`                                 |
| `HX-Trigger`              | `HxResponseTrigger`            | `String` or `axum_htmx::serde::HxEvent`* |
| `HX-Trigger-After-Settle` | `HxResponseTriggerAfterSettle` | `String` or `axum_htmx::serde::HxEvent`* |
| `HX-Trigger-After-Swap`   | `HxResponseTriggerAfterSwap`   | `String` or `axum_htmx::serde::HxEvent`* |

_* requires the `serde` feature flag to be enabled._

## Request Guards

__Requires features `guards`.__

In addition to the extractors, there is also a route-wide layer request guard
for the `HX-Request` header. This will redirect any requests without the header
to "/" by default.

_It should be noted that this is NOT a replacement for an auth guard. A user can
trivially set the `HX-Request` header themselves. This is merely a convenience
for preventing users from receiving partial responses without context. If you
need to secure an endpoint you should be using a proper auth system._

### Example: Extractors

In this example, we'll look for the `HX-Boosted` header, which is set when
applying the [hx-boost](https://htmx.org/attributes/hx-boost/) attribute to an
element. In our case, we'll use it to determine what kind of response we send.

When is this useful? When using a templating engine, like
[minijinja](https://github.com/mitsuhiko/minijinja), it is common to extend
different templates from a `_base.html` template. However, htmx works by sending
partial responses, so extending our `_base.html` would result in lots of extra
data being sent over the wire.

If we wanted to swap between pages, we would need to support both full template
responses and partial responses _(as the page can be accessed directly or
through a boosted anchor)_, so we look for the `HX-Boosted` header and extend
from a `_partial.html` template instead.

```rust
use axum::response::IntoResponse;
use axum_htmx::HxBoosted;

async fn get_index(HxBoosted(boosted): HxBoosted) -> impl IntoResponse {
    if boosted {
        // Send a template extending from _partial.html
    } else {
        // Send a template extending from _base.html
    }
}
```

### Example: Responders

We can trigger any event being listened to by the DOM using an [htmx
trigger](https://htmx.org/attributes/hx-trigger/) header.

```rust
use axum_htmx::HxResponseTrigger;

// When we load our page, we will trigger any event listeners for "my-event.
async fn index() -> (&'static str, HxResponseTrigger) {
    (
        "Hello, world!",
        HxResponseTrigger(vec!["my-event".to_string()]),
    )
}
```

...`htmx` even allow arbitrary data to be sent along with the event, which we
can use via the `serde` feature flag and the `HxEvent` type.

```rust
use axum_htmx::HxEvent;

// Note that we are using `HxResponseTrigger` from the `axum_htmx::serde` module
// instead of the root module.
use axum_htmx::serde::HxResponseTrigger;

async fn index() -> (&'static str, HxResponseTrigger) {
    let event = HxEvent::new_with_data(
        "my-event",
        json!({"level": "info", "message": {
            "title": "Hello, world!",
            "body": "This is a test message.",
        }}),
    )
    .unwrap();

    ("Hello, world!", HxResponseTrigger(event))
}
```

### Example: Router Guard

```rust
use axum::Router;
use axum_htmx::HxRequestGuardLayer;

fn router_one() -> Router {
    Router::new()
        // Redirects to "/" if the HX-Request header is not present
        .layer(HxRequestGuardLayer::default())
}

fn router_two() -> Router {
    Router::new()
        .layer(HxRequestGuardLayer::new("/redirect-to-this-route"))
}
```

### Feature Flags

<!-- markdownlint-disable -->
| Flag     | Default  | Description                                                                        | Dependencies                                |
|----------|----------|------------------------------------------------------------------------------------|---------------------------------------------|
| `guards` | Disabled | Adds request guard layers.                                                         | `tower`, `futures-core`, `pin-project-lite` |
| `serde`  | Disabled | Adds serde support for the `HxResponseTrigger*` and `HxLocation` response headers. | `serde`, `serde_json`                       |
<!-- markdownlint-enable -->

## Contributing

Contributions are always welcome! If you have an idea for a feature or find a
bug, let me know. PR's are appreciated, but if it's not a small change, please
open an issue first so we're all on the same page!

## License

`axum-htmx` is dual-licensed under either

- **[MIT License](/LICENSE-MIT)**
- **[Apache License, Version 2.0](/LICENSE-APACHE)**

at your option.
