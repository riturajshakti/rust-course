[← Course Index](../README.md)

# 17.2 — Your First Axum Server

## What You'll Learn

- The minimal axum server: `Router`, `route`, `TcpListener`, `axum::serve`
- Every type a handler can return, and the HTTP response each produces
- Why handlers must be `async`, and the `Handler` trait error when they aren't
- Project structure, hot reload, and the default 404/405 behaviour

## Coming From Another Language

| Other | Rust / axum |
|---|---|
| `app.get('/', (req, res) => ...)` | `.route("/", get(handler))` |
| `app.listen(3000)` | `axum::serve(listener, app).await` |
| FastAPI `@app.get("/")` | `async fn handler() -> impl IntoResponse` |
| Go `http.HandleFunc` + `ListenAndServe` | same two pieces, one `.await` |

The structural difference: axum handlers don't take a `req` and a `res`. They
take **extractors** as typed arguments and **return** a response value. There is
no `res.send()` — the return value *is* the response. That makes every handler
an ordinary testable function, which lesson 17.13 leans on hard.

## Setup

```bash
cargo add axum
cargo add tokio --features full
```

```toml
axum = "0.8.9"
tokio = { version = "1.54.1", features = ["full"] }
```

Axum 0.8 is the version used throughout this module. Route syntax and a few
extractor names changed from 0.7 — check `Cargo.toml` before copying snippets
off the internet.

## The smallest server that works

```rust
use axum::{Router, routing::get};

async fn root() -> &'static str {
    "Hello, axum!"
}

#[tokio::main]
async fn main() {
    let app = Router::new().route("/", get(root));

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000").await.unwrap();
    println!("listening on http://{}", listener.local_addr().unwrap());
    axum::serve(listener, app).await.unwrap();
}
```

Four moving parts:

- `#[tokio::main]` turns `async fn main` into a sync `main` that starts the
  runtime (lesson 15.2).
- `Router` maps paths to method routers.
- `get(root)` is a `MethodRouter` — a route that answers GET with `root`.
- `axum::serve` drives the accept loop until the process ends.

Binding port `0` lets the OS pick a free port, which is how the examples below
were captured:

```rust
let listener = tokio::net::TcpListener::bind("127.0.0.1:0").await.unwrap();
println!("listening on http://{}", listener.local_addr().unwrap());
```

Output:

```
listening on http://127.0.0.1:51973
```

```console
$ curl -s localhost:51973/
Hello, axum!
```

With headers:

```console
$ curl -si localhost:51973/
HTTP/1.1 200 OK
content-type: text/plain; charset=utf-8
content-length: 12
date: Sun, 25 Oct 2026 16:56:58 GMT

Hello, axum!
```

Axum set the status, content type, and length from the fact that you returned a
`&'static str`. Nothing else is configured.

## Handlers must be async

```rust
fn sync_handler() -> &'static str { "hi" }
Router::new().route("/", get(sync_handler));
```

Does not compile:

```
error[E0277]: the trait bound `fn() -> &'static str {sync_handler}: Handler<_, _>` is not satisfied
 --> src/main.rs:5:52
  |
5 |     let app: Router = Router::new().route("/", get(sync_handler));
  |                                                --- ^^^^^^^^^^^^ the trait `Handler<_, _>` is not implemented for fn item `fn() -> &'static str {sync_handler}`
  |
  = note: Consider using `#[axum::debug_handler]` to improve the error message
```

Intentionally broken. The fix is `async fn`. The `Handler` trait, in one line:
it is implemented automatically for any `async fn` whose arguments all implement
an extractor trait and whose return type implements `IntoResponse`. You never
write `impl Handler` yourself — you just satisfy those two conditions, and the
"not satisfied" error above means one of them failed.

## What a handler can return

Anything implementing `IntoResponse`. The important implementations:

```rust
use axum::{Json, http::StatusCode, response::Html};
use serde_json::json;

async fn root() -> &'static str { "Hello, axum!" }
async fn owned() -> String { format!("uptime {}s", 42) }
async fn html() -> Html<&'static str> { Html("<h1>Hi</h1>") }
async fn created() -> StatusCode { StatusCode::CREATED }
async fn with_body() -> (StatusCode, &'static str) { (StatusCode::NOT_FOUND, "no such thing") }
async fn as_json() -> Json<serde_json::Value> { Json(json!({ "ok": true, "count": 3 })) }
async fn unit() {}
```

Each one, hit with curl:

```console
$ curl -si localhost:51973/html
HTTP/1.1 200 OK
content-type: text/html; charset=utf-8
content-length: 11
date: Sun, 25 Oct 2026 16:56:58 GMT

<h1>Hi</h1>
```

```console
$ curl -si -X POST localhost:51973/created
HTTP/1.1 201 Created
content-length: 0
date: Sun, 25 Oct 2026 16:56:58 GMT
```

```console
$ curl -si localhost:51973/missing
HTTP/1.1 404 Not Found
content-type: text/plain; charset=utf-8
content-length: 13
date: Sun, 25 Oct 2026 16:56:58 GMT

no such thing
```

```console
$ curl -si localhost:51973/json
HTTP/1.1 200 OK
content-type: application/json
content-length: 21
date: Sun, 25 Oct 2026 16:56:58 GMT

{"count":3,"ok":true}
```

```console
$ curl -si localhost:51973/unit
HTTP/1.1 200 OK
content-length: 0
date: Sun, 25 Oct 2026 16:56:58 GMT
```

Note `Html` and `Json` set the content type; `&str`/`String` set
`text/plain`; `()` sets nothing and returns an empty 200.

## Tuples compose a response

The tuple forms, in order of how much they let you set:

| Return type | Sets |
|---|---|
| `T` | body (and its default content type) |
| `(StatusCode, T)` | status + body |
| `(StatusCode, HeaderMap, T)` | status + headers + body |
| `(StatusCode, [(HeaderName, &str); N], T)` | status + literal headers + body |

```rust
use axum::http::{StatusCode, header};

async fn download() -> (StatusCode, [(header::HeaderName, &'static str); 1], &'static str) {
    (StatusCode::OK, [(header::CONTENT_DISPOSITION, "attachment; filename=\"a.txt\"")], "data")
}
```

The rule: the status comes first, headers in the middle, body last. Put them in
any other order and you get an `IntoResponse` trait error.

## Returning `Result`

`Result<T, E>` is `IntoResponse` when both halves are. The `Err` branch becomes
the response — which is how `?` works in handlers.

```rust
use axum::extract::Path;

async fn maybe(Path(n): Path<u32>) -> Result<String, StatusCode> {
    if n % 2 == 0 { Ok(format!("{n} is even")) } else { Err(StatusCode::BAD_REQUEST) }
}
```

```console
$ curl -s localhost:51973/maybe/4
4 is even

$ curl -si localhost:51973/maybe/5
HTTP/1.1 400 Bad Request
content-length: 0
date: Sun, 25 Oct 2026 16:56:58 GMT
```

A bare `StatusCode` error is fine for a toy. Lesson 17.6 replaces it with a
proper `AppError` enum that carries a JSON body.

## Taking the body

An argument of type `String` consumes the request body as text.

```rust
async fn echo(body: String) -> String {
    format!("you said: {body}")
}
```

```console
$ curl -s -X POST localhost:51973/echo -d 'hello there'
you said: hello there
```

Use `axum::body::Bytes` for binary, `Json<T>` for JSON (lesson 17.4).

## Methods and the router

```rust
use axum::routing::{get, post};

let app = Router::new()
    .route("/", get(root))
    .route("/html", get(html))
    .route("/created", post(created))
    .route("/maybe/{n}", get(maybe))
    .route("/echo", post(echo));
```

`get`, `post`, `put`, `patch`, `delete`, `head`, `options`, `trace`, plus `any`
for all of them. Path parameters use `{name}` braces in axum 0.8 — the `:name`
colon syntax from 0.7 and earlier is gone and will panic at startup.

## Default 404 and 405

Both come free. A path with no route:

```console
$ curl -si localhost:51973/nope
HTTP/1.1 404 Not Found
content-length: 0
date: Sun, 25 Oct 2026 16:56:58 GMT
```

A path that exists on another method — note axum supplies the `allow` header:

```console
$ curl -si localhost:51973/created
HTTP/1.1 405 Method Not Allowed
allow: POST
content-length: 0
date: Sun, 25 Oct 2026 16:56:58 GMT
```

Both bodies are empty. Lesson 17.3 covers `.fallback()` to give them content.

## Binding and addresses

```rust
tokio::net::TcpListener::bind("127.0.0.1:3000")  // localhost only
tokio::net::TcpListener::bind("0.0.0.0:3000")    // all interfaces — needed in Docker
tokio::net::TcpListener::bind("127.0.0.1:0")     // OS picks a free port
```

Inside a container, binding `127.0.0.1` means nothing outside the container can
reach you — use `0.0.0.0`. Read the port from the environment in real code:

```rust
let port: u16 = std::env::var("PORT").ok()
    .and_then(|s| s.parse().ok())
    .unwrap_or(3000);
let listener = tokio::net::TcpListener::bind(("0.0.0.0", port)).await?;
```

## Project structure for a server

A layout that scales without a framework telling you to:

```
src/
  main.rs        # config, pool, router assembly, serve
  routes/
    mod.rs       # pub fn router() -> Router<AppState>
    users.rs     # handlers for /users
    health.rs
  models.rs      # domain structs, FromRow
  dto.rs         # request/response shapes (serde)
  error.rs       # AppError + IntoResponse
  state.rs       # AppState
migrations/
tests/
  api.rs         # integration tests
```

Keep `main.rs` to wiring only, and have each route module expose one function
returning a `Router`:

```rust
pub fn router() -> Router<AppState> {
    Router::new()
        .route("/users", get(list).post(create))
        .route("/users/{id}", get(show).delete(remove))
}
```

Then `main.rs` is `Router::new().merge(routes::users::router())`. Multiple
methods on one path chain off a single `.route` call, as shown.

## Hot reload

Not built in. Install a watcher:

```bash
cargo install cargo-watch
cargo watch -x run
```

Or `bacon` for a nicer TUI. Either way a Rust rebuild is seconds, not
milliseconds — keep `cargo check` in the loop for fast feedback and restart on
`run` only when you actually want to hit the endpoint:

```bash
cargo watch -x check -x run
```

## Graceful shutdown, briefly

```rust
axum::serve(listener, app)
    .with_graceful_shutdown(async {
        tokio::signal::ctrl_c().await.ok();
    })
    .await?;
```

In-flight requests finish before the process exits. Lesson 18.4 does this
properly, including SIGTERM.

## Quick Reference

| Piece | Code |
|---|---|
| Router | `Router::new()` |
| One route | `.route("/p", get(h))` |
| Multiple methods | `.route("/p", get(h).post(h2))` |
| Path param | `.route("/p/{id}", get(h))` |
| Listener | `tokio::net::TcpListener::bind(addr).await?` |
| Serve | `axum::serve(listener, app).await?` |
| Return text | `-> &'static str` / `-> String` |
| Return HTML | `-> Html<&'static str>` |
| Return JSON | `-> Json<T>` |
| Status only | `-> StatusCode` |
| Status + body | `-> (StatusCode, T)` |
| Fallible | `-> Result<T, E>` where both are `IntoResponse` |
| Raw body | argument of type `String` or `Bytes` |
| Hot reload | `cargo watch -x run` |

## Common Mistakes

**Writing a non-`async` handler.** You get `the trait bound ...: Handler<_, _>
is not satisfied`. Add `async`.

**Using `:id` for path params.** That is axum 0.7 syntax; 0.8 uses `{id}` and
panics at startup otherwise.

**Forgetting `features = ["full"]` on tokio.** `#[tokio::main]` won't exist
without `macros` and `rt-multi-thread`.

**Binding `127.0.0.1` in Docker.** Nothing outside the container can connect.
Bind `0.0.0.0`.

**Putting the status after the body in a tuple.** `(T, StatusCode)` is not an
`IntoResponse`; status first, body last.

**Reaching for `res.send()`.** There is no response object. Return the value.

## Exercises

1. Build a server on port 0 that prints its chosen port, with `GET /health`
   returning `Json(json!({"status":"ok"}))`. Hit it with `curl -i` and confirm
   `content-type: application/json`.
2. Add `POST /shout` that reads the body as `String` and returns it uppercased
   with status 201.
3. Add `GET /teapot` returning `(StatusCode::IM_A_TEAPOT, "short and stout")`
   and verify the status line.
4. Chain `get` and `delete` on the same `/item` route with two different
   handlers. Then `curl -X PUT` it and read the `allow` header axum produces.
5. Split your handlers into `src/routes/mod.rs` exposing `pub fn router() ->
   Router`, and reduce `main.rs` to listener setup plus `.merge(routes::router())`.

---

[← Course Index](../README.md)
