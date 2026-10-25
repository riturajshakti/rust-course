[← Course Index](../README.md)

# 17.5 — Shared State and Dependency Injection

## What You'll Learn

- `State<T>` and `.with_state()`, and why it is compile-time checked
- `Arc<AppState>` vs a `Clone`-able struct of `Arc`s
- Substate extraction with `FromRef`
- A full in-memory CRUD app behind `Arc<RwLock<...>>`, and trait-based DI

## Coming From Another Language

| Other | axum |
|---|---|
| Express `app.locals.db` | `State<AppState>` |
| FastAPI `Depends(get_db)` | an extractor reading `State` |
| Go passing `*Server` into handler methods | `State<AppState>` on each handler |
| A module-level global / singleton | a `Clone`-able `AppState` the router owns |

Axum's state is **typed and checked at compile time**: a router whose handlers
take `State<AppState>` has type `Router<AppState>` and cannot be served until
you call `.with_state(...)`. There is no "forgot to register the dependency"
runtime crash.

## The basic shape

```rust
use axum::{Router, extract::State, routing::get};

#[derive(Clone)]
struct AppState { app_name: String }

async fn info(State(st): State<AppState>) -> String {
    st.app_name
}

let app = Router::new()
    .route("/info", get(info))
    .with_state(AppState { app_name: "todos".into() });
```

Two requirements on the state type: `Clone` and `Send + Sync + 'static`. Axum
clones it once per request, so cloning must be cheap — which is what all the
`Arc`s below are for.

## Forgetting `.with_state` is a compile error

```rust
#[derive(Clone)]
struct AppState { n: u32 }

async fn h(State(s): State<AppState>) -> String { format!("{}", s.n) }

let app: Router = Router::new().route("/", get(h));
```

Does not compile:

```
error[E0308]: mismatched types
  --> src/main.rs:10:48
   |
10 |     let app: Router = Router::new().route("/", get(h));
   |                                     -----      ^^^^^^ expected `MethodRouter`, found `MethodRouter<AppState>`
   |                                     |
   |                                     arguments to this method are incorrect
   |
   = note: expected struct `MethodRouter<()>`
              found struct `MethodRouter<AppState>`
```

Intentionally broken. `Router` is shorthand for `Router<()>`. Read the error as
"this router still needs state" — the fix is `.with_state(...)`, or declaring
the function's return type `Router<AppState>` if it is a sub-router whose state
is supplied later.

## `Arc<AppState>` vs a struct of `Arc`s

Two shapes, both common:

```rust
// Shape A: wrap the whole thing.
type SharedState = Arc<AppState>;
struct AppState { pool: PgPool, config: Config }
// handlers: State(st): State<Arc<AppState>>, then st.pool

// Shape B: a Clone-able struct holding cheap handles.
#[derive(Clone)]
struct AppState { pool: PgPool, config: Config }
// handlers: State(st): State<AppState>, then st.pool
```

Shape B is idiomatic in axum, because the things you actually share are already
cheaply clonable: `sqlx::PgPool` is an `Arc` internally, `reqwest::Client` is an
`Arc` internally, and a small `Config` clone is a few allocations once per
request. Wrap in `Arc` only the fields that are genuinely expensive or that need
interior mutability.

Shape A buys you one thing: `State<Arc<AppState>>` makes the clone a single
atomic increment no matter how many fields you add. If `AppState` grows a dozen
`String` config fields, switch. Mixing is fine too — `Arc<Config>` inside a
`Clone` state.

## Substate with `FromRef`

A handler that only needs the config should not have to know about the pool.
`FromRef` lets `State<Config>` be extracted from an `AppState`.

```rust
use axum::extract::FromRef;

#[derive(Clone, Debug)]
struct Config { app_name: String, max_todos: usize }

#[derive(Clone)]
struct AppState {
    config: Config,
    todos: Arc<RwLock<HashMap<u64, Todo>>>,
    next_id: Arc<AtomicU64>,
    hits: Arc<AtomicU64>,
}

impl FromRef<AppState> for Config {
    fn from_ref(s: &AppState) -> Config { s.config.clone() }
}

async fn info(State(cfg): State<Config>) -> impl IntoResponse {
    Json(json!({ "app": cfg.app_name, "max_todos": cfg.max_todos }))
}
```

```console
$ curl -s localhost:52157/info
{"app":"todos","max_todos":3}
```

The router is still `Router<AppState>` and `.with_state(state)` is unchanged;
axum calls `Config::from_ref(&state)` for that handler. `FromRef` is also how a
custom extractor reads state — the JWT extractor in lesson 17.11 is generic over
`S: Send + Sync` with `Keys: FromRef<S>`, making it reusable across apps.

Note `from_ref` returns an owned value, so it clones. Keep substates small or
make them `Arc<Config>`.

## A full CRUD app on `Arc<RwLock<...>>`

An in-memory store is the right first step: it exercises state, concurrency, and
every HTTP verb without a database. Swap the `HashMap` for SQLx in lesson 17.8
and the handlers barely change.

```rust
use std::sync::{Arc, RwLock, atomic::{AtomicU64, Ordering}};

#[derive(Clone, Debug, Serialize)]
struct Todo { id: u64, title: String, done: bool }

#[derive(Deserialize)]
struct CreateTodo { title: String }

#[derive(Deserialize)]
struct UpdateTodo { title: Option<String>, done: Option<bool> }
```

`UpdateTodo`'s `Option` fields are what make PATCH semantics work: absent means
"leave alone", present means "set to this".

Create, with a capacity check:

```rust
async fn create(State(st): State<AppState>, Json(body): Json<CreateTodo>) -> impl IntoResponse {
    let mut guard = st.todos.write().unwrap();
    if guard.len() >= st.config.max_todos {
        return (StatusCode::CONFLICT, Json(json!({ "error": "full" }))).into_response();
    }
    let id = st.next_id.fetch_add(1, Ordering::Relaxed);
    let todo = Todo { id, title: body.title, done: false };
    guard.insert(id, todo.clone());
    (StatusCode::CREATED, Json(todo)).into_response()
}
```

```console
$ curl -s -X POST localhost:52157/todos -H 'content-type: application/json' -d '{"title":"write lesson"}'
{"id":1,"title":"write lesson","done":false}
$ curl -s -X POST localhost:52157/todos -H 'content-type: application/json' -d '{"title":"run examples"}'
{"id":2,"title":"run examples","done":false}
$ curl -s -X POST localhost:52157/todos -H 'content-type: application/json' -d '{"title":"ship it"}'
{"id":3,"title":"ship it","done":false}

$ curl -si -X POST localhost:52157/todos -H 'content-type: application/json' -d '{"title":"extra"}' | tail -1
{"error":"full"}
```

Note the capacity check happens **while holding the write lock**. Checking the
length first and inserting after would be a race: two requests could both see
`len() == 2` and both insert.

List, with a read lock:

```rust
async fn list(State(st): State<AppState>) -> Json<Vec<Todo>> {
    st.hits.fetch_add(1, Ordering::Relaxed);
    let guard = st.todos.read().unwrap();
    let mut v: Vec<Todo> = guard.values().cloned().collect();
    v.sort_by_key(|t| t.id);
    Json(v)
}
```

```console
$ curl -s localhost:52157/todos
[{"id":1,"title":"write lesson","done":false},{"id":2,"title":"run examples","done":false},{"id":3,"title":"ship it","done":false}]
```

Read, with 404:

```rust
async fn show(State(st): State<AppState>, Path(id): Path<u64>) -> impl IntoResponse {
    match st.todos.read().unwrap().get(&id) {
        Some(t) => (StatusCode::OK, Json(t.clone())).into_response(),
        None => (StatusCode::NOT_FOUND, Json(json!({ "error": "not found", "id": id }))).into_response(),
    }
}
```

```console
$ curl -s localhost:52157/todos/2
{"id":2,"title":"run examples","done":false}

$ curl -si localhost:52157/todos/99 | tail -1
{"error":"not found","id":99}
```

Partial update:

```rust
async fn update(
    State(st): State<AppState>,
    Path(id): Path<u64>,
    Json(p): Json<UpdateTodo>,
) -> impl IntoResponse {
    let mut guard = st.todos.write().unwrap();
    match guard.get_mut(&id) {
        Some(t) => {
            if let Some(title) = p.title { t.title = title; }
            if let Some(done) = p.done { t.done = done; }
            (StatusCode::OK, Json(t.clone())).into_response()
        }
        None => (StatusCode::NOT_FOUND, Json(json!({ "error": "not found" }))).into_response(),
    }
}
```

```console
$ curl -s -X PATCH localhost:52157/todos/2 -H 'content-type: application/json' -d '{"done":true}'
{"id":2,"title":"run examples","done":true}

$ curl -s -X PATCH localhost:52157/todos/2 -H 'content-type: application/json' -d '{"title":"run ALL examples"}'
{"id":2,"title":"run ALL examples","done":true}
```

The second PATCH left `done: true` alone — that is the `Option` doing its job.

Delete:

```rust
async fn remove(State(st): State<AppState>, Path(id): Path<u64>) -> StatusCode {
    if st.todos.write().unwrap().remove(&id).is_some() {
        StatusCode::NO_CONTENT
    } else {
        StatusCode::NOT_FOUND
    }
}
```

```console
$ curl -si -X DELETE localhost:52157/todos/1 | head -1
HTTP/1.1 204 No Content

$ curl -si -X DELETE localhost:52157/todos/1 | head -1
HTTP/1.1 404 Not Found

$ curl -s localhost:52157/todos
[{"id":2,"title":"run ALL examples","done":true},{"id":3,"title":"ship it","done":false}]
```

Counters without a lock, via the substate-free `AtomicU64`:

```rust
async fn stats(State(st): State<AppState>) -> impl IntoResponse {
    let n = st.todos.read().unwrap().len();
    Json(json!({ "todos": n, "hits": st.hits.load(Ordering::Relaxed) }))
}
```

```console
$ curl -s localhost:52157/stats
{"hits":2,"todos":2}
```

## Wiring it up

```rust
let app = Router::new()
    .route("/info", get(info))
    .route("/stats", get(stats))
    .route("/todos", get(list).post(create))
    .route("/todos/{id}", get(show).patch(update).delete(remove))
    .with_state(state);
```

`.with_state` goes **last**, after every route that needs it. Calling it early
turns the router into `Router<()>` and subsequent `.route` calls with stateful
handlers won't typecheck.

## `std::sync::RwLock` or `tokio::sync::RwLock`?

Use `std::sync::RwLock` when the critical section is short, synchronous, and
contains no `.await` — as above. It is faster and the guard code is simpler.

Use `tokio::sync::RwLock` only when you must hold the lock **across an await**
(`let g = lock.write().await; some_io().await;`). Holding a `std` guard across
an await point does not compile, because the guard is not `Send`:

```
error: future cannot be sent between threads safely
   ...
   = note: future is not `Send` as this value is used across an await
```

That error is the signal to switch, not a reason to reach for the tokio lock
pre-emptively.

On `.unwrap()` after `read()`/`write()`: it fails only if another thread panicked
while holding the lock. In a handler that is usually the right behaviour — the
data may be inconsistent. Use `parking_lot::RwLock`, whose guards don't return a
`Result`, if the unwraps bother you.

## `Extension` is the older way

Before `State`, shared values went through `.layer(Extension(x))` and
`Extension<T>` in handlers (lesson 17.3). It still works and is still right for
**per-request** data that middleware computes. It is wrong for app-wide
dependencies:

| | `State<T>` | `Extension<T>` |
|---|---|---|
| Checked | compile time | runtime, by type |
| Missing | won't compile | 500 at request time |
| Cost | one clone | type-map lookup |
| Good for | pool, config, repos | authenticated user, request id |

## Dependency injection via traits

To test handlers without a database, put a trait behind the state field.

```rust
trait UserRepo: Send + Sync + 'static {
    fn find(&self, id: u64) -> Option<User>;
    fn count(&self) -> usize;
}

struct PgRepo;
impl UserRepo for PgRepo {
    fn find(&self, id: u64) -> Option<User> { Some(User { id, email: format!("user{id}@db") }) }
    fn count(&self) -> usize { 1_000_000 }
}

#[derive(Default)]
struct FakeRepo { users: Mutex<Vec<User>> }
impl UserRepo for FakeRepo {
    fn find(&self, id: u64) -> Option<User> {
        self.users.lock().unwrap().iter().find(|u| u.id == id).cloned()
    }
    fn count(&self) -> usize { self.users.lock().unwrap().len() }
}

#[derive(Clone)]
struct AppState { users: Arc<dyn UserRepo> }

async fn whoami(State(st): State<AppState>, Path(id): Path<u64>) -> Json<Value> {
    Json(json!({ "found": st.users.find(id), "total": st.users.count() }))
}

fn router(state: AppState) -> Router {
    Router::new().route("/users/{id}", get(whoami)).with_state(state)
}
```

Both wirings, exercised:

```rust
let prod = AppState { users: Arc::new(PgRepo) };
println!("prod count = {}", prod.users.count());
println!("prod find(5) = {:?}", prod.users.find(5));

let fake = FakeRepo::default();
fake.users.lock().unwrap().push(User { id: 5, email: "test@fake".into() });
let test = AppState { users: Arc::new(fake) };
println!("test count = {}", test.users.count());
println!("test find(5) = {:?}", test.users.find(5));
println!("test find(9) = {:?}", test.users.find(9));
```

Output:

```
prod count = 1000000
prod find(5) = Some(User { id: 5, email: "user5@db" })
test count = 1
test find(5) = Some(User { id: 5, email: "test@fake" })
test find(9) = None
```

`Arc<dyn UserRepo>` is the dynamic choice: one state type, repo swapped at
runtime, a vtable call per method. The alternative is a generic state
`AppState<R: UserRepo>` — monomorphized, zero-cost, but the generic parameter
propagates into every handler signature and router type. Start with `dyn`; reach
for the generic only if a profiler says the vtable matters.

For `async` trait methods — which real repos need — either make the trait
methods `async fn` (allowed in traits since Rust 1.75, but then the trait is not
`dyn`-compatible), or add `#[async_trait::async_trait]` to keep
`Arc<dyn UserRepo>` working. The common choice is `async_trait` for exactly this
reason.

## Initialization order in `main`

```rust
#[tokio::main]
async fn main() -> anyhow::Result<()> {
    tracing_subscriber::fmt::init();              // 1. logging first, so everything below is visible
    let config = Config::from_env()?;             // 2. config, fail fast on a bad value
    let pool = PgPoolOptions::new()               // 3. external connections
        .max_connections(config.db_max_conns)
        .connect(&config.database_url).await?;
    sqlx::migrate!().run(&pool).await?;           // 4. migrations before serving
    let state = AppState { pool, config: config.clone() };  // 5. assemble
    let app = routes::router().with_state(state); // 6. router
    let listener = TcpListener::bind(("0.0.0.0", config.port)).await?;  // 7. bind last
    axum::serve(listener, app).await?;
    Ok(())
}
```

The ordering principle: everything that can fail should fail **before** the
listener binds, so a misconfigured deploy never accepts a request it cannot
serve. A load balancer seeing a refused connection is better than one seeing
500s.

## Quick Reference

| Need | Form |
|---|---|
| Declare | `#[derive(Clone)] struct AppState { ... }` |
| Attach | `.with_state(state)` — last in the chain |
| Extract | `State(st): State<AppState>` |
| Sub-router typed on state | `fn router() -> Router<AppState>` |
| Substate | `impl FromRef<AppState> for Config` then `State<Config>` |
| Shared mutable | `Arc<RwLock<T>>` (std) |
| Shared mutable across await | `Arc<tokio::sync::RwLock<T>>` |
| Counter | `Arc<AtomicU64>` + `fetch_add(1, Ordering::Relaxed)` |
| Swappable dependency | `Arc<dyn Trait>` field, or `AppState<T: Trait>` |
| Per-request value | `Extension<T>` inserted by middleware |

## Common Mistakes

**Calling `.with_state` before adding stateful routes.** The router becomes
`Router<()>` and later `.route` calls won't typecheck.

**Reading `expected MethodRouter, found MethodRouter<AppState>` as a routing
bug.** It means state is missing; add `.with_state`.

**Holding a `std::sync` guard across `.await`.** `future cannot be sent between
threads safely`. Shorten the critical section, or use `tokio::sync::RwLock`.

**Check-then-act outside the lock.** `if store.read().len() < max { store.write().insert(..) }`
is a race. Take the write lock once and check inside it.

**Putting a `PgPool` behind `Arc<Mutex<...>>`.** It is already an `Arc` of a
pool and is designed for concurrent use. A mutex serializes your entire
database access.

**Using `Extension` for the pool.** Runtime-checked; a missing layer is a 500 in
production rather than an error at compile time.

## Exercises

1. Build the todo app above, then add `GET /todos?done=true` filtering with a
   `Query<Filter>` extractor alongside `State`.
2. Add a `started_at: Instant` field to `AppState` and a `GET /uptime` that
   reports seconds since start — using a substate `FromRef` so the handler never
   sees the todo map.
3. Replace the two `AtomicU64`s with one `Arc<RwLock<Metrics>>` struct and
   observe what happens to `list` under `curl` run 100 times in a loop.
4. Define `trait Clock { fn now(&self) -> DateTime<Utc>; }` with a real and a
   fixed implementation, put `Arc<dyn Clock>` in `AppState`, and stamp each todo
   with `created_at`. Build the router twice with different clocks and show the
   timestamps differ.
5. Write a handler that acquires the write lock and then `.await`s a
   `tokio::time::sleep`. Read the compile error, then fix it two ways: narrowing
   the scope, and switching to `tokio::sync::RwLock`.

---

[← Course Index](../README.md)
