[← Course Index](../README.md)

# 17.8 — SQLx: Connecting to Postgres

## What You'll Learn

- `PgPoolOptions`, connection strings, and pool sizing
- `query` vs `query_as` vs `query!` — and exactly why the macros need a live
  database at **compile** time
- `FromRow`, `fetch_one`/`fetch_optional`/`fetch_all`/`execute`
- Parameter binding, the Postgres↔Rust type map, and `NULL` → `Option`

## A note on the runnable examples

Every captured output on this page comes from **SQLite via SQLx**, because this
machine has no Postgres server. The API is identical — `sqlx::query`,
`query_as`, `FromRow`, the pool — so everything transfers. Three differences are
called out inline where they appear:

| | Postgres | SQLite |
|---|---|---|
| Placeholders | `$1`, `$2` | `?` |
| Pool type | `PgPool`, `PgPoolOptions` | `SqlitePool`, `SqlitePoolOptions` |
| Unique-violation SQLSTATE | `"23505"` | `"2067"` |

To follow along exactly as captured:

```bash
cargo add sqlx --features runtime-tokio,sqlite,chrono,uuid
```

For Postgres, swap the feature and the types:

```bash
cargo add sqlx --features runtime-tokio,postgres,chrono,uuid,json
```

SQLx is the query-first choice: you write SQL, it maps rows to structs. Diesel
and SeaORM are the ORM alternatives if you want a query builder and
schema-as-Rust instead.

## Coming From Another Language

| Other | SQLx |
|---|---|
| `pg.Pool` + `pool.query('... $1', [v])` | `sqlx::query("... $1").bind(v)` |
| Prisma / TypeORM entities | no entities; `FromRow` on a plain struct |
| `psycopg` / SQLAlchemy Core | closest analogue — SQL with bound params |
| Go `database/sql` + `sqlx` | nearly a direct port of the idea |

The novelty is `query!`, which **validates your SQL against a real schema while
compiling** and generates a struct from the result columns. No other mainstream
ecosystem does this.

## Connection strings

```
postgres://user:password@host:5432/dbname
postgres://user:password@host:5432/dbname?sslmode=require
postgres:///dbname?host=/var/run/postgresql        # unix socket
sqlite::memory:
sqlite://data.db?mode=rwc                          # create if absent
```

Read it from the environment, never hardcode it:

```rust
let url = std::env::var("DATABASE_URL")?;
```

`DATABASE_URL` is also the variable the `query!` macros and `sqlx-cli` look for,
so using that exact name saves configuration later.

## Creating the pool

```rust
use sqlx::sqlite::SqlitePoolOptions;
use std::time::Duration;

let pool = SqlitePoolOptions::new()
    .max_connections(5)
    .min_connections(1)
    .acquire_timeout(Duration::from_secs(3))
    .idle_timeout(Duration::from_secs(600))
    .max_lifetime(Duration::from_secs(1800))
    .connect("sqlite::memory:")
    .await?;

println!("pool size={} idle={}", pool.size(), pool.num_idle());
```

Output:

```
pool size=1 idle=1
```

The Postgres form is identical with `PgPoolOptions` and a `postgres://` URL.

| Option | Meaning | Sensible value |
|---|---|---|
| `max_connections` | hard cap | see sizing below |
| `min_connections` | kept warm | 1-2 |
| `acquire_timeout` | wait for a free connection, then error | 3-5s |
| `idle_timeout` | close connections idle this long | 10 min |
| `max_lifetime` | recycle regardless | 30 min |
| `test_before_acquire` | ping before handing out | default true |

`.connect()` opens connections eagerly and fails fast on a bad URL —
that is what you want at startup. `.connect_lazy()` returns immediately and
fails on first use; only use it when the database may legitimately not be up yet.

`max_lifetime` matters more than it looks: it survives a failover, where the
pooled connections point at a machine that is now a replica.

## Pool sizing

Start with `max_connections = 2 * CPU cores` for a CPU-bound workload, or
`cores + effective_spindle_count` for a disk-bound one. Then check the ceiling:

```
app_instances × max_connections ≤ postgres max_connections (default 100)
```

Four replicas at 25 connections each already saturates a default Postgres. Go
over and new connections are refused server-side, which looks like a random
outage. Bigger is not better — a pool larger than the database can serve in
parallel just moves queueing from your process into Postgres, where it is harder
to see. Put PgBouncer in front if you need hundreds of app instances, and use
transaction-mode pooling with prepared statements disabled
(`?statement_cache_capacity=0`).

A `Pool` is `Arc` internally. **Clone it freely; never wrap it in a `Mutex`.**

## Pool in `AppState`

```rust
#[derive(Clone)]
struct AppState { pool: SqlitePool }

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let url = std::env::var("DATABASE_URL").unwrap_or_else(|_| "sqlite::memory:".into());
    let pool = SqlitePoolOptions::new()
        .max_connections(5)
        .acquire_timeout(Duration::from_secs(3))
        .connect(&url)
        .await?;

    let app = Router::new()
        .route("/health", get(health))
        .route("/users", get(list).post(create))
        .route("/users/{id}", get(show))
        .with_state(AppState { pool });

    let listener = tokio::net::TcpListener::bind("127.0.0.1:0").await?;
    axum::serve(listener, app).await?;
    Ok(())
}
```

A health check that actually touches the database:

```rust
async fn health(State(st): State<AppState>) -> impl IntoResponse {
    match sqlx::query_scalar::<_, i64>("SELECT 1").fetch_one(&st.pool).await {
        Ok(_) => (StatusCode::OK, Json(json!({
            "db": "up", "pool_size": st.pool.size(), "idle": st.pool.num_idle()
        }))),
        Err(e) => (StatusCode::SERVICE_UNAVAILABLE, Json(json!({
            "db": "down", "error": e.to_string()
        }))),
    }
}
```

```console
$ curl -s localhost:52340/health
{"db":"up","idle":0,"pool_size":1}
```

## `query` vs `query_as` vs `query!`

Three APIs, three trade-offs.

**`sqlx::query`** — runtime SQL, rows accessed by name or index:

```rust
use sqlx::Row;

let rows = sqlx::query("SELECT id, email FROM users ORDER BY id").fetch_all(&pool).await?;
for row in &rows {
    let id: i64 = row.get("id");
    let email: String = row.get(1);
    println!("id={id} email={email} columns={}", row.len());
}
```

Output:

```
id=1 email=raj@x.com columns=2
id=2 email=sam@x.com columns=2
```

`row.get` panics on a bad name or type; `row.try_get` returns a `Result`.

**`sqlx::query_as`** — runtime SQL, rows mapped into a `FromRow` type. The
default choice:

```rust
#[derive(Debug, FromRow)]
struct User {
    id: i64,
    email: String,
    display_name: Option<String>,
    created_at: DateTime<Utc>,
}

let u: User = sqlx::query_as("SELECT id, email, display_name, created_at FROM users WHERE id = ?")
    .bind(1_i64)
    .fetch_one(&pool)
    .await?;
println!("{:?}", u);
```

Output:

```
User { id: 1, email: "raj@x.com", display_name: Some("Raj"), created_at: 2026-10-25T17:09:44Z }
```

Tuples work too, with no struct at all:

```rust
let found: Vec<(i64, String)> = sqlx::query_as("SELECT id, email FROM users WHERE email = ?")
    .bind(email)
    .fetch_all(&pool)
    .await?;
```

**`sqlx::query!` / `query_as!`** — compile-time checked:

```rust
let user = sqlx::query!("SELECT id, email FROM users WHERE id = $1", id)
    .fetch_one(&pool)
    .await?;
println!("{} {}", user.id, user.email);   // fields generated from the columns
```

Notice three things: parameters are **arguments**, not `.bind()` calls; there is
no `User` struct, the macro generates an anonymous one; and a typo in a column
name is a **compile error**.

| | `query` | `query_as` | `query!` |
|---|---|---|---|
| SQL checked | no | no | yes, at compile time |
| Types checked | no | partially (decode at runtime) | yes, against the schema |
| Needs a DB to compile | no | no | **yes** (or offline data) |
| Dynamic SQL | yes | yes | no — SQL must be a literal |
| Result type | `Row` | your struct | generated struct |

## The `DATABASE_URL` / offline-mode requirement

This is the single biggest source of confusion with SQLx, so here it is plainly.

`sqlx::query!` is a **procedural macro**. To check your SQL it must know the
schema, and the only way it learns the schema is by connecting to a real
database **while `cargo build` runs**. So:

- `DATABASE_URL` must be set at build time, pointing at a database whose schema
  is already migrated.
- A `.env` file in the crate root is read by the macro, which is why SQLx
  projects have one.
- No database reachable at build time → compile error, even though your code is
  perfectly valid:

```
error: error returned from database: relation "users" does not exist
```

or

```
error: set `DATABASE_URL` to use query macros
```

That breaks CI and Docker builds, which have no database. The fix is **offline
mode**: cache the schema metadata in the repo.

```bash
cargo install sqlx-cli --no-default-features --features postgres
cargo sqlx prepare           # writes .sqlx/ — commit this directory
```

With `.sqlx/` committed, `cargo build` uses the cached query metadata and needs
no database. Two rules follow:

1. **Re-run `cargo sqlx prepare` whenever you change a query or a migration**,
   and commit the result. Stale cache means a compile error that mentions a query
   you already fixed.
2. Add `cargo sqlx prepare --check` to CI so a forgotten `prepare` fails the
   build rather than the deploy.

**This lesson and lesson 17.9 use `query`/`query_as` throughout, not the
macros.** The runtime APIs need no build-time database and work with dynamic
SQL, which lesson 17.9's `QueryBuilder` section requires. Adopt the macros once
your schema and CI are stable — the compile-time checking is genuinely excellent,
and catching a renamed column at `cargo check` rather than in production is worth
the setup.

## `FromRow`

```rust
#[derive(Debug, FromRow)]
struct User {
    id: i64,
    email: String,
    display_name: Option<String>,
    created_at: DateTime<Utc>,
}
```

Matching is **by column name**, so the `SELECT` list must name every field.
`SELECT *` happens to work until someone reorders or adds a column — name your
columns.

Useful attributes:

```rust
#[derive(FromRow)]
#[sqlx(rename_all = "camelCase")]
struct Row {
    #[sqlx(rename = "user_id")]
    id: i64,
    #[sqlx(default)]
    note: String,          // use Default if the column is absent
    #[sqlx(skip)]
    computed: u32,         // never read from a row
    #[sqlx(flatten)]
    address: Address,      // columns of a joined table, inline
    #[sqlx(json)]
    settings: Settings,    // a jsonb column deserialized via serde
}
```

`#[sqlx(flatten)]` is how lesson 17.9 maps a JOIN into nested structs.

## The fetch methods

| Method | Returns | Zero rows | Many rows |
|---|---|---|---|
| `fetch_one` | `T` | `Err(RowNotFound)` | first row |
| `fetch_optional` | `Option<T>` | `Ok(None)` | first row |
| `fetch_all` | `Vec<T>` | `Ok(vec![])` | all, buffered |
| `fetch` | `impl Stream` | empty stream | streamed, not buffered |
| `execute` | `QueryResult` | n/a | no rows, just counts |
| `query_scalar(..).fetch_one` | first column of first row | `Err(RowNotFound)` | — |

```rust
let all: Vec<User> = sqlx::query_as("SELECT id, email, display_name, created_at FROM users ORDER BY id")
    .fetch_all(&pool)
    .await?;
for u in &all { println!("{} {} {:?}", u.id, u.email, u.display_name); }
```

Output:

```
1 raj@x.com Some("Raj")
2 sam@x.com None
```

`fetch_optional` is what you want for a lookup by id:

```rust
let missing: Option<User> = sqlx::query_as("SELECT id, email, display_name, created_at FROM users WHERE id = ?")
    .bind(99_i64)
    .fetch_optional(&pool)
    .await?;
println!("id 99 -> {:?}", missing.map(|u| u.email));
```

Output:

```
id 99 -> None
```

`fetch_one` on no rows is an error, and that error is why lesson 17.6 insists on
translating it:

```rust
let e = sqlx::query_as::<_, User>("SELECT ... WHERE id = ?")
    .bind(99_i64)
    .fetch_one(&pool)
    .await
    .unwrap_err();
println!("{:?} / {}", e, e);
println!("is RowNotFound: {}", matches!(e, sqlx::Error::RowNotFound));
```

Output:

```
RowNotFound / no rows returned by a query that expected to return at least one row
is RowNotFound: true
```

`execute` for writes:

```rust
let r = sqlx::query("INSERT INTO users (email, display_name) VALUES (?, ?)")
    .bind("raj@x.com")
    .bind("Raj")
    .execute(&pool)
    .await?;
println!("insert: rows_affected={} last_insert_rowid={}", r.rows_affected(), r.last_insert_rowid());
```

Output:

```
insert: rows_affected=1 last_insert_rowid=1
```

`last_insert_rowid()` is SQLite-only. On Postgres use `INSERT ... RETURNING id`
with `fetch_one`, which lesson 17.9 covers and the handler below demonstrates.

And a scalar:

```rust
let n: i64 = sqlx::query_scalar("SELECT COUNT(*) FROM users").fetch_one(&pool).await?;
println!("count={n}");
```

Output:

```
count=2
```

`fetch` returns a `Stream` and is the one to use for a large export —
`fetch_all` on a million rows buffers a million rows in memory.

## Binding and SQL injection

`.bind()` sends the value as a **protocol-level parameter**, separate from the
SQL text. The database never parses it as SQL, so injection is structurally
impossible — not escaped, just not in the same channel.

```rust
let evil = "raj@x.com' OR '1'='1";
let found: Vec<(i64, String)> = sqlx::query_as("SELECT id, email FROM users WHERE email = ?")
    .bind(evil)
    .fetch_all(&pool)
    .await?;
println!("rows for {evil:?}: {}", found.len());
```

Output:

```
rows for "raj@x.com' OR '1'='1": 0
```

Zero rows, because it searched for a user whose email is literally that string.

The only way to reintroduce injection is to build the SQL string yourself:

```rust
// NEVER do this
let sql = format!("SELECT * FROM users WHERE email = '{}'", user_input);
```

When a query's **shape** must be dynamic, use `QueryBuilder` (lesson 17.9),
which keeps bindings as bindings. Placeholders are `$1, $2` on Postgres and `?`
on SQLite/MySQL; bind order follows the numbering on Postgres and the `?` order
on SQLite.

## Type mapping

| Postgres | Rust | Notes |
|---|---|---|
| `smallint` / `int2` | `i16` | |
| `integer` / `int4` | `i32` | |
| `bigint` / `int8` | `i64` | `BIGSERIAL` id → `i64` |
| `real` / `double precision` | `f32` / `f64` | |
| `numeric` | `rust_decimal::Decimal` or `bigdecimal::BigDecimal` | needs the `rust_decimal`/`bigdecimal` feature; **use this for money, never `f64`** |
| `boolean` | `bool` | |
| `text` / `varchar` / `char` | `String`, `&str` | |
| `bytea` | `Vec<u8>`, `&[u8]` | |
| `uuid` | `uuid::Uuid` | `uuid` feature |
| `timestamptz` | `chrono::DateTime<Utc>` | `chrono` feature |
| `timestamp` | `chrono::NaiveDateTime` | no zone — prefer `timestamptz` |
| `date` / `time` | `NaiveDate` / `NaiveTime` | |
| `interval` | `sqlx::postgres::types::PgInterval` | |
| `json` / `jsonb` | `serde_json::Value` or `sqlx::types::Json<T>` | `json` feature |
| `int4[]` / `text[]` | `Vec<i32>` / `Vec<String>` | Postgres only |
| `inet` / `cidr` | `std::net::IpAddr` | `ipnetwork` feature |
| enum type | your enum + `#[derive(sqlx::Type)]` | |
| anything nullable | `Option<T>` | |

Two gotchas. First, integer width is checked: a `BIGSERIAL` column read into
`i32` is a decode error, not a silent cast:

```rust
let e = sqlx::query_as::<_, (i64, i64)>("SELECT id, email FROM users").fetch_all(&pool).await.unwrap_err();
println!("{}", e);
```

Output:

```
error occurred while decoding column 1: mismatched types; Rust type `i64` (as SQL type `INTEGER`) is not compatible with SQL type `TEXT`
```

Second, Postgres has no unsigned integers. A `u32` id does not round-trip; use
`i32`/`i64` in row structs and convert at the API boundary if you must.

A Postgres enum maps with one derive:

```rust
#[derive(sqlx::Type, Debug)]
#[sqlx(type_name = "user_role", rename_all = "lowercase")]
enum Role { Admin, Member, Guest }
```

## `NULL` → `Option`

A nullable column must be `Option<T>` in the row struct. On Postgres, decoding
`NULL` into a non-`Option` field is an error:

```
error occurred while decoding column 1: unexpected null; try decoding as an `Option`
```

SQLite is more permissive because of its dynamic typing — it coerced `NULL` into
an empty `String` here:

```rust
let r = sqlx::query_as::<_, (i64, String)>("SELECT id, display_name FROM users WHERE id = 2")
    .fetch_one(&pool).await;
println!("NULL -> String on sqlite: {:?}", r);

let ok: (i64, Option<String>) = sqlx::query_as("SELECT id, display_name FROM users WHERE id = 2")
    .fetch_one(&pool).await?;
println!("as Option: {:?}", ok);
```

Output:

```
NULL -> String on sqlite: Ok((2, ""))
as Option: (2, None)
```

That silent coercion is a reason to develop against the database you deploy to.
Either way, model nullable columns as `Option<T>` and the behaviour is correct
on both.

Binding a `None` writes a real `NULL`, with a turbofish to pin the type:

```rust
sqlx::query("INSERT INTO users (email, display_name) VALUES (?, ?)")
    .bind("sam@x.com")
    .bind(None::<String>)
    .execute(&pool)
    .await?;
```

## Database errors

```rust
let e = sqlx::query("INSERT INTO users (email) VALUES (?)")
    .bind("raj@x.com")
    .execute(&pool)
    .await
    .unwrap_err();
if let sqlx::Error::Database(db) = &e {
    println!("code={:?} message={}", db.code(), db.message());
    println!("is_unique_violation={}", db.is_unique_violation());
}
```

Output:

```
code=Some("2067") message=UNIQUE constraint failed: users.email
is_unique_violation=true
```

`2067` is SQLite's code; Postgres returns `23505`. `is_unique_violation()`,
`is_foreign_key_violation()`, and `is_check_violation()` are the portable
helpers — prefer them over hardcoded SQLSTATE strings.

The `sqlx::Error` variants worth matching on: `RowNotFound`,
`Database(DatabaseError)`, `PoolTimedOut`, `PoolClosed`, `ColumnDecode`,
`TypeNotFound`, `Io`.

## Handlers end to end

```rust
async fn show(State(st): State<AppState>, Path(id): Path<i64>) -> impl IntoResponse {
    match sqlx::query_as::<_, User>("SELECT id, email, display_name FROM users WHERE id = ?")
        .bind(id)
        .fetch_optional(&st.pool)
        .await
    {
        Ok(Some(u)) => (StatusCode::OK, Json(json!(u))).into_response(),
        Ok(None) => (StatusCode::NOT_FOUND, Json(json!({"error":"not found","id":id}))).into_response(),
        Err(e) => (StatusCode::INTERNAL_SERVER_ERROR, Json(json!({"error":e.to_string()}))).into_response(),
    }
}

async fn create(State(st): State<AppState>, Json(b): Json<NewUser>) -> impl IntoResponse {
    let res = sqlx::query_as::<_, User>(
        "INSERT INTO users (email, display_name) VALUES (?, ?) RETURNING id, email, display_name",
    )
    .bind(&b.email)
    .bind(&b.display_name)
    .fetch_one(&st.pool)
    .await;

    match res {
        Ok(u) => (StatusCode::CREATED, Json(json!(u))).into_response(),
        Err(sqlx::Error::Database(db)) if db.is_unique_violation() =>
            (StatusCode::CONFLICT, Json(json!({"error":"email taken","detail":db.message()}))).into_response(),
        Err(e) => (StatusCode::INTERNAL_SERVER_ERROR, Json(json!({"error":e.to_string()}))).into_response(),
    }
}
```

```console
$ curl -si -X POST localhost:52340/users -H 'content-type: application/json' \
    -d '{"email":"raj@x.com","display_name":"Raj"}' | sed -n '1p;$p'
HTTP/1.1 201 Created
{"display_name":"Raj","email":"raj@x.com","id":1}

$ curl -s -X POST localhost:52340/users -H 'content-type: application/json' -d '{"email":"sam@x.com"}'
{"display_name":null,"email":"sam@x.com","id":2}

$ curl -si -X POST localhost:52340/users -H 'content-type: application/json' \
    -d '{"email":"raj@x.com"}' | sed -n '1p;$p'
HTTP/1.1 409 Conflict
{"detail":"UNIQUE constraint failed: users.email","error":"email taken"}

$ curl -s localhost:52340/users
[{"id":1,"email":"raj@x.com","display_name":"Raj"},{"id":2,"email":"sam@x.com","display_name":null}]

$ curl -si localhost:52340/users/99 | sed -n '1p;$p'
HTTP/1.1 404 Not Found
{"error":"not found","id":99}
```

In a real app those `match`es collapse to `?` with the `AppError` from lesson
17.6. The point here is to show every branch explicitly.

`RETURNING` is doing real work: one round trip inserts the row and gives back
the generated id and defaults. It is standard on Postgres and supported by
SQLite 3.35+.

## Quick Reference

| Need | Form |
|---|---|
| Pool | `PgPoolOptions::new().max_connections(n).connect(&url).await?` |
| Clone for state | `#[derive(Clone)] struct AppState { pool: PgPool }` |
| Runtime SQL, row access | `sqlx::query(sql).bind(v).fetch_all(&pool)` + `row.get("c")` |
| Runtime SQL, into struct | `sqlx::query_as::<_, T>(sql).bind(v).fetch_one(&pool)` |
| Compile-checked | `sqlx::query!("... $1", v)` — needs `DATABASE_URL` or `.sqlx/` |
| Offline cache | `cargo sqlx prepare`, commit `.sqlx/`, CI `--check` |
| Exactly one row | `.fetch_one()` → `Err(RowNotFound)` if none |
| Zero or one | `.fetch_optional()` → `Option<T>` |
| Many | `.fetch_all()` → `Vec<T>` |
| Huge result | `.fetch()` → `Stream` |
| Write | `.execute()` → `.rows_affected()` |
| Single value | `query_scalar(sql).fetch_one()` |
| Insert and read back | `INSERT ... RETURNING ...` + `fetch_one` |
| Row mapping | `#[derive(FromRow)]`, by column name |
| Nullable column | `Option<T>` |
| Bind a NULL | `.bind(None::<String>)` |
| Unique violation | `sqlx::Error::Database(db) if db.is_unique_violation()` |
| Health check | `query_scalar::<_, i64>("SELECT 1").fetch_one(&pool)` |

## Common Mistakes

**Expecting `query!` to compile without a database.** It is a macro that talks
to Postgres at build time. Commit `.sqlx/` via `cargo sqlx prepare`, or use
`query_as`.

**Forgetting to re-run `cargo sqlx prepare` after editing a query.** The stale
cache produces a compile error about SQL you have already fixed.

**Wrapping the pool in `Arc<Mutex<...>>`.** It is already shared and concurrent;
a mutex serializes your whole database layer.

**`fetch_one` for a lookup by id.** A missing row becomes a 500. Use
`fetch_optional`.

**`f64` for money.** `numeric` → `Decimal`. Floating-point rounding on currency
is a bug you will be asked to explain.

**`u32`/`u64` in row structs.** Postgres has no unsigned integers.

**`max_connections` × replicas over the server's limit.** New connections get
refused and the outage looks random.

**Mixing `?` and `$1` placeholders.** Postgres wants `$1`; SQLite and MySQL want
`?`.

## Exercises

1. Build an in-memory SQLite pool, create a `posts` table, insert two rows, and
   read them back with `query_as` into a `FromRow` struct.
2. Add a nullable `published_at` column. Read it as `DateTime<Utc>` and then as
   `Option<DateTime<Utc>>`, and record what each does.
3. Write `find_post(pool, id) -> Result<Option<Post>, sqlx::Error>` with
   `fetch_optional`, and a second version with `fetch_one` that maps
   `RowNotFound` to `Ok(None)`. Decide which you prefer.
4. Add a `UNIQUE` constraint on `posts.slug`, insert a duplicate, and match the
   error with `db.is_unique_violation()` to return a 409.
5. Wire the pool into an axum `AppState` with `GET /posts`, `GET /posts/{id}`,
   and `POST /posts` using `INSERT ... RETURNING`. Add a `/health` endpoint that
   runs `SELECT 1` and reports `pool.size()`.

---

[← Course Index](../README.md)
