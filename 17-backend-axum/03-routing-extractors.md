[← Course Index](../README.md)

# 17.3 — Routing, Path, and Query Extractors

## What You'll Learn

- Axum 0.8 route syntax: `{param}`, `{*wildcard}`, `.nest`, `.merge`, fallbacks
- `Path<T>` for one param, tuples, structs, and `HashMap`
- `Query<T>` with serde, plus the `Vec<T>` trap and what a rejection looks like
- Extractor ordering, optional extractors, and a custom `FromRequestParts` extractor

## Coming From Another Language

| Other | axum |
|---|---|
| Express `req.params.id` | `Path(id): Path<u32>` — parsed and typed |
| Express `req.query.page` | `Query(p): Query<Pagination>` |
| FastAPI `def f(id: int, page: int = 1)` | same idea, via two extractors |
| Go `mux.Vars(r)["id"]` then `strconv.Atoi` | one `Path<u32>` does both |

Extractors are just function parameters whose types implement `FromRequestParts`
(headers, path, query — cheap, repeatable) or `FromRequest` (the body —
consuming, one per handler). A failed extraction never reaches your code: axum
returns the extractor's rejection response directly.

## Route syntax

```rust
Router::new()
    .route("/users", get(list))
    .route("/users/{id}", get(show))
    .route("/orgs/{org}/repos/{repo}", get(repo))
    .route("/files/{*rest}", get(serve_file))
    .route("/multi", get(a).post(b).delete(c));
```

Axum 0.8 uses `{name}` braces. The `:name` colon syntax from 0.7 is gone — a
route string with `:` panics at startup. `{*name}` is a greedy wildcard that
captures the rest of the path, slashes included:

```console
$ curl -s localhost:52005/files/a/b/c.txt
rest=a/b/c.txt
```

Chaining methods on one path is a single `.route` call:

```console
$ curl -s localhost:52005/multi
GET
$ curl -s -X POST localhost:52005/multi
POST
$ curl -s -X DELETE localhost:52005/multi
DELETE
```

Routes do not overlap by prefix — `/users` and `/users/{id}` are distinct
patterns, and a literal segment always wins over a param at the same position.
Registering the same method on the same path twice panics at startup rather than
silently shadowing.

## `Path<T>`: one parameter

```rust
use axum::extract::Path;

async fn one(Path(id): Path<u32>) -> String {
    format!("id={id}")
}
```

```console
$ curl -s localhost:52005/one/42
id=42
```

The destructuring `Path(id): Path<u32>` is the idiom — the pattern on the left,
the extractor type on the right. You can also take it whole and use `.0`:

```rust
async fn raw_path(path: Path<String>) -> String {
    format!("captured {}", path.0)
}
```

A type mismatch is a 400 before your handler runs:

```console
$ curl -si localhost:52005/one/abc
HTTP/1.1 400 Bad Request
content-type: text/plain; charset=utf-8
content-length: 42
date: Sun, 25 Oct 2026 16:58:57 GMT

Invalid URL: Cannot parse `abc` to a `u32`
```

## `Path<T>`: tuples, structs, and maps

```rust
async fn two(Path((a, b)): Path<(String, u32)>) -> String {
    format!("a={a} b={b}")
}

#[derive(Deserialize, Debug)]
struct PathParams { org: String, repo: String }

async fn three(Path(p): Path<PathParams>) -> String {
    format!("{:?}", p)
}

async fn any_params(Path(m): Path<HashMap<String, String>>) -> String {
    let mut v: Vec<_> = m.into_iter().collect();
    v.sort();
    format!("{:?}", v)
}
```

```console
$ curl -s localhost:52005/two/abc/7
a=abc b=7

$ curl -s localhost:52005/three/rust-lang/cargo
PathParams { org: "rust-lang", repo: "cargo" }

$ curl -s localhost:52005/map/1/2/3
[("x", "1"), ("y", "2"), ("z", "3")]
```

For a tuple, **positional order** matters and the arity must match the route's
param count exactly. For a struct, the **field names** must match the route's
param names. Prefer the struct form past two params — it survives a reorder of
the route pattern.

## `Query<T>`

```rust
use axum::extract::Query;

#[derive(Deserialize, Debug)]
struct Pagination {
    #[serde(default = "one_page")] page: u32,
    #[serde(default = "twenty")] per_page: u32,
    q: Option<String>,
}
fn one_page() -> u32 { 1 }
fn twenty() -> u32 { 20 }

async fn list(Query(p): Query<Pagination>) -> String {
    format!("{:?}", p)
}
```

```console
$ curl -s 'localhost:52005/list'
Pagination { page: 1, per_page: 20, q: None }
```

Missing params fall back to `#[serde(default)]` for plain fields and `None` for
`Option` fields. Supply them and they parse:

```console
$ curl -s 'localhost:52005/resq?page=9'
ok Pagination { page: 9, per_page: 20, q: None }
```

A parse failure is a 400 with serde's message:

```console
$ curl -si 'localhost:52005/list?page=x'
HTTP/1.1 400 Bad Request
...
Failed to deserialize query string: page: invalid digit found in string
```

## The `Query<Vec<T>>` trap

`Query` deserializes with `serde_urlencoded`, which has **no sequence support**.
A `Vec<String>` field looks reasonable and fails at runtime:

```rust
#[derive(Deserialize, Debug)]
struct Pagination { #[serde(default)] tags: Vec<String>, /* ... */ }
```

```console
$ curl -s 'localhost:52005/list?tags=a&tags=b'
Failed to deserialize query string: tags: invalid type: string "a", expected a sequence
```

Three workarounds: take a single `String` and split on commas
(`?tags=a,b`); use `axum_extra::extract::Query`, which does support repeated
keys; or take `Query<HashMap<String, String>>` and parse by hand.

## Raw query as a map

```rust
async fn raw_query(Query(m): Query<HashMap<String, String>>) -> String {
    let mut v: Vec<_> = m.into_iter().collect();
    v.sort();
    format!("{:?}", v)
}
```

```console
$ curl -s 'localhost:52005/rawq?a=1&b=two&flag='
[("a", "1"), ("b", "two"), ("flag", "")]
```

An empty value is an empty string, not absent.

## `HeaderMap`

```rust
use axum::http::{HeaderMap, header};

async fn headers(h: HeaderMap) -> String {
    let ua = h.get(header::USER_AGENT).and_then(|v| v.to_str().ok()).unwrap_or("none");
    let custom = h.get("x-request-id").and_then(|v| v.to_str().ok()).unwrap_or("none");
    format!("ua={ua} x-request-id={custom} count={}", h.len())
}
```

```console
$ curl -s -H 'x-request-id: abc-123' localhost:52005/headers
ua=curl/8.7.1 x-request-id=abc-123 count=4
```

Header names are case-insensitive, lookups are by `&str` or a `header::*`
constant, and values are bytes — `to_str()` returns a `Result` because a header
value need not be UTF-8.

## The body as `String` or `Bytes`

```rust
use axum::body::Bytes;

async fn body_string(body: String) -> String { format!("{} chars: {body}", body.len()) }
async fn body_bytes(body: Bytes) -> String {
    format!("{} bytes: {:?}", body.len(), &body[..body.len().min(8)])
}
```

```console
$ curl -s -X POST localhost:52005/body -d 'hello'
5 chars: hello

$ curl -s -X POST localhost:52005/bytes --data-binary $'\x01\x02\x03hi'
5 bytes: [1, 2, 3, 104, 105]
```

`String` rejects a non-UTF-8 body with 400; `Bytes` takes anything.

## `Uri`, `Request`, and `MatchedPath`

```rust
use axum::{extract::{MatchedPath, Request}, http::Uri};

async fn the_uri(uri: Uri) -> String {
    format!("path={} query={:?}", uri.path(), uri.query())
}

async fn whole_request(req: Request) -> String {
    format!("{} {} version={:?}", req.method(), req.uri(), req.version())
}

async fn which(path: MatchedPath, Path(id): Path<u32>) -> String {
    format!("matched={} id={}", path.as_str(), id)
}
```

```console
$ curl -s 'localhost:52005/uri?a=1&b=2'
path=/uri query=Some("a=1&b=2")

$ curl -s 'localhost:52005/req?x=1'
GET /req?x=1 version=HTTP/1.1

$ curl -s localhost:52005/which/77
matched=/which/{id} id=77
```

`MatchedPath` gives the route **pattern**, not the concrete path — exactly what
you want as a metrics label, since `/users/{id}` is one cardinality bucket while
`/users/123` is millions.

## `Extension`

Request-scoped values that middleware inserts, read out by type.

```rust
use axum::Extension;

#[derive(Clone, Debug)]
struct BuildInfo { version: &'static str }

async fn protected(Extension(info): Extension<BuildInfo>) -> impl IntoResponse {
    Json(json!({ "version": info.version }))
}

let app = Router::new()
    .route("/protected", get(protected))
    .layer(Extension(BuildInfo { version: "1.4.2" }));
```

Extensions are a type-keyed map, so they are checked at **runtime**: ask for a
type nothing inserted and you get a 500. That is why lesson 17.5 prefers
`State` for application-wide dependencies and keeps `Extension` for per-request
data that middleware computes (the authenticated user, a request id).

## Extractor ordering

`FromRequestParts` extractors run in argument order and can appear any number of
times. The single `FromRequest` extractor — the one that consumes the body
(`String`, `Bytes`, `Json<T>`, `Form<T>`, `Request`) — **must be last**.

```rust
#[axum::debug_handler]
async fn wrong(body: String, headers: HeaderMap) -> String {
    format!("{} {}", body, headers.len())
}
```

Intentionally broken. Without the attribute you get an opaque `Handler<_, _> is
not satisfied`. With `#[axum::debug_handler]` (needs `cargo add axum --features
macros`):

```
error: `String` consumes the request body and thus must be the last argument to the handler function
 --> src/main.rs:4:22
  |
4 | async fn wrong(body: String, headers: HeaderMap) -> String {
  |                      ^^^^^^
```

Put `#[axum::debug_handler]` on any handler whose trait error you cannot read.
It costs nothing in release builds and is the single most useful axum debugging
tool.

## Optional and fallible extractors

`Option<T>` works for extractors implementing `OptionalFromRequestParts` —
`Path`, `Extension`, `MatchedPath`:

```rust
async fn opt_path(p: Option<Path<u32>>) -> String {
    match p {
        Some(Path(id)) => format!("got id={id}"),
        None => "no path param here".into(),
    }
}
```

Registered on both `/optp` and `/optp/{id}`:

```console
$ curl -s localhost:52005/optp
no path param here

$ curl -s localhost:52005/optp/5
got id=5
```

`Query` does **not** implement it, so `Option<Query<T>>` fails to compile. For
anything, take a `Result` with the extractor's rejection type and handle the
error yourself:

```rust
use axum::extract::rejection::QueryRejection;

async fn res_query(q: Result<Query<Pagination>, QueryRejection>) -> String {
    match q {
        Ok(Query(p)) => format!("ok {:?}", p),
        Err(e) => format!("rejected: {e}"),
    }
}
```

```console
$ curl -s 'localhost:52005/resq?page=x'
rejected: Failed to deserialize query string: page: invalid digit found in string
```

Every extractor has a matching rejection in `axum::extract::rejection`:
`PathRejection`, `QueryRejection`, `JsonRejection`, `BytesRejection`.

## Nesting and merging

```rust
let api = Router::new()
    .route("/users", get(nested_users))
    .route("/users/{id}", get(nested_user));

let app = Router::new().nest("/api/v1", api);
```

```console
$ curl -s localhost:52005/api/v1/users
nested users list

$ curl -s localhost:52005/api/v1/users/9
nested user 9
```

`.nest(prefix, router)` mounts a router under a prefix; the inner router's paths
are relative and it never sees the prefix. `.merge(router)` combines two routers
at the same level with no prefix — use it to split a flat API across modules.
`.nest_service` mounts a plain tower `Service`, such as `ServeDir` for static
files.

## Fallbacks

```rust
async fn not_found(uri: Uri) -> impl IntoResponse {
    (StatusCode::NOT_FOUND, Json(json!({ "error": "no route", "path": uri.path() })))
}

async fn method_fallback() -> impl IntoResponse {
    (StatusCode::METHOD_NOT_ALLOWED, Json(json!({ "error": "method not allowed" })))
}

let app = Router::new()
    // ... routes ...
    .method_not_allowed_fallback(method_fallback)
    .fallback(not_found);
```

```console
$ curl -si localhost:52005/nope | tail -1
{"error":"no route","path":"/nope"}

$ curl -si -X PATCH localhost:52005/multi | tail -1
{"error":"method not allowed"}
```

`.fallback` handles "no route matched this path". `.method_not_allowed_fallback`
handles "path matched, method didn't", replacing the empty 405 from lesson 17.2
while keeping the `allow` header. A nested router can carry its own `.fallback`,
which wins for paths under its prefix — handy for serving a JSON 404 under
`/api` and an HTML one everywhere else.

## A custom extractor with `FromRequestParts`

The real pattern: validate something about the request once, then take it as a
typed argument in every handler that needs it.

```rust
use axum::{extract::FromRequestParts, http::request::Parts};

struct ApiKey(String);

impl<S: Send + Sync> FromRequestParts<S> for ApiKey {
    type Rejection = (StatusCode, Json<serde_json::Value>);

    async fn from_request_parts(parts: &mut Parts, _state: &S) -> Result<Self, Self::Rejection> {
        let key = parts
            .headers
            .get("x-api-key")
            .and_then(|v| v.to_str().ok())
            .ok_or_else(|| (StatusCode::UNAUTHORIZED, Json(json!({"error": "missing x-api-key"}))))?;
        if key.len() < 8 {
            return Err((StatusCode::FORBIDDEN, Json(json!({"error": "key too short"}))));
        }
        Ok(ApiKey(key.to_owned()))
    }
}

async fn protected(ApiKey(key): ApiKey, Extension(info): Extension<BuildInfo>) -> impl IntoResponse {
    Json(json!({ "key": key, "version": info.version }))
}
```

```console
$ curl -si localhost:52082/protected | tail -1
{"error":"missing x-api-key"}

$ curl -si -H 'x-api-key: abc' localhost:52082/protected | tail -1
{"error":"key too short"}

$ curl -si -H 'x-api-key: supersecret' localhost:52082/protected | tail -1
{"key":"supersecret","version":"1.4.2"}
```

Three things make this work:

- `Rejection` must implement `IntoResponse`; a `(StatusCode, Json<_>)` tuple does.
- The `S` generic is the router state. `S: Send + Sync` keeps the extractor
  usable with any state; constrain it (or `S: FromRef<...>`) when the extractor
  needs to read state, as the JWT extractor in lesson 17.11 does.
- Implement `FromRequest` instead only when you need the **body**. `Parts` has
  headers, method, uri, and extensions — everything but the body.

Axum 0.8 declares these as plain `async fn` in the trait, no `#[async_trait]`.

## Quick Reference

| Need | Extractor / API |
|---|---|
| One path param | `Path<u32>` |
| Several, positional | `Path<(String, u32)>` |
| Several, named | `Path<MyStruct>` (field names = route names) |
| All path params | `Path<HashMap<String, String>>` |
| Query string | `Query<MyStruct>` or `Query<HashMap<String, String>>` |
| Repeated query keys | `axum_extra::extract::Query` |
| Headers | `HeaderMap` |
| Body as text / bytes | `String` / `Bytes` (last argument) |
| Whole request | `Request` (last argument) |
| URL only | `Uri` |
| Route pattern | `MatchedPath` |
| Middleware-inserted value | `Extension<T>` |
| Optional | `Option<Path<T>>`, `Option<Extension<T>>` |
| Fallible | `Result<Query<T>, QueryRejection>` |
| Mount under prefix | `.nest("/api", r)` |
| Combine flat | `.merge(r)` |
| Mount a tower service | `.nest_service("/static", svc)` |
| 404 / 405 | `.fallback(h)` / `.method_not_allowed_fallback(h)` |
| Custom (no body) | `impl FromRequestParts for T` |
| Custom (with body) | `impl FromRequest for T` |
| Readable handler errors | `#[axum::debug_handler]` |

## Common Mistakes

**`:id` instead of `{id}`.** Axum 0.8 panics at startup on the old syntax.

**Mismatching `Path` tuple arity or struct field names.** Both are runtime
rejections, not compile errors — a tuple with the wrong arity yields
`Wrong number of path arguments`.

**`Vec<T>` in a `Query` struct.** `serde_urlencoded` has no sequence support.
Split a comma string or use `axum_extra`'s `Query`.

**Putting a body extractor before another argument.** Reach for
`#[axum::debug_handler]` the moment you see `Handler<_, _> is not satisfied`.

**Using `Extension` for the database pool.** It is runtime-checked and will 500
if the layer is missing. Use `State` (lesson 17.5).

**Two body extractors in one handler.** Only one can consume the body; use
`Json<T>` and read headers separately from `HeaderMap`.

## Exercises

1. Route `GET /orgs/{org}/members/{user_id}` into a struct `Path` extractor and
   print both values. Then request it with a non-numeric `user_id` and read the
   400 body.
2. Add `GET /search` with `Query<SearchParams { q: String, limit: Option<u32> }>`.
   Make `q` required and confirm the rejection text when it is missing.
3. Build a `Router` for `/posts` and nest it under `/api/v2`. Add a nested
   `.fallback` that returns a JSON 404 only for paths under that prefix.
4. Write a `RequestId` extractor implementing `FromRequestParts` that reads
   `x-request-id`, generates a `Uuid::new_v4()` when absent, and never rejects.
   Use it in two handlers.
5. Write a handler taking `MatchedPath`, `HeaderMap`, and `String` body, in the
   wrong order. Add `#[axum::debug_handler]`, read the error, then fix it.

---

[← Course Index](../README.md)
