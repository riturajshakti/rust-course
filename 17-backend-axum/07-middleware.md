[← Course Index](../README.md)

# 17.7 — Middleware and tower Layers

## What You'll Learn

- `Service` and `Layer` in one paragraph, and what `ServiceBuilder` fixes
- The tower-http layers worth having on day one
- `middleware::from_fn` for custom middleware, with timing and auth examples
- The onion model, and why `.layer(a).layer(b)` runs in the opposite order to
  `ServiceBuilder`

## Coming From Another Language

| Other | axum / tower |
|---|---|
| `app.use(fn)` | `.layer(middleware::from_fn(f))` |
| `(req, res, next) => { ...; next(); }` | `async fn(req, next) -> Response` |
| Django `MIDDLEWARE = [...]` list | `ServiceBuilder` chain, same top-to-bottom order |
| Go `func(http.Handler) http.Handler` | `tower::Layer` — identical idea, typed |

The one structural difference: axum middleware **returns** the response instead
of mutating a `res` object. Short-circuiting is "return a response instead of
calling `next`", which makes an auth guard three lines.

## `Service` and `Layer`

`tower::Service` is `async fn call(&mut self, Request) -> Result<Response, E>` —
one trait for anything request-to-response. `tower::Layer` wraps a `Service` in
another `Service`. A `Router` is a `Service`; `.layer(l)` replaces it with
`l.layer(router)`.

The practical consequence: **middleware is reusable across frameworks**. Any
tower middleware works in axum, tonic (gRPC), or a reqwest client, including
retry, load-shed, rate-limit, and concurrency-limit from `tower` itself.

You rarely implement `Service` by hand. `middleware::from_fn` covers almost
everything; implement the trait only when you need to hold state across requests
or control `poll_ready` (backpressure).

## Setup

```bash
cargo add tower
cargo add tower-http --features trace,cors,compression-full,timeout,limit,set-header
cargo add tracing tracing-subscriber
```

tower-http is heavily feature-gated — forgetting a feature means the layer type
simply does not exist.

## The layers worth having

```rust
use tower::ServiceBuilder;
use tower_http::{
    compression::CompressionLayer,
    cors::{Any, CorsLayer},
    limit::RequestBodyLimitLayer,
    set_header::SetResponseHeaderLayer,
    timeout::TimeoutLayer,
    trace::TraceLayer,
};

let app = Router::new()
    .route("/hello", get(hello))
    .layer(
        ServiceBuilder::new()
            .layer(TraceLayer::new_for_http())
            .layer(middleware::from_fn(timing))
            .layer(CorsLayer::new().allow_origin(Any).allow_methods(Any))
            .layer(CompressionLayer::new())
            .layer(SetResponseHeaderLayer::overriding(
                header::HeaderName::from_static("x-powered-by"),
                HeaderValue::from_static("axum"),
            ))
            .layer(TimeoutLayer::with_status_code(
                StatusCode::REQUEST_TIMEOUT,
                Duration::from_millis(100),
            ))
            .layer(RequestBodyLimitLayer::new(1024)),
    );
```

| Layer | Does |
|---|---|
| `TraceLayer::new_for_http()` | a structured span + log per request |
| `CorsLayer` | preflight handling and `access-control-*` headers |
| `CompressionLayer` | gzip/br/zstd when the client asks |
| `TimeoutLayer` | aborts the handler, returns 408 |
| `RequestBodyLimitLayer` | 413 above a byte cap |
| `SetResponseHeaderLayer` | a fixed header on every response |
| `CatchPanicLayer` | a panic becomes 500 instead of a dropped connection |
| `RequestIdLayer` / `PropagateRequestIdLayer` | generate and echo `x-request-id` |
| `ServeDir` (via `.nest_service`) | static files |

All of it on one response:

```console
$ curl -si localhost:52262/hello
HTTP/1.1 200 OK
content-type: text/plain; charset=utf-8
x-powered-by: axum
access-control-allow-origin: *
x-response-time-ms: 0
content-length: 5
date: Sun, 25 Oct 2026 17:07:31 GMT

hello
```

## `TimeoutLayer`

```rust
async fn slow() -> &'static str {
    tokio::time::sleep(Duration::from_millis(300)).await;
    "slow done"
}
```

With a 100ms timeout:

```console
$ curl -si localhost:52262/slow | head -4
HTTP/1.1 408 Request Timeout
x-powered-by: axum
access-control-allow-origin: *
x-response-time-ms: 101
```

The handler's future is dropped at 100ms — and that is the detail to remember.
A dropped future means any work after the current `.await` never happens, so a
handler that has already written one of two database rows leaves the second
unwritten. Put multi-step writes in a transaction (lesson 17.9) so a cancelled
request rolls back.

Note `TimeoutLayer::new` is deprecated in tower-http 0.7; use
`with_status_code`, which makes the status explicit.

## `CompressionLayer`

Only compresses when the client advertises support.

```console
$ curl -s -D - -o /dev/null localhost:52262/big | grep -i content-
content-type: text/plain; charset=utf-8
content-length: 2000

$ curl -s -D - -o /dev/null -H 'accept-encoding: gzip' localhost:52262/big
HTTP/1.1 200 OK
content-type: text/plain; charset=utf-8
x-powered-by: axum
vary: accept-encoding
content-encoding: gzip
transfer-encoding: chunked
```

2000 bytes of `x` compress to 37:

```console
$ curl -s -H 'accept-encoding: gzip' --output - localhost:52262/big | wc -c
      37
$ curl -s localhost:52262/big | wc -c
    2000
```

`content-length` disappears in favour of `transfer-encoding: chunked`, because
the compressed length isn't known until the body is streamed. `vary:
accept-encoding` is added so caches don't serve gzip to a client that can't read
it.

## `CorsLayer`

```console
$ curl -si -X OPTIONS localhost:52262/hello \
    -H 'Origin: http://example.com' \
    -H 'Access-Control-Request-Method: GET' | head -6
HTTP/1.1 200 OK
access-control-allow-methods: *
access-control-allow-origin: *
allow: GET,HEAD
content-length: 0
```

`CorsLayer` answers the preflight itself — the request never reaches a handler.
`Any` is for development only; in production name the origins:

```rust
CorsLayer::new()
    .allow_origin(["https://app.example.com".parse::<HeaderValue>().unwrap()])
    .allow_methods([Method::GET, Method::POST])
    .allow_headers([header::CONTENT_TYPE, header::AUTHORIZATION])
    .allow_credentials(true)
```

`allow_credentials(true)` with `allow_origin(Any)` is rejected by browsers and
panics in `CorsLayer` — the spec forbids the combination.

## Custom middleware with `from_fn`

The signature is `async fn(Request, Next) -> Response`. Call `next.run(req)` to
continue; return anything else to short-circuit.

```rust
use axum::{extract::Request, middleware::Next, response::Response};
use std::time::Instant;

async fn timing(req: Request, next: Next) -> Response {
    let method = req.method().clone();
    let uri = req.uri().clone();
    let start = Instant::now();

    let mut res = next.run(req).await;

    let ms = start.elapsed().as_millis();
    println!("[timing] {method} {uri} -> {} in {ms}ms", res.status());
    res.headers_mut().insert(
        "x-response-time-ms",
        HeaderValue::from_str(&ms.to_string()).unwrap(),
    );
    res
}
```

Server output:

```
[timing] GET /hello -> 200 OK in 0ms
[timing] GET /slow -> 408 Request Timeout in 101ms
```

Clone anything you need from `req` **before** `next.run(req)`, which takes
ownership. That one rule accounts for most of the borrow errors beginners hit
here.

Extractors work in `from_fn` middleware too, before the `Request` argument:

```rust
async fn with_extractors(
    headers: HeaderMap,
    Path(id): Path<u64>,
    req: Request,
    next: Next,
) -> Response { /* ... */ }
```

Same rule as handlers: `Request` is last.

## Short-circuiting: an auth guard

```rust
async fn require_key(State(st): State<AppState>, req: Request, next: Next) -> Response {
    match req.headers().get("x-api-key").and_then(|v| v.to_str().ok()) {
        Some(k) if k == st.api_key => next.run(req).await,
        Some(_) => (StatusCode::FORBIDDEN, Json(json!({"error":"bad key"}))).into_response(),
        None => (StatusCode::UNAUTHORIZED, Json(json!({"error":"no key"}))).into_response(),
    }
}
```

Wired with `from_fn_with_state`, since it reads `State`:

```rust
let admin_routes = Router::new()
    .route("/admin", get(admin))
    .route_layer(middleware::from_fn_with_state(state.clone(), require_key));
```

```console
$ curl -si localhost:52262/admin | sed -n '1p;$p'
HTTP/1.1 401 Unauthorized
{"error":"no key"}

$ curl -si -H 'x-api-key: wrong' localhost:52262/admin | sed -n '1p;$p'
HTTP/1.1 403 Forbidden
{"error":"bad key"}

$ curl -si -H 'x-api-key: secret123' localhost:52262/admin | sed -n '1p;$p'
HTTP/1.1 200 OK
admin area
```

Use `from_fn_with_state` when the middleware needs state, `from_fn` when it
doesn't. A middleware can also **insert** data for downstream handlers:

```rust
async fn attach_user(mut req: Request, next: Next) -> Response {
    req.extensions_mut().insert(CurrentUser { id: 7 });
    next.run(req).await
}
```

Handlers then take `Extension<CurrentUser>` (lesson 17.3). This is the standard
alternative to the extractor-based auth of lesson 17.11; the extractor is
preferable because a handler that forgets `Extension<CurrentUser>` compiles
either way, but a handler without the auth extractor is visibly unprotected.

## The onion, and why order matters

Layers nest. The outermost sees the request first and the response last:

```rust
async fn order_a(req: Request, next: Next) -> Response {
    println!("[A] before");
    let res = next.run(req).await;
    println!("[A] after");
    res
}
// order_b is identical, printing [B]
```

With `ServiceBuilder`, `a` added before `b`:

```
[A] before
[B] before
[handler] running
[B] after
[A] after
```

`ServiceBuilder` reads top-to-bottom like you'd expect: the first `.layer` call
is the **outermost**.

Chained `.layer()` calls on the `Router` behave the **opposite** way:

```rust
let app = Router::new()
    .route("/x", get(handler))
    .layer(middleware::from_fn(a))
    .layer(middleware::from_fn(b));
```

```
[B] before
[A] before
[handler]
[A] after
[B] after
```

`b` is outermost because each `.layer` wraps everything added so far — the last
call wins. That is the single most confusing thing about axum middleware.
**Always use `ServiceBuilder` for more than one layer**, so the written order
matches the execution order.

Order has real consequences:

- `TraceLayer` outermost, so it records everything including timeouts and
  rejections from inner layers.
- `CompressionLayer` outside anything that writes a body, but inside `TraceLayer`
  so logs show the uncompressed size.
- `RequestBodyLimitLayer` innermost of the request-side layers, closest to the
  handler.
- Auth before anything expensive — no point decompressing a request you will
  reject.

## Global, per-route, and `route_layer`

```rust
Router::new()
    .route("/public", get(public))
    .route("/private", get(private).layer(middleware::from_fn(auth)))  // this route only
    .route_layer(middleware::from_fn(metrics))   // all routes above, matched only
    .layer(middleware::from_fn(timing))          // everything, including 404s
```

| Form | Applies to | Runs on a 404? |
|---|---|---|
| `.layer(l)` on the `Router` | every route and the fallback | yes |
| `.route_layer(l)` | only requests that matched a route | no |
| `.layer(l)` on a `MethodRouter` | that one route | n/a |

The distinction matters for auth: with `.layer`, an unauthenticated request to a
nonexistent path gets 401, leaking which paths need auth and which don't. With
`.route_layer` it gets a clean 404. Use `.route_layer` for auth and
`.layer` for tracing.

A nested router can carry its own layers, which is the cleanest way to protect a
subtree:

```rust
let admin = Router::new()
    .route("/users", get(list_users))
    .route("/users/{id}", delete(delete_user))
    .route_layer(middleware::from_fn_with_state(state.clone(), require_admin));

let app = Router::new().merge(public_routes).nest("/admin", admin);
```

## `RequestBodyLimitLayer`

```rust
.layer(RequestBodyLimitLayer::new(1024))
```

Anything larger gets `413 Payload Too Large` before your handler allocates.
Axum has a 2MB default via `DefaultBodyLimit`; this layer overrides it, and
`DefaultBodyLimit::disable()` removes the cap for a specific upload route:

```rust
.route("/upload", post(upload).layer(DefaultBodyLimit::disable()))
```

## `TraceLayer`

```rust
tracing_subscriber::fmt().init();
// ...
.layer(TraceLayer::new_for_http())
```

Gives a span per request with method, path, status, and latency. Customize what
is recorded:

```rust
TraceLayer::new_for_http()
    .make_span_with(|req: &Request<_>| {
        tracing::info_span!("http", method = %req.method(), uri = %req.uri().path())
    })
    .on_response(|res: &Response, latency: Duration, _: &tracing::Span| {
        tracing::info!(status = res.status().as_u16(), ?latency, "done");
    })
```

Use `MatchedPath` (lesson 17.3) rather than the raw URI as the span field if
these feed a metrics system — `/users/{id}` is one label, `/users/123` is
unbounded cardinality. Lesson 18.2 covers tracing properly.

## Other tower layers

From `tower` itself, usable here:

```rust
use tower::limit::ConcurrencyLimitLayer;
use tower::load_shed::LoadShedLayer;

ServiceBuilder::new()
    .layer(LoadShedLayer::new())              // 503 instead of queueing
    .layer(ConcurrencyLimitLayer::new(100))   // at most 100 in flight
```

`tower::retry` and `tower::timeout` also exist but are a better fit on the
*client* side (lesson 17.12) — retrying an inbound request is rarely what you
want.

## Quick Reference

| Need | Form |
|---|---|
| Custom middleware | `middleware::from_fn(f)` |
| ...needing state | `middleware::from_fn_with_state(state, f)` |
| Signature | `async fn f(req: Request, next: Next) -> Response` |
| Continue | `next.run(req).await` |
| Short-circuit | return a response without calling `next` |
| Pass data down | `req.extensions_mut().insert(v)` + `Extension<T>` |
| Compose many | `ServiceBuilder::new().layer(a).layer(b)` — a is outermost |
| Everything incl. 404 | `.layer(l)` on the `Router` |
| Matched routes only | `.route_layer(l)` |
| One route | `.route("/p", get(h).layer(l))` |
| Request logging | `TraceLayer::new_for_http()` |
| CORS | `CorsLayer::new().allow_origin(...)` |
| gzip | `CompressionLayer::new()` |
| Deadline | `TimeoutLayer::with_status_code(status, dur)` |
| Body cap | `RequestBodyLimitLayer::new(n)` / `DefaultBodyLimit::disable()` |
| Fixed header | `SetResponseHeaderLayer::overriding(name, value)` |
| Panic to 500 | `CatchPanicLayer::new()` |

## Common Mistakes

**Assuming chained `.layer()` runs top-to-bottom.** It is the reverse. Use
`ServiceBuilder`.

**Using `req` after `next.run(req)`.** It is moved. Clone method, uri, or headers
first.

**Forgetting the tower-http feature flag.** `CorsLayer` not found means the
`cors` feature is missing, not that the name changed.

**Putting auth in `.layer` rather than `.route_layer`.** Unauthenticated 404s
turn into 401s, leaking your route table.

**`allow_credentials(true)` with `allow_origin(Any)`.** Browsers reject it and
`CorsLayer` panics. Name the origins.

**Assuming a timed-out handler finished.** Its future is dropped mid-flight. Use
a transaction for multi-step writes.

**Reaching for `impl Service` first.** `from_fn` handles nearly everything;
hand-written services are for backpressure and cross-request state.

## Exercises

1. Add `TraceLayer`, `CorsLayer`, and a `from_fn` middleware that stamps
   `x-request-id` with a `Uuid::new_v4()` on every response. Verify with
   `curl -i`.
2. Write two logging middlewares and register them with chained `.layer()`
   calls, then with `ServiceBuilder`. Record the print order both ways.
3. Write an auth middleware that short-circuits with 401 and attach it with
   `.route_layer`. Confirm an unknown path still returns 404 rather than 401.
4. Add a middleware that inserts `Extension<RequestStart>(Instant::now())` and a
   handler that reports its own elapsed time from that extension.
5. Set `TimeoutLayer` to 50ms and `RequestBodyLimitLayer` to 10 bytes. Trigger
   both (a sleeping handler and `curl -d` with a long string) and record the two
   status codes.

---

[← Course Index](../README.md)
