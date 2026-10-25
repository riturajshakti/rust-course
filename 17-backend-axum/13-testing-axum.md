[← 17.12 HTTP Clients with reqwest](12-reqwest.md) · [Course Index](../README.md) · [18.1 Configuration and Environment →](../18-production/01-config.md)

# 17.13 — Testing an Axum Application

## What You'll Learn

- Three levels: calling a handler as a function, `oneshot` on the `Router`, and a
  real server on port 0
- Asserting status and JSON body, and the `http-body-util` incantation to read one
- Test-database strategy, fixtures, testing auth-protected routes
- `wiremock` for upstream calls, and what to unit vs integration test

Every test on this page was run; the `cargo test` output is real.

## Coming From Another Language

| Other | Rust |
|---|---|
| `supertest(app).get('/x').expect(200)` | `app.oneshot(Request..)` then assert |
| `pytest` + `TestClient(app)` | `axum_test::TestServer` |
| Go `httptest.NewRecorder()` | `oneshot` — same "no socket" idea |

The thing that makes axum pleasant to test: a handler is **an ordinary async
function**. There is no framework context to fake, so the cheapest test is just
calling it.

## Setup

```bash
cargo add --dev tower --features util      # for oneshot
cargo add --dev http-body-util             # to read a response body
cargo add --dev reqwest --features json    # for real-server tests
cargo add --dev wiremock                   # to mock upstreams
cargo add --dev axum-test                  # optional, nicer assertions
```

## Structure the app so it can be tested

The one structural requirement: **`main.rs` must not own the router**. Put
everything in `lib.rs` and expose a function that builds a `Router` from a state
value.

```rust
// src/lib.rs
pub fn router(state: AppState) -> Router {
    Router::new()
        .route("/health", get(health))
        .route("/todos", get(list).post(create))
        .route("/todos/{id}", get(show).delete(remove))
        .with_state(state)
}

pub async fn migrate(pool: &SqlitePool) -> Result<(), sqlx::Error> {
    sqlx::query("CREATE TABLE IF NOT EXISTS todos (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        title TEXT NOT NULL,
        done INTEGER NOT NULL DEFAULT 0)").execute(pool).await?;
    Ok(())
}
```

```rust
// src/main.rs — wiring only
#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let pool = SqlitePoolOptions::new().max_connections(1).connect("sqlite::memory:").await?;
    migrate(&pool).await?;
    let app = router(AppState { pool });
    let listener = tokio::net::TcpListener::bind("127.0.0.1:0").await?;
    axum::serve(listener, app).await?;
    Ok(())
}
```

Files in `tests/` are separate crates that can only see your `pub` items, so
`pub` on `router`, `AppState`, and the DTOs is the price of integration tests.
A binary-only crate cannot be integration-tested at all.

The app under test (abridged — `show` 404s on a missing id, `create` rejects a
blank title, `remove` requires an `ApiKey` extractor matching
`Authorization: Bearer test-key`):

```rust
pub async fn health() -> impl IntoResponse { Json(json!({ "status": "ok" })) }

pub async fn show(State(st): State<AppState>, Path(id): Path<i64>)
    -> Result<Json<Todo>, (StatusCode, Json<Value>)>
{
    sqlx::query_as::<_, Todo>("SELECT id, title, done FROM todos WHERE id = ?")
        .bind(id).fetch_optional(&st.pool).await
        .map_err(|_| (StatusCode::INTERNAL_SERVER_ERROR, Json(json!({"error":"db"}))))?
        .map(Json)
        .ok_or((StatusCode::NOT_FOUND, Json(json!({"error":"not found","id":id}))))
}
```

## Test helpers

Three small functions carry every test below.

```rust
async fn test_pool() -> SqlitePool {
    let pool = SqlitePoolOptions::new()
        .max_connections(1)
        .connect("sqlite::memory:").await.unwrap();
    migrate(&pool).await.unwrap();
    pool
}

async fn seed(pool: &SqlitePool, titles: &[&str]) {
    for t in titles {
        sqlx::query("INSERT INTO todos (title) VALUES (?)").bind(t).execute(pool).await.unwrap();
    }
}

async fn body_json(res: axum::response::Response) -> serde_json::Value {
    let bytes = res.into_body().collect().await.unwrap().to_bytes();
    serde_json::from_slice(&bytes).unwrap()
}
```

`body_json` is the one piece of ceremony worth memorizing: `into_body()`,
`.collect().await` (from `http_body_util::BodyExt`), `.to_bytes()`. An axum body
is a stream, so there is no `res.text()` — forgetting the `BodyExt` import is the
classic first stumble, and the error is `no method named collect`.

## Level 1: call the handler directly

The cheapest test. No router, no HTTP, no serialization.

```rust
#[tokio::test]
async fn handler_called_directly() {
    let res = health().await;
    let res = axum::response::IntoResponse::into_response(res);
    assert_eq!(res.status(), StatusCode::OK);
    assert_eq!(body_json(res).await, serde_json::json!({"status": "ok"}));
}
```

With extractors, construct them by hand — they are plain tuple structs:

```rust
#[tokio::test]
async fn show_handler_directly() {
    let pool = test_pool().await;
    seed(&pool, &["buy milk"]).await;
    let state = AppState { pool };

    let axum::Json(todo) = show(State(state.clone()), Path(1)).await.unwrap();
    assert_eq!(todo, Todo { id: 1, title: "buy milk".into(), done: false });

    let (status, _) = show(State(state), Path(999)).await.unwrap_err();
    assert_eq!(status, StatusCode::NOT_FOUND);
}
```

Note the handler's `Result` gives you typed access to both branches — you assert
on the `Err` value, not a parsed response body.

What this level **cannot** test: routing, extractor rejections, middleware,
serialization of the response. For business logic in a handler it is ideal; for
anything about HTTP, go one level up.

## Level 2: `oneshot` on the `Router`

`tower::ServiceExt::oneshot` drives one request through the whole router —
routing, extractors, middleware, response building — with no socket.

```rust
#[tokio::test]
async fn oneshot_get_list() {
    let pool = test_pool().await;
    seed(&pool, &["a", "b"]).await;
    let app = router(AppState { pool });

    let res = app
        .oneshot(Request::builder().uri("/todos").body(Body::empty()).unwrap())
        .await
        .unwrap();

    assert_eq!(res.status(), StatusCode::OK);
    let json = body_json(res).await;
    assert_eq!(json.as_array().unwrap().len(), 2);
    assert_eq!(json[0]["title"], "a");
}
```

`oneshot` consumes the router, so build a fresh one per request (or `.clone()`
it — `Router` is cheap to clone).

A POST, asserting the response **and** the database:

```rust
#[tokio::test]
async fn oneshot_post_creates() {
    let pool = test_pool().await;
    let app = router(AppState { pool: pool.clone() });

    let res = app
        .oneshot(
            Request::builder()
                .method("POST")
                .uri("/todos")
                .header(header::CONTENT_TYPE, "application/json")
                .body(Body::from(r#"{"title":"write tests"}"#))
                .unwrap(),
        )
        .await
        .unwrap();

    assert_eq!(res.status(), StatusCode::CREATED);
    let json = body_json(res).await;
    assert_eq!(json["id"], 1);
    assert_eq!(json["title"], "write tests");
    assert_eq!(json["done"], false);

    let n: i64 = sqlx::query_scalar("SELECT COUNT(*) FROM todos").fetch_one(&pool).await.unwrap();
    assert_eq!(n, 1);
}
```

Cloning the pool before moving it into the state is what lets you check the side
effect afterwards. Asserting the response alone would pass even if the insert
silently did nothing.

The failure cases are the ones worth the most, because they exercise code you
did not write:

```rust
#[tokio::test]
async fn oneshot_post_rejects_blank_title() {
    let app = router(AppState { pool: test_pool().await });
    let res = app
        .oneshot(Request::builder().method("POST").uri("/todos")
            .header(header::CONTENT_TYPE, "application/json")
            .body(Body::from(r#"{"title":"   "}"#)).unwrap())
        .await.unwrap();
    assert_eq!(res.status(), StatusCode::UNPROCESSABLE_ENTITY);
    assert_eq!(body_json(res).await["error"], "title required");
}

#[tokio::test]
async fn oneshot_post_rejects_missing_content_type() {
    let app = router(AppState { pool: test_pool().await });
    let res = app
        .oneshot(Request::builder().method("POST").uri("/todos")
            .body(Body::from(r#"{"title":"x"}"#)).unwrap())
        .await.unwrap();
    assert_eq!(res.status(), StatusCode::UNSUPPORTED_MEDIA_TYPE);
}

#[tokio::test]
async fn unknown_route_404s() {
    let app = router(AppState { pool: test_pool().await });
    let res = app.oneshot(Request::builder().uri("/nope").body(Body::empty()).unwrap())
        .await.unwrap();
    assert_eq!(res.status(), StatusCode::NOT_FOUND);
}
```

The 415 is axum's `Json` extractor rejecting the request — a level-1 test cannot
reach it, because you would have constructed the `Json<T>` yourself.

## Testing auth-protected routes

Both branches in one test, since the pair is the actual contract:

```rust
#[tokio::test]
async fn delete_requires_auth() {
    let pool = test_pool().await;
    seed(&pool, &["doomed"]).await;

    let app = router(AppState { pool: pool.clone() });
    let res = app
        .oneshot(Request::builder().method("DELETE").uri("/todos/1").body(Body::empty()).unwrap())
        .await.unwrap();
    assert_eq!(res.status(), StatusCode::UNAUTHORIZED);

    let app = router(AppState { pool: pool.clone() });
    let res = app
        .oneshot(Request::builder().method("DELETE").uri("/todos/1")
            .header(header::AUTHORIZATION, "Bearer test-key")
            .body(Body::empty()).unwrap())
        .await.unwrap();
    assert_eq!(res.status(), StatusCode::NO_CONTENT);

    let n: i64 = sqlx::query_scalar("SELECT COUNT(*) FROM todos").fetch_one(&pool).await.unwrap();
    assert_eq!(n, 0);
}

#[tokio::test]
async fn delete_missing_is_404() {
    let app = router(AppState { pool: test_pool().await });
    let res = app
        .oneshot(Request::builder().method("DELETE").uri("/todos/42")
            .header(header::AUTHORIZATION, "Bearer test-key")
            .body(Body::empty()).unwrap())
        .await.unwrap();
    assert_eq!(res.status(), StatusCode::NOT_FOUND);
}
```

Testing the unauthorized case matters more than the authorized one: that is the
assertion that fails when someone removes the extractor from the signature.

With real JWTs (lesson 17.11), mint a token in a helper rather than pasting a
fixed string, so an expiry change does not break every test:

```rust
fn token_for(keys: &Keys, user_id: u64, role: &str) -> String {
    let iat = now();
    let claims = Claims { sub: user_id.to_string(), role: role.into(), iat, exp: iat + 900 };
    encode(&Header::default(), &claims, &keys.enc).unwrap()
}
```

A deliberately expired token (`exp: iat - 1`) is how you test the 401-expired
path without sleeping.

## Level 3: a real server on port 0

For anything touching real sockets — a client library, keep-alive, compression,
streaming — spawn the server on an OS-assigned port.

```rust
async fn spawn_app() -> String {
    let pool = test_pool().await;
    seed(&pool, &["from server"]).await;
    let app = router(AppState { pool });
    let listener = tokio::net::TcpListener::bind("127.0.0.1:0").await.unwrap();
    let addr = listener.local_addr().unwrap();
    tokio::spawn(async move { axum::serve(listener, app).await.unwrap() });
    format!("http://{addr}")
}

#[tokio::test]
async fn real_server_with_reqwest() {
    let base = spawn_app().await;
    let client = reqwest::Client::new();

    let res = client.get(format!("{base}/health")).send().await.unwrap();
    assert_eq!(res.status(), 200);
    assert_eq!(res.json::<serde_json::Value>().await.unwrap()["status"], "ok");

    let todos: Vec<Todo> = client.get(format!("{base}/todos")).send().await.unwrap()
        .json().await.unwrap();
    assert_eq!(todos, vec![Todo { id: 1, title: "from server".into(), done: false }]);

    let created: Todo = client.post(format!("{base}/todos"))
        .json(&serde_json::json!({"title":"via reqwest"}))
        .send().await.unwrap()
        .json().await.unwrap();
    assert_eq!(created.id, 2);

    let res = client.get(format!("{base}/todos/999")).send().await.unwrap();
    assert_eq!(res.status(), 404);
}
```

Three details that make this reliable:

- **Port 0**, so tests run in parallel without port collisions. A hardcoded 3000
  makes your suite serial and flaky.
- `bind` **before** `tokio::spawn`, and read `local_addr()` from the bound
  listener. Spawning first means racing the server's startup, and the usual
  "fix" is a `sleep` that is either too short or too slow.
- No shutdown needed — the spawned task dies when the test process exits.

## `axum-test` for less boilerplate

```bash
cargo add --dev axum-test
```

```rust
use axum_test::TestServer;

#[tokio::test]
async fn axum_test_crate() {
    let pool = test_pool().await;
    let server = TestServer::new(router(AppState { pool }));

    server.get("/health").await.assert_status_ok();
    server.get("/health").await.assert_json(&serde_json::json!({"status":"ok"}));

    let created = server.post("/todos").json(&serde_json::json!({"title":"from axum-test"})).await;
    created.assert_status(StatusCode::CREATED);
    assert_eq!(created.json::<Todo>().id, 1);

    server.get("/todos/999").await.assert_status_not_found();

    server.delete("/todos/1").await.assert_status_unauthorized();
    server.delete("/todos/1")
        .add_header("authorization", "Bearer test-key")
        .await
        .assert_status(StatusCode::NO_CONTENT);
}
```

No `Request::builder`, no `body_json`, and the assertion failures name the
status they expected. It also keeps cookies across requests, which makes a
session-based login flow testable in a few lines. The trade-off is a dev
dependency with its own version churn; `oneshot` needs nothing but `tower`.

## Test database strategy

Four options, in order of how much they cost to set up:

**1. In-memory SQLite per test** — what this lesson uses. Instant, fully
isolated, zero setup. The catch: it is not the database you deploy to, so
Postgres-specific SQL (`RETURNING` on a conflict, `jsonb` operators, `= ANY($1)`)
is untested. Fine for handler logic, not for your queries.

Note `max_connections(1)` in `test_pool`: with `sqlite::memory:` **each
connection gets its own empty database**, so a larger pool makes tests
mysteriously see no rows.

**2. Transaction per test, rolled back.** Fast and isolated against a real
Postgres. Each test takes a transaction, runs everything inside it, and never
commits:

```rust
#[tokio::test]
async fn transaction_per_test_rolls_back() {
    let pool = test_pool().await;
    seed(&pool, &["kept"]).await;

    {
        let mut tx = pool.begin().await.unwrap();
        sqlx::query("INSERT INTO todos (title) VALUES ('temporary')")
            .execute(&mut *tx).await.unwrap();
        let n: i64 = sqlx::query_scalar("SELECT COUNT(*) FROM todos")
            .fetch_one(&mut *tx).await.unwrap();
        assert_eq!(n, 2);
    }

    let n: i64 = sqlx::query_scalar("SELECT COUNT(*) FROM todos").fetch_one(&pool).await.unwrap();
    assert_eq!(n, 1);
}
```

Drop-rollback (lesson 17.9) does the cleanup. The real limitation: the code
under test must accept the transaction rather than the pool, so handlers taking
`&pool` from `State` cannot be tested this way — and a handler that opens its
own transaction will not see your uncommitted data. Works beautifully for
repository-layer tests, awkwardly for handler tests.

**3. A fresh schema (or database) per test.** The most faithful. Create a uniquely
named schema, migrate, run, drop:

```rust
async fn fresh_db() -> PgPool {
    let name = format!("test_{}", uuid::Uuid::new_v4().simple());
    let admin = PgPool::connect(&std::env::var("DATABASE_URL").unwrap()).await.unwrap();
    sqlx::query(&format!("CREATE DATABASE {name}")).execute(&admin).await.unwrap();
    let url = format!("{}/{name}", base_url());
    let pool = PgPool::connect(&url).await.unwrap();
    sqlx::migrate!().run(&pool).await.unwrap();
    pool
}
```

Fully parallel, exercises your real migrations and real SQL. Costs a few hundred
milliseconds per test; use `CREATE DATABASE ... TEMPLATE` to cut that.

**4. One shared database with cleanup.** `TRUNCATE` between tests. Cheap to
write, and forces `cargo test -- --test-threads=1` because tests fight over the
same rows. Avoid.

The practical answer: option 1 or 2 for the bulk of the suite, option 3 for a
smaller set of tests that run the real queries.

## Tests are isolated by default

Each `#[tokio::test]` gets its own runtime, and these two get their own database:

```rust
#[tokio::test]
async fn tests_are_isolated_a() {
    let pool = test_pool().await;
    seed(&pool, &["only in a"]).await;
    let n: i64 = sqlx::query_scalar("SELECT COUNT(*) FROM todos").fetch_one(&pool).await.unwrap();
    assert_eq!(n, 1);
}

#[tokio::test]
async fn tests_are_isolated_b() {
    let pool = test_pool().await;
    let n: i64 = sqlx::query_scalar("SELECT COUNT(*) FROM todos").fetch_one(&pool).await.unwrap();
    assert_eq!(n, 0);
}
```

Both pass regardless of order, which is the property to protect. Cargo runs
tests in **parallel threads** by default, so any shared mutable resource — a
fixed port, a shared database, a temp file with a fixed name — is a race.

## Fixtures and factories

A factory with sensible defaults keeps a test about the one field it cares
about:

```rust
fn todo_factory(title: &str) -> NewTodo {
    NewTodo { title: title.to_string() }
}

fn user_factory() -> NewUser {
    NewUser {
        email: format!("user-{}@example.com", uuid::Uuid::new_v4().simple()),
        password: "correct horse battery staple".into(),
        role: "user".into(),
    }
}
```

Unique generated emails avoid unique-constraint collisions when tests share a
database. Override only what matters with struct update syntax:

```rust
let admin = NewUser { role: "admin".into(), ..user_factory() };
```

SQLx also has `#[sqlx::test]`, which creates a fresh migrated database per test
and hands you the pool, with optional `fixtures(...)` loading SQL files:

```rust
#[sqlx::test(fixtures("todos"))]
async fn with_fixtures(pool: SqlitePool) {
    let n: i64 = sqlx::query_scalar("SELECT COUNT(*) FROM todos").fetch_one(&pool).await.unwrap();
    assert_eq!(n, 3);
}
```

It replaces the `test_pool` helper and the `#[tokio::test]` attribute both.

## Mocking upstream HTTP

A handler that calls an external API should not call it in tests. `wiremock`
(lesson 17.12) starts a real server on a random port; make the base URL part of
`AppState` so a test can point it anywhere:

```rust
#[derive(Clone)]
pub struct AppState { pub pool: SqlitePool, pub http: Client, pub upstream: String }

#[tokio::test]
async fn handler_handles_upstream_500() {
    let mock = MockServer::start().await;
    Mock::given(method("GET")).and(path("/rates"))
        .respond_with(ResponseTemplate::new(500))
        .mount(&mock).await;

    let app = router(AppState {
        pool: test_pool().await,
        http: Client::new(),
        upstream: mock.uri(),
    });

    let res = app.oneshot(Request::builder().uri("/convert?from=EUR").body(Body::empty()).unwrap())
        .await.unwrap();
    assert_eq!(res.status(), StatusCode::BAD_GATEWAY);
}
```

That last assertion is the valuable one: it pins the mapping from "upstream
broke" to "we return 502", which is easy to get wrong and impossible to test
against the live API.

## What to unit vs integration test

| Test | Level | What it proves |
|---|---|---|
| Pure logic (price calc, slug, validation rule) | `#[test]` in the module | the algorithm |
| Handler branches | call the function | the decision, not the HTTP |
| Routing, extractors, status codes, middleware | `oneshot` | the HTTP contract |
| Auth rejection on every protected route | `oneshot` | nobody dropped the extractor |
| SQL queries | integration, real DB | the SQL is valid and does what you think |
| Upstream failure mapping | `oneshot` + `wiremock` | your error taxonomy |
| Full flows (register → login → use token) | real server + reqwest | the pieces fit |

Where to spend effort, for a web service specifically: **the integration tests
are the valuable ones**. Most bugs are in the wiring — a wrong status code, an
extractor in the wrong order, SQL that compiles but filters wrongly, an error
mapped to 500 instead of 404 — and none of those are reachable by a unit test.
A unit test of `fn handler()` that mocks the repository mostly asserts that you
wrote the handler you wrote.

Three tests that repay themselves immediately:

1. Every protected route returns 401 without credentials.
2. Every endpoint's error path returns the status and envelope you documented.
3. One happy-path flow through the real server.

## Running them

```bash
cargo test                       # unit + all integration targets
cargo test --test api            # one file in tests/
cargo test --lib                 # only #[test]/#[tokio::test] inside src/
cargo test oneshot               # name filter
cargo test -- --nocapture        # show println! from passing tests
cargo test -- --test-threads=1   # serial, if you have shared state
```

```console
$ cargo test
running 13 tests
test handler_called_directly ... ok
test unknown_route_404s ... ok
test oneshot_post_rejects_missing_content_type ... ok
test oneshot_post_rejects_blank_title ... ok
test tests_are_isolated_b ... ok
test delete_missing_is_404 ... ok
test show_handler_directly ... ok
test tests_are_isolated_a ... ok
test oneshot_post_creates ... ok
test oneshot_get_list ... ok
test delete_requires_auth ... ok
test transaction_per_test_rolls_back ... ok
test real_server_with_reqwest ... ok

test result: ok. 13 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.05s

running 1 test
test axum_test_crate ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Thirteen integration tests covering routing, extractors, auth, JSON rejections,
database side effects, and a real socket, in 50ms. That speed is why axum
services end up well tested.

A failure, so you know what to read:

```
---- oneshot_get_list stdout ----

thread 'oneshot_get_list' panicked at tests/api.rs:64:5:
assertion `left == right` failed
  left: String("a")
 right: "z"

failures:
    oneshot_get_list

test result: FAILED. 0 passed; 1 failed; 0 ignored; 0 measured; 12 filtered out
```

`left` is the actual value, `right` the expected — the opposite of some other
languages, and a `String("a")` vs `"z"` mismatch there means a `serde_json::Value`
compared against a `&str` literal, which works but prints asymmetrically.

## Quick Reference

| Need | Form |
|---|---|
| Testable layout | `lib.rs` with `pub fn router(state) -> Router` |
| Async test | `#[tokio::test] async fn ...` |
| Call a handler | `handler(State(s), Path(1)).await` |
| Drive the router | `app.oneshot(Request::builder()...)` (`tower` `util` feature) |
| Build a request | `Request::builder().method("POST").uri("/x").header(..).body(Body::from(..))` |
| Read the body | `res.into_body().collect().await?.to_bytes()` (`http_body_util::BodyExt`) |
| Assert status | `assert_eq!(res.status(), StatusCode::CREATED)` |
| Assert JSON | parse to `serde_json::Value` or your DTO, then `assert_eq!` |
| Real server | `TcpListener::bind("127.0.0.1:0")` + `tokio::spawn(axum::serve(..))` |
| Nicer assertions | `axum_test::TestServer::new(router)` |
| Fresh DB per test | `#[sqlx::test]`, or a per-test in-memory pool |
| Load SQL fixtures | `#[sqlx::test(fixtures("name"))]` |
| Roll back | take a `Transaction`, never commit |
| Mock an upstream | `wiremock::MockServer` + a base URL in `AppState` |
| Auth header | `.header(header::AUTHORIZATION, "Bearer ...")` |
| One file | `cargo test --test api` |
| See output | `cargo test -- --nocapture` |

## Common Mistakes

**Putting the router in `main.rs`.** Integration tests cannot reach it. Move it
to `lib.rs` and make it `pub`.

**Forgetting `use http_body_util::BodyExt`.** `no method named collect` on the
body, with no hint about the import.

**Forgetting `features = ["util"]` on dev-dependency `tower`.** `oneshot` does
not exist.

**Reusing a `Router` after `oneshot`.** It consumes `self`. Rebuild or clone.

**`max_connections` above 1 with `sqlite::memory:`.** Each connection gets its
own empty database and rows vanish.

**A hardcoded port in tests.** Collides under parallel runs. Bind port 0.

**`tokio::spawn` before `bind`.** You race startup. Bind, read `local_addr`, then
spawn.

**Asserting only the response on a write.** Clone the pool and check the row.

**Only testing the authorized path.** The 401 test is the one that catches a
deleted extractor.

**Reaching for `--test-threads=1` to fix flakiness.** That hides shared state
instead of removing it.

**Calling a live upstream in tests.** Slow, flaky, and you cannot make it return
a 500. Use `wiremock`.

## Exercises

1. Restructure a server so `lib.rs` exposes `pub fn router(state) -> Router`,
   and write one `oneshot` test asserting `GET /health` returns 200 with
   `{"status":"ok"}`.
2. Write `body_json` as a helper and use it to assert the full JSON body of a
   `POST /todos` 201 response, including the generated id.
3. Add tests for all four failure modes of a JSON endpoint: missing field,
   wrong type, malformed JSON, no content type. Record the four statuses.
4. Test a protected route twice: without an `Authorization` header (401) and
   with a valid one (200). Then delete the extractor from the handler and watch
   only the first test fail.
5. Write `spawn_app() -> String` returning a base URL on port 0, and a reqwest
   test that creates a todo, reads it back, deletes it, and confirms a
   subsequent GET is 404.
6. Convert the suite to `#[sqlx::test]` with a SQL fixture file seeding three
   rows, and confirm each test still sees exactly three.
7. Add a handler calling an upstream API, then a `wiremock` test asserting an
   upstream 500 becomes a 502 from your service.

---

[← 17.12 HTTP Clients with reqwest](12-reqwest.md) · [Course Index](../README.md) · [18.1 Configuration and Environment →](../18-production/01-config.md)
