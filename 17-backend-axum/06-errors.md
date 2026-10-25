[← 17.5 Shared State and Dependency Injection](05-state.md) · [Course Index](../README.md) · [17.7 Middleware and tower Layers →](07-middleware.md)

# 17.6 — Error Handling in Axum

## What You'll Learn

- The core pattern: one `AppError` enum + `impl IntoResponse`
- `thiserror` for the enum, `#[from]` so `?` just works in handlers
- Hiding internal details from clients while logging them in full
- A consistent error envelope, a JSON 404 fallback, and `anyhow` at the edges

## Coming From Another Language

| Other | Rust / axum |
|---|---|
| `throw new NotFoundError()` + an error middleware | return `Err(AppError::NotFound(..))`; `IntoResponse` is the middleware |
| FastAPI `raise HTTPException(404, ...)` | one enum variant per failure mode |
| Go `if err != nil { return err }` everywhere | `?` with a `From` impl doing the conversion |

There is no error middleware and no exception unwinding. A handler returns
`Result<T, AppError>`; axum calls `AppError::into_response()`. Everything —
status, body, logging — lives in that one impl, so every endpoint is consistent
by construction.

## Setup

```bash
cargo add thiserror anyhow
cargo add tracing tracing-subscriber
```

## The error enum

`thiserror` generates `Display` from the attribute strings and `From` impls from
`#[from]`. That is all the boilerplate there is.

```rust
use thiserror::Error;
use axum::extract::rejection::JsonRejection;

#[derive(Debug, Error)]
enum AppError {
    #[error("{0} {1} not found")]
    NotFound(&'static str, String),
    #[error("validation failed")]
    Validation(Vec<FieldError>),
    #[error("{0}")]
    BadRequest(String),
    #[error("not authenticated")]
    Unauthorized,
    #[error("insufficient permissions")]
    Forbidden,
    #[error("{0} already exists")]
    Conflict(String),
    #[error("database error")]
    Database(#[from] sqlx::Error),
    #[error("bad json")]
    Json(#[from] JsonRejection),
    #[error(transparent)]
    Internal(#[from] anyhow::Error),
}

#[derive(Debug, Serialize)]
struct FieldError { field: &'static str, message: String }
```

One variant per **HTTP outcome**, not per internal cause. `Database` and
`Internal` both become 500s and both hide their detail; the client-facing
variants carry only information the client is allowed to see.

`#[error(transparent)]` on `Internal` forwards `Display` to the inner
`anyhow::Error` so logs keep the full chain.

## The response envelope

One shape for every error in the API. Clients can write one error handler.

```rust
#[derive(Serialize)]
struct ErrorBody<'a> { error: ErrorDetail<'a> }

#[derive(Serialize)]
struct ErrorDetail<'a> {
    code: &'a str,
    message: String,
    #[serde(skip_serializing_if = "Option::is_none")]
    fields: Option<Vec<FieldError>>,
}
```

`code` is the stable machine-readable string clients branch on; `message` is for
humans and may change; `fields` appears only for validation errors.

## Status and code per variant

```rust
impl AppError {
    fn code(&self) -> &'static str {
        match self {
            AppError::NotFound(..) => "not_found",
            AppError::Validation(_) => "validation_error",
            AppError::BadRequest(_) => "bad_request",
            AppError::Unauthorized => "unauthorized",
            AppError::Forbidden => "forbidden",
            AppError::Conflict(_) => "conflict",
            AppError::Database(_) | AppError::Internal(_) => "internal_error",
            AppError::Json(_) => "invalid_json",
        }
    }

    fn status(&self) -> StatusCode {
        match self {
            AppError::NotFound(..) => StatusCode::NOT_FOUND,
            AppError::Validation(_) => StatusCode::UNPROCESSABLE_ENTITY,
            AppError::BadRequest(_) => StatusCode::BAD_REQUEST,
            AppError::Unauthorized => StatusCode::UNAUTHORIZED,
            AppError::Forbidden => StatusCode::FORBIDDEN,
            AppError::Conflict(_) => StatusCode::CONFLICT,
            AppError::Json(r) => r.status(),
            AppError::Database(_) | AppError::Internal(_) => StatusCode::INTERNAL_SERVER_ERROR,
        }
    }
}
```

Exhaustive `match`es are the point: add a variant and the compiler lists both
places to update. `AppError::Json(r) => r.status()` reuses axum's own 400/415/422
choice rather than guessing.

## `impl IntoResponse` — log, then redact

```rust
impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        let status = self.status();
        let code = self.code();

        if status.is_server_error() {
            tracing::error!(error = ?self, "request failed");
        } else {
            tracing::warn!(error = %self, "request rejected");
        }

        let (message, fields) = match &self {
            AppError::Database(_) | AppError::Internal(_) =>
                ("something went wrong".to_string(), None),
            AppError::Validation(f) => (
                "validation failed".to_string(),
                Some(f.iter().map(|e| FieldError { field: e.field, message: e.message.clone() }).collect()),
            ),
            AppError::Json(r) => (r.body_text(), None),
            other => (other.to_string(), None),
        };

        (status, Json(ErrorBody { error: ErrorDetail { code, message, fields } })).into_response()
    }
}
```

Three decisions worth copying:

- **Log before redacting.** The full error goes to `tracing`, the sanitized
  version to the client.
- `is_server_error()` picks the level: 5xx is `error!` (page someone), 4xx is
  `warn!` (the client's problem).
- `?self` uses `Debug` for the structured log; `%self` uses `Display`.

## `?` in handlers

With the `#[from]` impls in place, `?` converts automatically.

```rust
async fn lookup(id: u64) -> Result<User, AppError> {
    match id {
        1 => Ok(User { id: 1, email: "raj@x.com".into() }),
        13 => Err(sqlx::Error::RowNotFound.into()),
        14 => Err(anyhow::anyhow!("redis connection refused on 127.0.0.1:6379").into()),
        _ => Err(AppError::NotFound("user", id.to_string())),
    }
}

async fn show(Path(id): Path<u64>) -> Result<Json<User>, AppError> {
    let u = lookup(id).await?;
    Ok(Json(u))
}
```

Happy path:

```console
$ curl -si localhost:52222/users/1 | sed -n '1p;$p'
HTTP/1.1 200 OK
{"id":1,"email":"raj@x.com"}
```

An explicit domain not-found:

```console
$ curl -si localhost:52222/users/7 | sed -n '1p;$p'
HTTP/1.1 404 Not Found
{"error":{"code":"not_found","message":"user 7 not found"}}
```

A `sqlx::Error` that reached the handler untranslated — note it becomes a 500,
not a 404:

```console
$ curl -si localhost:52222/users/13 | sed -n '1p;$p'
HTTP/1.1 500 Internal Server Error
{"error":{"code":"internal_error","message":"something went wrong"}}
```

An `anyhow::Error` from some internal subsystem:

```console
$ curl -si localhost:52222/users/14 | sed -n '1p;$p'
HTTP/1.1 500 Internal Server Error
{"error":{"code":"internal_error","message":"something went wrong"}}
```

## Mapping `sqlx::Error` properly

`#[from] sqlx::Error` makes everything a 500, which is wrong for two common
cases. Translate those at the call site, where you know what the query meant:

```rust
async fn find_user(pool: &PgPool, id: i64) -> Result<User, AppError> {
    sqlx::query_as::<_, User>("SELECT id, email FROM users WHERE id = $1")
        .bind(id)
        .fetch_one(pool)
        .await
        .map_err(|e| match e {
            sqlx::Error::RowNotFound => AppError::NotFound("user", id.to_string()),
            other => AppError::Database(other),
        })
}
```

Or use `fetch_optional` and `ok_or_else`, which reads better:

```rust
sqlx::query_as::<_, User>("SELECT id, email FROM users WHERE id = $1")
    .bind(id)
    .fetch_optional(pool)
    .await?
    .ok_or_else(|| AppError::NotFound("user", id.to_string()))
```

A unique-constraint violation should be a 409, not a 500:

```rust
.map_err(|e| match &e {
    sqlx::Error::Database(db) if db.code().as_deref() == Some("23505") =>
        AppError::Conflict(format!("email {email}")),
    _ => AppError::Database(e),
})
```

`23505` is the Postgres `unique_violation` SQLSTATE code. For SQLite the check is
`db.code().as_deref() == Some("2067")`.

## Validation errors with field detail

```rust
async fn create(body: Result<Json<NewUser>, JsonRejection>) -> Result<Json<User>, AppError> {
    let Json(b) = body?;
    let mut errs = Vec::new();
    if !b.email.contains('@') {
        errs.push(FieldError { field: "email", message: "must contain @".into() });
    }
    if b.age < 18 {
        errs.push(FieldError { field: "age", message: format!("must be >= 18, got {}", b.age) });
    }
    if !errs.is_empty() {
        return Err(AppError::Validation(errs));
    }
    if b.email == "taken@x.com" {
        return Err(AppError::Conflict(format!("email {}", b.email)));
    }
    Ok(Json(User { id: 2, email: b.email }))
}
```

```console
$ curl -si -X POST localhost:52222/users -H 'content-type: application/json' \
    -d '{"email":"nope","age":12}' | sed -n '1p;$p'
HTTP/1.1 422 Unprocessable Entity
{"error":{"code":"validation_error","message":"validation failed","fields":[{"field":"email","message":"must contain @"},{"field":"age","message":"must be >= 18, got 12"}]}}
```

Collect **all** the field errors before returning. A form that reveals one
problem at a time is miserable to use. Lesson 17.10 replaces the hand-rolled
checks with the `validator` crate while keeping this response shape.

```console
$ curl -si -X POST localhost:52222/users -H 'content-type: application/json' \
    -d '{"email":"taken@x.com","age":30}' | sed -n '1p;$p'
HTTP/1.1 409 Conflict
{"error":{"code":"conflict","message":"email taken@x.com already exists"}}
```

## Folding extractor rejections into `AppError`

Taking `Result<Json<T>, JsonRejection>` instead of `Json<T>` routes axum's
rejections through your envelope, via `?` and the `#[from] JsonRejection`:

```console
$ curl -si -X POST localhost:52222/users -H 'content-type: application/json' \
    -d '{"email":"a@b.c"}' | sed -n '1p;$p'
HTTP/1.1 422 Unprocessable Entity
{"error":{"code":"invalid_json","message":"Failed to deserialize the JSON body into the target type: missing field `age` at line 1 column 17"}}

$ curl -si -X POST localhost:52222/users -H 'content-type: application/json' -d '{bad' | sed -n '1p;$p'
HTTP/1.1 400 Bad Request
{"error":{"code":"invalid_json","message":"Failed to parse the request body as JSON: key must be a string at line 1 column 2"}}

$ curl -si -X POST localhost:52222/users -d '{"email":"a@b.c","age":30}' | sed -n '1p;$p'
HTTP/1.1 415 Unsupported Media Type
{"error":{"code":"invalid_json","message":"Expected request with `Content-Type: application/json`"}}
```

Three different statuses, one envelope, no `text/plain` leaking out. The
`AppJson<T>` extractor from lesson 17.4 is the alternative — it keeps the
handler signature clean at the cost of a newtype.

## The remaining statuses

```rust
async fn forbidden() -> Result<&'static str, AppError> { Err(AppError::Forbidden) }
async fn unauth() -> Result<&'static str, AppError> { Err(AppError::Unauthorized) }
async fn boom() -> Result<&'static str, AppError> {
    Err(AppError::Internal(anyhow::anyhow!("secret internal detail: db password rotated")))
}
```

```console
$ curl -si localhost:52222/unauth | sed -n '1p;$p'
HTTP/1.1 401 Unauthorized
{"error":{"code":"unauthorized","message":"not authenticated"}}

$ curl -si localhost:52222/forbidden | sed -n '1p;$p'
HTTP/1.1 403 Forbidden
{"error":{"code":"forbidden","message":"insufficient permissions"}}

$ curl -si localhost:52222/boom | sed -n '1p;$p'
HTTP/1.1 500 Internal Server Error
{"error":{"code":"internal_error","message":"something went wrong"}}
```

401 means "I don't know who you are"; 403 means "I know, and no". Getting these
backwards breaks client retry logic.

## A JSON 404 fallback

A handler can return `AppError` directly, since `AppError: IntoResponse`:

```rust
async fn fallback(uri: Uri) -> AppError {
    AppError::NotFound("route", uri.path().to_string())
}

let app = Router::new()
    // ... routes ...
    .fallback(fallback);
```

```console
$ curl -si localhost:52222/nope/path | sed -n '1p;$p'
HTTP/1.1 404 Not Found
{"error":{"code":"not_found","message":"route /nope/path not found"}}
```

Now *every* response from the service, including unrouted paths, has the same
shape.

## What the server logged

The whole point of the pattern — the client saw `"something went wrong"`, the
operator sees everything:

```
 WARN request rejected error=user 7 not found
ERROR request failed error=Database(RowNotFound)
ERROR request failed error=Internal(redis connection refused on 127.0.0.1:6379)
 WARN request rejected error=validation failed
 WARN request rejected error=email taken@x.com already exists
 WARN request rejected error=bad json
 WARN request rejected error=bad json
 WARN request rejected error=bad json
 WARN request rejected error=not authenticated
 WARN request rejected error=insufficient permissions
ERROR request failed error=Internal(secret internal detail: db password rotated)
 WARN request rejected error=route /nope/path not found
```

Compare the last `ERROR` line with the 500 body above: the database password
detail never left the process. Lesson 18.2 adds request ids to these lines so a
user reporting "it said something went wrong" can be matched to a log entry —
include the id in the response body for that.

## `anyhow` at the edges vs typed errors

| | typed (`thiserror`) | `anyhow` |
|---|---|---|
| Where | library code, domain logic, anything a caller branches on | leaf operations nobody inspects |
| Gives you | exhaustive `match`, compiler-enforced coverage | `?` across any error type, a context chain |
| In axum | `AppError` — the handler's error type | the payload of `AppError::Internal` |

The hybrid above is the practical answer: typed variants for every outcome the
API distinguishes, plus one `Internal(#[from] anyhow::Error)` catch-all so
`?` works on arbitrary errors from file IO, parsing, or third-party crates. Add
context as you go:

```rust
use anyhow::Context;

let cfg = std::fs::read_to_string(path)
    .with_context(|| format!("reading config at {path}"))?;
```

The context chain shows up in the `ERROR` log line and nowhere else.

Do **not** put `anyhow::Error` directly as a handler's error type and call it a
day: it cannot distinguish a 404 from a 500, so every failure becomes a 500.

## Panics are not errors

A panic in a handler aborts that task; by default the connection drops and the
client sees no response. Add `tower_http::catch_panic::CatchPanicLayer` to turn
one into a 500:

```rust
use tower_http::catch_panic::CatchPanicLayer;

let app = Router::new()/* ... */.layer(CatchPanicLayer::new());
```

It is a safety net, not a strategy — a panic still means a bug. Lesson 17.7
covers layers in general.

## Quick Reference

| Need | Form |
|---|---|
| Define | `#[derive(Debug, Error)] enum AppError { ... }` |
| Message | `#[error("...{0}...")]` |
| Auto-convert | `#[error("...")] Variant(#[from] OtherError)` |
| Forward `Display` | `#[error(transparent)]` |
| Make it a response | `impl IntoResponse for AppError` |
| Use in handler | `-> Result<Json<T>, AppError>` then `?` |
| Reuse axum's status | `rejection.status()` / `rejection.body_text()` |
| Row-not-found as 404 | `fetch_optional().await?.ok_or_else(...)` |
| Unique violation as 409 | match `db.code() == Some("23505")` |
| Hide internals | redact `message` for 5xx, log the full error |
| JSON 404 | `.fallback(h)` returning `AppError` |
| Catch panics | `.layer(CatchPanicLayer::new())` |

## Common Mistakes

**Letting `#[from] sqlx::Error` turn a missing row into a 500.** Use
`fetch_optional` and map to `NotFound` where the query is written.

**Returning `e.to_string()` for 5xx.** Database URLs, hostnames, and SQL leak.
Redact for the client and log the full error.

**Using `anyhow::Error` as the handler error type.** Everything becomes 500.

**Returning one validation error at a time.** Collect the whole `Vec` first.

**Swapping 401 and 403.** 401 is "authenticate"; 403 is "authenticated but not
allowed".

**Forgetting the fallback.** Unrouted paths return an empty `text/plain` 404
while every other error is JSON.

**Adding a variant and only updating `status()`.** The exhaustive `match` in
`code()` will catch it — don't add a `_ =>` arm that defeats that.

## Exercises

1. Build the `AppError` enum with `NotFound`, `Validation`, `Conflict`, and
   `Internal`, plus `IntoResponse`, and hit each from a handler with curl.
2. Add a `TooManyRequests { retry_after: u64 }` variant mapping to 429 with a
   `Retry-After` header. (Hint: build the `Response` with the header in
   `into_response`.)
3. Change the envelope to include a `request_id` read from an
   `Extension<RequestId>` inserted by middleware, and confirm the same id appears
   in both the log line and the JSON body.
4. Write a handler returning `Err(AppError::Internal(anyhow!("db password is
   hunter2")))` and prove with curl that the string never appears in the
   response, then find it in the log.
5. Add a variant wrapping `std::num::ParseIntError` with `#[from]`, and write a
   handler that parses a query param with `?` so the conversion is implicit. Make
   it a 400, not a 500.

---

[← 17.5 Shared State and Dependency Injection](05-state.md) · [Course Index](../README.md) · [17.7 Middleware and tower Layers →](07-middleware.md)
