[← 17.3 Routing, Path, and Query Extractors](03-routing-extractors.md) · [Course Index](../README.md) · [17.5 Shared State and Dependency Injection →](05-state.md)

# 17.4 — JSON Requests and Responses

## What You'll Learn

- `Json<T>` as both extractor and response, and the content-type rules
- 201 Created with a body and a `Location` header
- Every `JsonRejection` variant with its real status and body, and how to
  replace them with your own JSON shape
- `Result<Json<T>, AppError>` handlers and DTO/domain separation

## Coming From Another Language

| Other | axum |
|---|---|
| `express.json()` middleware + `req.body` | `Json(body): Json<CreateUser>` |
| `res.status(201).json(obj)` | `(StatusCode::CREATED, Json(obj))` |
| FastAPI `def f(u: CreateUser)` | identical ergonomics, no runtime validator |
| Go `json.NewDecoder(r.Body).Decode(&v)` | the extractor does it, or rejects |

There is no body-parsing middleware to register. `Json<T>` is a
`FromRequest` extractor: it checks the content type, reads the body, and runs
`serde_json::from_slice`. If any step fails your handler never runs.

## Setup

```bash
cargo add axum tokio --features tokio/full
cargo add serde --features derive
cargo add serde_json
```

## Extracting and responding

```rust
use axum::{Json, http::{StatusCode, header}, response::IntoResponse};
use serde::{Deserialize, Serialize};

#[derive(Deserialize, Debug)]
struct CreateUser { name: String, email: String, age: Option<u8> }

#[derive(Serialize)]
struct UserResponse { id: u64, name: String, email: String }

async fn create(Json(body): Json<CreateUser>) -> impl IntoResponse {
    println!("handler got {:?}", body);
    (
        StatusCode::CREATED,
        [(header::LOCATION, "/users/101")],
        Json(UserResponse { id: 101, name: body.name, email: body.email }),
    )
}
```

```console
$ curl -si -X POST localhost:52106/users \
    -H 'content-type: application/json' \
    -d '{"name":"raj","email":"r@x.com"}'
HTTP/1.1 201 Created
content-type: application/json
location: /users/101
content-length: 41
date: Sun, 25 Oct 2026 17:01:26 GMT

{"id":101,"name":"raj","email":"r@x.com"}
```

Server-side:

```
handler got CreateUser { name: "raj", email: "r@x.com", age: None }
```

The tuple is `(status, headers, body)`. `Json` sets `content-type:
application/json`; the explicit `LOCATION` header rides alongside. Returning
`201 Created` with both a `Location` header and the created resource is the
convention worth following.

## Plain responses

```rust
async fn plain_json() -> Json<UserResponse> {
    Json(UserResponse { id: 1, name: "raj".into(), email: "r@x.com".into() })
}

async fn list() -> Json<Vec<UserResponse>> {
    Json(vec![
        UserResponse { id: 1, name: "raj".into(), email: "r@x.com".into() },
        UserResponse { id: 2, name: "sam".into(), email: "s@x.com".into() },
    ])
}
```

```console
$ curl -s localhost:52106/one
{"id":1,"name":"raj","email":"r@x.com"}

$ curl -s localhost:52106/users
[{"id":1,"name":"raj","email":"r@x.com"},{"id":2,"name":"sam","email":"s@x.com"}]
```

`Json<T>` works as a response for any `T: Serialize` — structs, `Vec<T>`,
`HashMap`, `Option<T>`, `serde_json::Value`. Serialization happens when the
response is built; if it fails (rare, but possible with a custom `Serialize`)
axum emits a 500.

## The four `JsonRejection` cases

These are the defaults you get for free, and the bodies your clients will see
until you override them. Worth memorizing the status codes.

**Missing required field** — valid JSON, wrong shape, so 422:

```console
$ curl -si -X POST localhost:52106/users -H 'content-type: application/json' -d '{"name":"raj"}'
HTTP/1.1 422 Unprocessable Entity
content-type: text/plain; charset=utf-8
content-length: 99
date: Sun, 25 Oct 2026 17:01:26 GMT

Failed to deserialize the JSON body into the target type: missing field `email` at line 1 column 14
```

**Wrong field type** — also 422:

```console
$ curl -si -X POST localhost:52106/users -H 'content-type: application/json' \
    -d '{"name":"raj","email":"r@x.com","age":"old"}'
HTTP/1.1 422 Unprocessable Entity
...
Failed to deserialize the JSON body into the target type: age: invalid type: string "old", expected u8 at line 1 column 43
```

**Malformed JSON** — not parseable at all, so 400:

```console
$ curl -si -X POST localhost:52106/users -H 'content-type: application/json' -d '{"name":'
HTTP/1.1 400 Bad Request
...
Failed to parse the request body as JSON: name: EOF while parsing a value at line 1 column 8
```

**Missing or wrong content type** — 415:

```console
$ curl -si -X POST localhost:52106/users -d '{"name":"raj","email":"r@x.com"}'
HTTP/1.1 415 Unsupported Media Type
content-type: text/plain; charset=utf-8
content-length: 54
date: Sun, 25 Oct 2026 17:01:26 GMT

Expected request with `Content-Type: application/json`
```

Note the body is `text/plain`, not JSON. A client parsing every response as JSON
will choke on it — which is the whole reason to customize these.

Content-type rules: axum accepts `application/json`, anything with a `+json`
suffix (`application/merge-patch+json`), and parameters are fine
(`application/json; charset=utf-8`). `curl -d` without a header sends
`application/x-www-form-urlencoded`, hence the 415 above.

## Customizing `JsonRejection`

Wrap `Json<T>` in your own extractor and map each rejection variant to your
error envelope. This is the single highest-value 40 lines in an axum codebase.

```rust
use axum::extract::{FromRequest, Request, rejection::JsonRejection};

struct AppJson<T>(T);

impl<S, T> FromRequest<S> for AppJson<T>
where
    Json<T>: FromRequest<S, Rejection = JsonRejection>,
    S: Send + Sync,
{
    type Rejection = (StatusCode, Json<serde_json::Value>);

    async fn from_request(req: Request, state: &S) -> Result<Self, Self::Rejection> {
        match Json::<T>::from_request(req, state).await {
            Ok(Json(v)) => Ok(AppJson(v)),
            Err(rej) => {
                let (code, kind) = match &rej {
                    JsonRejection::JsonDataError(_) => (StatusCode::UNPROCESSABLE_ENTITY, "data"),
                    JsonRejection::JsonSyntaxError(_) => (StatusCode::BAD_REQUEST, "syntax"),
                    JsonRejection::MissingJsonContentType(_) => (StatusCode::UNSUPPORTED_MEDIA_TYPE, "content_type"),
                    _ => (StatusCode::BAD_REQUEST, "other"),
                };
                Err((code, Json(json!({ "error": kind, "message": rej.body_text() }))))
            }
        }
    }
}

async fn strict(AppJson(b): AppJson<CreateUser>) -> impl IntoResponse {
    Json(json!({ "ok": true, "name": b.name, "age": b.age }))
}
```

Every case now answers with JSON:

```console
$ curl -s -X POST localhost:52106/strict -H 'content-type: application/json' -d '{"name":"raj"}'
{"error":"data","message":"Failed to deserialize the JSON body into the target type: missing field `email` at line 1 column 14"}

$ curl -s -X POST localhost:52106/strict -H 'content-type: application/json' -d '{oops'
{"error":"syntax","message":"Failed to parse the request body as JSON: key must be a string at line 1 column 2"}

$ curl -s -X POST localhost:52106/strict -d '{"name":"raj","email":"a@b.c"}'
{"error":"content_type","message":"Expected request with `Content-Type: application/json`"}

$ curl -s -X POST localhost:52106/strict -H 'content-type: application/json' \
    -d '{"name":"raj","email":"a@b.c","age":31}'
{"age":31,"name":"raj","ok":true}
```

`JsonRejection` is `#[non_exhaustive]`, hence the `_ =>` arm. `rej.body_text()`
is the string axum would have sent; `rej.status()` is the status it would have
used, if you want to keep axum's choice and only change the shape.

Usage note: the newtype destructures exactly like `Json`, so switching a handler
over is a one-word edit — `Json(b): Json<T>` becomes `AppJson(b): AppJson<T>`.

## `Result<Json<T>, E>` handlers

Any error type implementing `IntoResponse` works, which makes `?` usable in
handlers.

```rust
#[derive(Serialize)]
struct ApiError { error: &'static str, detail: String }

impl IntoResponse for ApiError {
    fn into_response(self) -> Response {
        (StatusCode::UNPROCESSABLE_ENTITY, Json(self)).into_response()
    }
}

async fn fallible(Json(b): Json<CreateUser>) -> Result<Json<UserResponse>, ApiError> {
    if !b.email.contains('@') {
        return Err(ApiError { error: "invalid_email", detail: format!("{:?} has no @", b.email) });
    }
    Ok(Json(UserResponse { id: 7, name: b.name, email: b.email }))
}
```

```console
$ curl -s -X POST localhost:52106/fallible -H 'content-type: application/json' \
    -d '{"name":"raj","email":"nope"}'
{"error":"invalid_email","detail":"\"nope\" has no @"}

$ curl -s -X POST localhost:52106/fallible -H 'content-type: application/json' \
    -d '{"name":"raj","email":"r@x.com"}'
{"id":7,"name":"raj","email":"r@x.com"}
```

The error type being `Serialize` **and** `IntoResponse` is the pattern: one
struct defines both the status code and the JSON body. Lesson 17.6 turns this
into a full `AppError` enum with one variant per failure mode.

## Nested payloads

Nothing special — serde handles arbitrary depth, and the extractor rejects the
whole payload if any level fails.

```rust
#[derive(Deserialize, Serialize, Debug)]
struct Order { id: u32, customer: Customer, lines: Vec<Line>, #[serde(default)] note: Option<String> }
#[derive(Deserialize, Serialize, Debug)]
struct Customer { name: String, address: Address }
#[derive(Deserialize, Serialize, Debug)]
struct Address { city: String, zip: String }
#[derive(Deserialize, Serialize, Debug)]
struct Line { sku: String, qty: u32, unit_cents: i64 }

async fn complex(Json(o): Json<Order>) -> impl IntoResponse {
    let total: i64 = o.lines.iter().map(|l| l.qty as i64 * l.unit_cents).sum();
    (StatusCode::OK, Json(json!({
        "order_id": o.id,
        "city": o.customer.address.city,
        "total_cents": total,
    })))
}
```

```console
$ curl -s -X POST localhost:52106/orders -H 'content-type: application/json' -d '{
  "id":9,
  "customer":{"name":"Raj","address":{"city":"Pune","zip":"411001"}},
  "lines":[{"sku":"A1","qty":2,"unit_cents":1500},{"sku":"B2","qty":1,"unit_cents":999}]
}'
{"city":"Pune","order_id":9,"total_cents":3999}
```

## Pass-through with `Value`

For webhooks and proxies where you don't own the schema:

```rust
async fn passthrough(Json(v): Json<serde_json::Value>) -> Json<serde_json::Value> {
    Json(json!({ "received": v, "keys": v.as_object().map(|m| m.len()) }))
}
```

```console
$ curl -s -X POST localhost:52106/echo -H 'content-type: application/json' -d '{"a":1,"b":[true,null]}'
{"keys":2,"received":{"a":1,"b":[true,null]}}
```

`Json<Value>` accepts any valid JSON, so the only rejections left are syntax and
content type. Convenient and dangerous — you have given up every type guarantee,
so use it only where you genuinely re-emit whatever you were handed.

## DTO vs domain model

Do not derive `Serialize` on your database row type and return it. Keep three
shapes:

```rust
// Domain / database row — owns the fields that exist in storage.
#[derive(sqlx::FromRow)]
struct User {
    id: i64,
    email: String,
    password_hash: String,
    created_at: DateTime<Utc>,
}

// Request DTO — only what a client is allowed to send.
#[derive(Deserialize)]
struct CreateUser { email: String, password: String }

// Response DTO — only what a client is allowed to see.
#[derive(Serialize)]
struct UserDto { id: i64, email: String, created_at: DateTime<Utc> }

impl From<User> for UserDto {
    fn from(u: User) -> Self {
        Self { id: u.id, email: u.email, created_at: u.created_at }
    }
}
```

Three concrete reasons, in order of how much they will hurt you:

1. **`password_hash` cannot leak.** It is not on `UserDto`, so no `#[serde(skip)]`
   can be forgotten.
2. **Clients cannot set `id` or `created_at`.** They are not on `CreateUser`, so
   mass-assignment is impossible by construction.
3. **A column rename does not break your API.** The `From` impl is the one place
   that changes, and the compiler points at it.

The `From` impl means handlers read `Ok(Json(user.into()))`. A single
`#[serde(rename_all = "camelCase")]` on the response DTO also gives frontend
clients the casing they expect without touching the domain type.

## Quick Reference

| Need | Form |
|---|---|
| Parse body | `Json(b): Json<T>` where `T: Deserialize` |
| Respond | `Json(v)` where `V: Serialize` |
| 201 + body | `(StatusCode::CREATED, Json(v))` |
| 201 + header + body | `(StatusCode::CREATED, [(header::LOCATION, "/x")], Json(v))` |
| Fallible | `-> Result<Json<T>, E>` where `E: IntoResponse` |
| Any JSON | `Json<serde_json::Value>` |
| Rejection kinds | `JsonDataError` 422, `JsonSyntaxError` 400, `MissingJsonContentType` 415 |
| Rejection text / status | `rej.body_text()` / `rej.status()` |
| Custom rejection | newtype + `impl FromRequest` delegating to `Json<T>` |
| Accepted content types | `application/json`, `*+json`, with parameters |

## Common Mistakes

**Forgetting `content-type: application/json` in curl.** You get a 415, not a
parse error. `curl -d` defaults to form encoding.

**Expecting 400 for a missing field.** Valid-JSON-wrong-shape is 422; only
unparseable JSON is 400. Clients that treat 422 as a server bug will be confused.

**Letting the default rejection body reach clients.** It is `text/plain` while
every other response is JSON. Wrap `Json<T>` as shown.

**Returning the database row type.** Sooner or later it grows a
`password_hash`, a `deleted_at`, or an internal note. Use a response DTO.

**Putting `Json<T>` before another extractor.** It consumes the body, so it must
be the handler's last argument (lesson 17.3).

**Matching `JsonRejection` without a `_` arm.** It is `#[non_exhaustive]`; the
compiler will insist.

## Exercises

1. `POST /posts` taking `{ "title": String, "body": String, "tags":
   Vec<String> }` and returning 201 with `{ "id": 1, ... }` plus a
   `Location: /posts/1` header. Verify with `curl -i`.
2. Send that endpoint four bad requests — missing `title`, `tags` as a number,
   truncated JSON, and no content type — and record the four status codes.
3. Write an `AppJson<T>` extractor that returns
   `{"error":{"code":"...","message":"..."}}` for every rejection variant, and
   switch the handler from `Json` to `AppJson` with a one-word edit.
4. Add `GET /posts/{id}` returning `Result<Json<Post>, ApiError>` that 404s for
   `id > 100` with a JSON body, and confirm both paths with curl.
5. Define `PostRow` with a `internal_notes: String` field, a `PostDto` without
   it, and a `From<PostRow> for PostDto`. Return `Json(row.into())` and prove the
   notes never appear in the response.

---

[← 17.3 Routing, Path, and Query Extractors](03-routing-extractors.md) · [Course Index](../README.md) · [17.5 Shared State and Dependency Injection →](05-state.md)
