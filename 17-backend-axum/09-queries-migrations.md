[← Course Index](../README.md)

# 17.9 — Queries, Transactions, and Migrations

## What You'll Learn

- CRUD end to end: `INSERT ... RETURNING`, `rows_affected`, bulk insert
- `QueryBuilder` for dynamic filters, LIMIT/OFFSET vs keyset pagination
- `pool.begin()` / `commit()` / `rollback()`, and drop-rollback
- `sqlx-cli` migrations, `sqlx::migrate!`, N+1 and JOIN-to-nested-struct

## Coming From Another Language

| Other | SQLx |
|---|---|
| Prisma `prisma migrate dev` | `sqlx migrate add` + `sqlx migrate run` |
| Knex `.where(...)` chained builder | `QueryBuilder` + `push` / `push_bind` |
| `BEGIN` / `COMMIT` by hand | `pool.begin()`, `tx.commit()`, drop = rollback |
| Rails `find_each` / Django `.iterator()` | `.fetch()` returning a `Stream` |

The one genuinely different thing is drop-rollback: a `Transaction` that goes out
of scope without `commit()` rolls back, so `?` and early returns need no cleanup.

Outputs on this page were captured against **SQLite via SQLx** — no Postgres on
this machine. Placeholders are `?` here and `$1`, `$2` on Postgres; everything
else transfers unchanged. Postgres-only behaviour is marked inline.

## The schema used throughout

```rust
sqlx::query("CREATE TABLE authors (id INTEGER PRIMARY KEY, name TEXT NOT NULL)").execute(pool).await?;
sqlx::query(
    "CREATE TABLE posts (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        author_id INTEGER NOT NULL REFERENCES authors(id),
        title TEXT NOT NULL,
        views INTEGER NOT NULL DEFAULT 0
    )").execute(pool).await?;
sqlx::query("CREATE TABLE balances (id INTEGER PRIMARY KEY, cents INTEGER NOT NULL)").execute(pool).await?;
```

```rust
#[derive(Debug, FromRow)]
struct Post { id: i64, author_id: i64, title: String, views: i64 }
```

## `INSERT ... RETURNING`

One round trip to insert and read back the generated id and defaults.

```rust
let p: Post = sqlx::query_as(
    "INSERT INTO posts (author_id, title) VALUES (?, ?) RETURNING id, author_id, title, views")
    .bind(1_i64)
    .bind("Rust is fun")
    .fetch_one(&pool)
    .await?;
println!("{:?}", p);
```

Output:

```
Post { id: 1, author_id: 1, title: "Rust is fun", views: 0 }
```

Standard on Postgres, supported by SQLite 3.35+. MySQL has no `RETURNING`, so
there you need `last_insert_id()` and a second `SELECT`. Always prefer
`RETURNING` where it exists — the alternative is racy under concurrency.

## Bulk insert with `QueryBuilder`

```rust
use sqlx::{QueryBuilder, Sqlite};

let mut qb: QueryBuilder<Sqlite> = QueryBuilder::new("INSERT INTO posts (author_id, title, views) ");
qb.push_values(
    [(1_i64, "Async in depth", 90_i64), (2, "Ownership", 300), (2, "Traits", 12)],
    |mut b, (a, t, v)| { b.push_bind(a).push_bind(t).push_bind(v); },
);
let r = qb.build().execute(&pool).await?;
println!("inserted {} rows", r.rows_affected());
```

Output:

```
inserted 3 rows
```

One statement, one round trip, every value still a real binding. The alternative
— a loop of single inserts — is N round trips; for a thousand rows that is the
difference between 20ms and 2 seconds. Chunk at a few thousand rows: Postgres
caps a statement at 65535 parameters.

Postgres has a faster path still, `COPY`, via `pool.copy_in_raw(...)` — worth it
above roughly 100k rows.

## `UPDATE` and `DELETE` with `rows_affected`

```rust
let r = sqlx::query("UPDATE posts SET views = views + 1 WHERE author_id = ?")
    .bind(2_i64).execute(&pool).await?;
println!("updated {}", r.rows_affected());

let r = sqlx::query("UPDATE posts SET views = 0 WHERE author_id = ?")
    .bind(99_i64).execute(&pool).await?;
println!("updated {} (no match)", r.rows_affected());
```

Output:

```
updated 2
updated 0 (no match)
```

`rows_affected() == 0` is how you detect "nothing matched" on a write — the
query itself is a perfectly good success. That is the 404 check for a PATCH or
DELETE:

```rust
let r = sqlx::query("DELETE FROM posts WHERE id = ?").bind(4_i64).execute(&pool).await?;
println!("deleted {}", r.rows_affected());
let r = sqlx::query("DELETE FROM posts WHERE id = ?").bind(4_i64).execute(&pool).await?;
println!("deleted {} (already gone)", r.rows_affected());
```

Output:

```
deleted 1
deleted 0 (already gone)
```

In a handler:

```rust
if r.rows_affected() == 0 {
    return Err(AppError::NotFound("post", id.to_string()));
}
Ok(StatusCode::NO_CONTENT)
```

`UPDATE ... RETURNING` with `fetch_optional` collapses the check and the fetch
into one query:

```rust
let updated: Option<Post> = sqlx::query_as(
    "UPDATE posts SET title = ? WHERE id = ? RETURNING id, author_id, title, views")
    .bind("Rust is great").bind(1_i64)
    .fetch_optional(&pool).await?;
println!("{:?}", updated);
```

Output:

```
Some(Post { id: 1, author_id: 1, title: "Rust is great", views: 0 })
```

`None` means no such row — exactly the 404 signal, with the updated row for free
on success. This is the idiomatic shape for a PATCH handler.

## Dynamic filters

The naive approach — `format!` the WHERE clause — reintroduces SQL injection.
`QueryBuilder` keeps every value a binding.

```rust
let mut qb: QueryBuilder<Sqlite> =
    QueryBuilder::new("SELECT id, author_id, title, views FROM posts WHERE 1=1");
if let Some(a) = author { qb.push(" AND author_id = ").push_bind(a); }
if let Some(v) = min_views { qb.push(" AND views >= ").push_bind(v); }
if let Some(l) = like { qb.push(" AND title LIKE ").push_bind(format!("%{l}%")); }
qb.push(" ORDER BY id");

println!("sql: {:?}", qb.sql());
let rows: Vec<Post> = qb.build_query_as().fetch_all(&pool).await?;
println!("  -> {:?}", rows.iter().map(|p| p.id).collect::<Vec<_>>());
```

Four filter combinations:

```
sql: SqlStr(ArcString("SELECT id, author_id, title, views FROM posts WHERE 1=1 AND author_id = ? ORDER BY id"))
  -> [3]
sql: SqlStr(ArcString("SELECT id, author_id, title, views FROM posts WHERE 1=1 AND views >= ? ORDER BY id"))
  -> [2, 3]
sql: SqlStr(ArcString("SELECT id, author_id, title, views FROM posts WHERE 1=1 AND author_id = ? AND views >= ? AND title LIKE ? ORDER BY id"))
  -> [1]
sql: SqlStr(ArcString("SELECT id, author_id, title, views FROM posts WHERE 1=1 ORDER BY id"))
  -> [1, 2, 3]
```

Three rules:

- `WHERE 1=1` saves tracking whether you need `WHERE` or `AND`. It costs nothing;
  the planner removes it.
- `push` is **raw SQL** — never put user input there. `push_bind` is for values.
- `qb.sql()` prints the generated statement, which is how you debug these.

A shortcut for the single most common case: `IS NULL OR` in plain SQL.

```rust
sqlx::query_as::<_, Post>(
    "SELECT id, author_id, title, views FROM posts
     WHERE (? IS NULL OR author_id = ?)
       AND (? IS NULL OR views >= ?)
     ORDER BY id")
    .bind(author).bind(author)
    .bind(min_views).bind(min_views)
```

That keeps a static query — so the `query_as!` macro and the query planner's
prepared-statement cache both work — at the cost of repeating each parameter.
Use `QueryBuilder` when the filter set is genuinely open-ended.

## Pagination: LIMIT/OFFSET

```rust
for page in 1..=3_i64 {
    let per = 2_i64;
    let rows: Vec<(i64, String)> = sqlx::query_as(
        "SELECT id, title FROM posts ORDER BY id LIMIT ? OFFSET ?")
        .bind(per).bind((page - 1) * per)
        .fetch_all(&pool).await?;
    println!("page {page}: {:?}", rows);
}
let total: i64 = sqlx::query_scalar("SELECT COUNT(*) FROM posts").fetch_one(&pool).await?;
println!("total={total}");
```

Output:

```
page 1: [(1, "Rust is great"), (2, "Async in depth")]
page 2: [(3, "Ownership")]
page 3: []
total=3
```

Simple, and what a page-number UI needs. Two real problems: `OFFSET 100000`
makes the database scan and discard 100000 rows, and a row inserted between
page 1 and page 2 shifts everything, so the user sees an item twice or not at
all.

## Pagination: keyset (cursor)

```rust
let mut cursor = 0_i64;
loop {
    let rows: Vec<(i64, String)> = sqlx::query_as(
        "SELECT id, title FROM posts WHERE id > ? ORDER BY id LIMIT 2")
        .bind(cursor)
        .fetch_all(&pool).await?;
    if rows.is_empty() { break; }
    println!("after id={cursor}: {:?}", rows);
    cursor = rows.last().unwrap().0;
}
```

Output:

```
after id=0: [(1, "Rust is great"), (2, "Async in depth")]
after id=2: [(3, "Ownership")]
```

Constant time at any depth — with an index on the cursor column it is an index
seek, not a scan — and stable under concurrent inserts. The cost: no jumping to
page 50, and the cursor column must be unique and ordered. For a non-unique sort
key, make the cursor a tuple:

```sql
WHERE (created_at, id) < ($1, $2) ORDER BY created_at DESC, id DESC LIMIT $3
```

Return the cursor to the client as an opaque string (base64 the tuple) so you
can change the implementation later. Use keyset for infinite scroll and API
pagination; keep LIMIT/OFFSET for an admin table with page numbers and few rows.

## Transactions

```rust
let mut tx = pool.begin().await?;
sqlx::query("UPDATE balances SET cents = cents - 2500 WHERE id = 1").execute(&mut *tx).await?;
sqlx::query("UPDATE balances SET cents = cents + 2500 WHERE id = 2").execute(&mut *tx).await?;
tx.commit().await?;

let b: Vec<(i64, i64)> = sqlx::query_as("SELECT id, cents FROM balances ORDER BY id")
    .fetch_all(&pool).await?;
println!("after commit: {:?}", b);
```

Output:

```
after commit: [(1, 7500), (2, 3000)]
```

The `&mut *tx` is the one piece of syntax to memorize: `Transaction` derefs to a
connection, and the executor wants `&mut Connection`. Pass `&pool` instead and
the query runs **outside** the transaction — a bug with no compile error and no
visible symptom until a rollback fails to undo it.

Explicit rollback, with a read inside the transaction showing the uncommitted
value:

```rust
let mut tx = pool.begin().await?;
sqlx::query("UPDATE balances SET cents = 0 WHERE id = 1").execute(&mut *tx).await?;
let mid: i64 = sqlx::query_scalar("SELECT cents FROM balances WHERE id = 1")
    .fetch_one(&mut *tx).await?;
println!("inside tx: id 1 = {mid}");
tx.rollback().await?;
let after: i64 = sqlx::query_scalar("SELECT cents FROM balances WHERE id = 1")
    .fetch_one(&pool).await?;
println!("after rollback: id 1 = {after}");
```

Output:

```
inside tx: id 1 = 0
after rollback: id 1 = 7500
```

## Drop means rollback

A `Transaction` dropped without `commit()` rolls back. This is the feature that
makes `?` safe in a transactional function.

```rust
{
    let mut tx = pool.begin().await?;
    sqlx::query("UPDATE balances SET cents = 999999 WHERE id = 2").execute(&mut *tx).await?;
    println!("tx dropped without commit");
}
let after: i64 = sqlx::query_scalar("SELECT cents FROM balances WHERE id = 2")
    .fetch_one(&pool).await?;
println!("id 2 = {after}");
```

Output:

```
tx dropped without commit
id 2 = 3000
```

So an early return needs no cleanup:

```rust
async fn transfer(pool: &SqlitePool, from: i64, to: i64, cents: i64) -> Result<(), sqlx::Error> {
    let mut tx = pool.begin().await?;
    sqlx::query("UPDATE balances SET cents = cents - ? WHERE id = ?")
        .bind(cents).bind(from).execute(&mut *tx).await?;

    let bal: i64 = sqlx::query_scalar("SELECT cents FROM balances WHERE id = ?")
        .bind(from).fetch_one(&mut *tx).await?;
    if bal < 0 {
        return Err(sqlx::Error::Protocol("insufficient funds".into()));
    }

    sqlx::query("UPDATE balances SET cents = cents + ? WHERE id = ?")
        .bind(cents).bind(to).execute(&mut *tx).await?;
    tx.commit().await?;
    Ok(())
}
```

```rust
println!("transfer 100000: {:?}", transfer(&pool, 1, 2, 100000).await.map_err(|e| e.to_string()));
let b: Vec<(i64, i64)> = sqlx::query_as("SELECT id, cents FROM balances ORDER BY id").fetch_all(&pool).await?;
println!("balances unchanged: {:?}", b);
println!("transfer 1000: {:?}", transfer(&pool, 1, 2, 1000).await.map_err(|e| e.to_string()));
let b: Vec<(i64, i64)> = sqlx::query_as("SELECT id, cents FROM balances ORDER BY id").fetch_all(&pool).await?;
println!("balances: {:?}", b);
```

Output:

```
transfer 100000: Err("encountered unexpected or invalid data: insufficient funds")
balances unchanged: [(1, 7500), (2, 3000)]
transfer 1000: Ok(())
balances: [(1, 6500), (2, 4000)]
```

The failed transfer debited nothing. Neither the `?` on a query error nor the
explicit `return Err` needed a rollback call.

This is also why lesson 17.7's `TimeoutLayer` warning matters: a cancelled
handler drops its transaction, which rolls back. That is the correct outcome,
and only reachable if the writes were in a transaction.

## Savepoints

`begin()` on a transaction creates a nested savepoint (needs `use sqlx::Acquire`).

```rust
use sqlx::Acquire;

let mut tx = pool.begin().await?;
sqlx::query("UPDATE balances SET cents = 1 WHERE id = 1").execute(&mut *tx).await?;
{
    let mut sp = tx.begin().await?;
    sqlx::query("UPDATE balances SET cents = 2 WHERE id = 2").execute(&mut *sp).await?;
    sp.rollback().await?;
}
tx.commit().await?;

let b: Vec<(i64, i64)> = sqlx::query_as("SELECT id, cents FROM balances ORDER BY id")
    .fetch_all(&pool).await?;
println!("after savepoint rollback + outer commit: {:?}", b);
```

Output:

```
after savepoint rollback + outer commit: [(1, 1), (2, 4000)]
```

The outer write committed; the savepoint's write did not. Useful for "try this
one step, and if it fails keep the rest" — a bulk import that skips bad rows.
Rarely needed otherwise.

## Isolation levels

Postgres defaults to `READ COMMITTED`, which is right most of the time. For
read-modify-write on a row other requests also touch, either raise the level or
lock explicitly:

```rust
let mut tx = pool.begin().await?;
sqlx::query("SELECT cents FROM balances WHERE id = $1 FOR UPDATE")
    .bind(1_i64).fetch_one(&mut *tx).await?;
```

`FOR UPDATE` locks the row until commit, serializing concurrent transfers.
Alternatively do the arithmetic in SQL — `SET cents = cents - $1` is atomic in a
single statement and needs no lock, which is why the `transfer` above is written
that way.

## Migrations with `sqlx-cli`

```bash
cargo install sqlx-cli --no-default-features --features postgres,sqlite
export DATABASE_URL=postgres://localhost/myapp

sqlx database create
sqlx migrate add create_notes        # writes migrations/<timestamp>_create_notes.sql
sqlx migrate add -r add_pinned       # -r for a reversible pair (.up.sql / .down.sql)
sqlx migrate run
sqlx migrate revert                  # only works for reversible migrations
sqlx migrate info
```

`migrations/20261025000001_create_notes.sql`:

```sql
CREATE TABLE notes (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    body TEXT NOT NULL
);
```

`migrations/20261025000002_add_pinned.sql`:

```sql
ALTER TABLE notes ADD COLUMN pinned INTEGER NOT NULL DEFAULT 0;
```

## Embedding migrations with `sqlx::migrate!`

```bash
cargo add sqlx --features migrate
```

The macro reads `./migrations` **at compile time** and embeds the SQL in the
binary, so a deployed container needs no migration files and no `sqlx` CLI.

```rust
#[tokio::main]
async fn main() -> Result<(), sqlx::Error> {
    let pool = SqlitePoolOptions::new().max_connections(1).connect("sqlite::memory:").await?;
    sqlx::migrate!("./migrations").run(&pool).await?;
    println!("migrations applied");

    let rows = sqlx::query(
        "SELECT version, description, success, checksum IS NOT NULL AS has_ck
         FROM _sqlx_migrations ORDER BY version")
        .fetch_all(&pool).await?;
    for r in &rows {
        println!("{} | {} | success={} | checksum={}",
            r.get::<i64, _>("version"), r.get::<String, _>("description"),
            r.get::<bool, _>("success"), r.get::<bool, _>("has_ck"));
    }

    sqlx::query("INSERT INTO notes (body, pinned) VALUES (?, ?)").bind("hi").bind(1_i64)
        .execute(&pool).await?;
    let n: Vec<(i64, String, i64)> = sqlx::query_as("SELECT id, body, pinned FROM notes")
        .fetch_all(&pool).await?;
    println!("{:?}", n);

    sqlx::migrate!("./migrations").run(&pool).await?;
    println!("second run is a no-op");
    Ok(())
}
```

Output:

```
migrations applied
20261025000001 | create notes | success=true | checksum=true
20261025000002 | add pinned | success=true | checksum=true
[(1, "hi", 1)]
second run is a no-op
```

`sqlx::migrate!()` with no argument defaults to `./migrations`.

## The `_sqlx_migrations` table

SQLx creates it automatically:

| Column | Meaning |
|---|---|
| `version` | the filename timestamp, the primary key |
| `description` | the filename's slug, underscores to spaces |
| `installed_on` | when it ran |
| `success` | false if it failed partway |
| `checksum` | SHA-384 of the file contents |
| `execution_time` | nanoseconds |

The `checksum` is why **you never edit an applied migration**. Change the file
and the next `run` fails:

```
error: migration 20261025000001 was previously applied but has been modified
```

Fix a mistake with a new migration. That error is SQLx protecting you from a
schema that differs between environments.

Each migration runs in its own transaction where the database allows it. Postgres
mostly does; some DDL (`CREATE INDEX CONCURRENTLY`) cannot run in one, and SQLx
has `-- no-transaction` as the first line of the file for that case.

## Migration best practices

- **Run migrations at startup, before binding the listener** (lesson 17.5's
  initialization order). Fail the deploy, not the requests.
- **Additive changes only during a rolling deploy.** Old and new code run at the
  same time, so adding a nullable column or a new table is safe; dropping or
  renaming one is not. Rename as: add new column → backfill → write both →
  switch reads → stop writing old → drop in a later release.
- **Always give a new `NOT NULL` column a `DEFAULT`**, or the migration fails on
  a non-empty table.
- **Backfill in batches, not one `UPDATE`.** A single statement over a million
  rows holds locks long enough to cause an outage.
- **Create indexes concurrently on Postgres** (`CREATE INDEX CONCURRENTLY`, with
  `-- no-transaction`). A plain `CREATE INDEX` locks writes on the table.
- **Commit `.sqlx/` alongside the migration** when using the `query!` macros, so
  CI can build (lesson 17.8).

## N+1 and JOINs

The classic mistake — one query for the list, then one per item:

```rust
let authors: Vec<Author> = sqlx::query_as("SELECT id, name FROM authors ORDER BY id")
    .fetch_all(&pool).await?;
let mut queries = 1;
for a in &authors {
    let n: i64 = sqlx::query_scalar("SELECT COUNT(*) FROM posts WHERE author_id = ?")
        .bind(a.id).fetch_one(&pool).await?;
    queries += 1;
    println!("  {} has {} posts", a.name, n);
}
println!("N+1 used {queries} queries");
```

One query instead:

```rust
let rows = sqlx::query(
    "SELECT a.name, COUNT(p.id) AS n FROM authors a
     LEFT JOIN posts p ON p.author_id = a.id
     GROUP BY a.id ORDER BY a.id")
    .fetch_all(&pool).await?;
for r in &rows {
    println!("  {} has {} posts", r.get::<String, _>("name"), r.get::<i64, _>("n"));
}
println!("JOIN used 1 query");
```

Output:

```
  Raj has 2 posts
  Sam has 1 posts
N+1 used 3 queries
  Raj has 2 posts
  Sam has 1 posts
JOIN used 1 query
```

Three queries for two authors; three thousand for a thousand. `LEFT JOIN`, not
`JOIN`, so an author with no posts still appears with `0`.

When a JOIN would duplicate the parent row across children, use two queries and
group in Rust instead:

```rust
let posts: Vec<Post> = sqlx::query_as(
    "SELECT id, author_id, title, views FROM posts WHERE author_id = ANY($1)")
    .bind(&author_ids)
    .fetch_all(&pool).await?;

let mut by_author: HashMap<i64, Vec<Post>> = HashMap::new();
for p in posts { by_author.entry(p.author_id).or_default().push(p); }
```

Two queries regardless of how many authors — the standard "dataloader" shape.
`= ANY($1)` with a `Vec` binding is Postgres-only; on SQLite build an `IN (?,?,?)`
list with `QueryBuilder`.

## Mapping a JOIN into nested structs

`#[sqlx(flatten)]` fills a nested struct from the same row.

```rust
#[derive(Debug, FromRow)]
struct AuthorRef {
    #[sqlx(rename = "author_id")] id: i64,
    #[sqlx(rename = "author_name")] name: String,
}

#[derive(Debug, FromRow)]
struct PostWithAuthor {
    id: i64,
    title: String,
    #[sqlx(flatten)]
    author: AuthorRef,
}

let joined: Vec<PostWithAuthor> = sqlx::query_as(
    "SELECT p.id, p.title, a.id AS author_id, a.name AS author_name
     FROM posts p JOIN authors a ON a.id = p.author_id ORDER BY p.id")
    .fetch_all(&pool).await?;
for j in &joined {
    println!("post {} {:?} by author {} ({})", j.id, j.title, j.author.id, j.author.name);
}
```

Output:

```
post 1 "Rust is great" by author 1 (Raj)
post 2 "Async in depth" by author 1 (Raj)
post 3 "Ownership" by author 2 (Sam)
```

The aliases are load-bearing. `FromRow` matches by **column name**, and both
tables have an `id`, so `SELECT p.id, a.id` gives two columns named `id` and the
flattened struct silently picks the wrong one. Alias every joined column.

## Quick Reference

| Need | Form |
|---|---|
| Insert + read back | `INSERT ... RETURNING ...` + `fetch_one` |
| Bulk insert | `QueryBuilder::new("INSERT ...").push_values(iter, f)` |
| Did the write match? | `result.rows_affected()` |
| Update or 404 | `UPDATE ... RETURNING ...` + `fetch_optional` |
| Dynamic filters | `QueryBuilder` + `push` (SQL) / `push_bind` (values) |
| Inspect built SQL | `qb.sql()` |
| Page by number | `LIMIT ? OFFSET ?` |
| Page by cursor | `WHERE id > ? ORDER BY id LIMIT ?` |
| Begin | `let mut tx = pool.begin().await?;` |
| Run inside | `.execute(&mut *tx)` — **not** `&pool` |
| Finish | `tx.commit().await?` / `tx.rollback().await?` |
| Implicit rollback | drop the `Transaction` |
| Savepoint | `use sqlx::Acquire;` then `tx.begin()` |
| Row lock | `SELECT ... FOR UPDATE` inside a transaction |
| New migration | `sqlx migrate add [-r] name` |
| Apply | `sqlx migrate run` or `sqlx::migrate!().run(&pool)` |
| State table | `_sqlx_migrations` (version, checksum, success) |
| Avoid N+1 | one JOIN, or a second `IN (...)` query grouped in Rust |
| JOIN to nested struct | `#[sqlx(flatten)]` + aliased columns |

## Common Mistakes

**`.execute(&pool)` inside a transaction block.** The statement runs outside the
transaction and is not rolled back. Use `&mut *tx`.

**Editing an applied migration.** The checksum check fails on the next run. Add
a new migration.

**`NOT NULL` with no `DEFAULT` on an existing table.** The migration fails.

**`format!` into a WHERE clause.** That is SQL injection. `QueryBuilder` with
`push_bind`, or `($1 IS NULL OR col = $1)`.

**Deep `OFFSET`.** `OFFSET 500000` scans and throws away 500000 rows. Use keyset.

**A query inside a loop over rows.** That is N+1. Count the queries in your logs.

**`SELECT p.id, a.id` with `#[sqlx(flatten)]`.** Duplicate column names; the
wrong one wins silently. Alias them.

**Forgetting `use sqlx::Acquire` for savepoints.** The error says `begin` is an
associated function, not a method, which does not obviously mean "import a
trait".

## Exercises

1. Create `authors` and `books`, insert an author with `INSERT ... RETURNING`,
   then bulk-insert three books with `QueryBuilder::push_values`.
2. Write `delete_book(pool, id) -> Result<bool, sqlx::Error>` returning whether
   anything was deleted, using `rows_affected`. Then rewrite it with
   `DELETE ... RETURNING id` and `fetch_optional`.
3. Build a `list_books` function taking `Option<i64>` author, `Option<String>`
   title substring, and a `limit`, assembled with `QueryBuilder`. Print
   `qb.sql()` for three different combinations.
4. Write `transfer(pool, from, to, cents)` in a transaction that errors when the
   source would go negative. Call it once so it fails and once so it succeeds,
   printing balances after each.
5. Add two migration files, run them with `sqlx::migrate!`, print
   `_sqlx_migrations`, then edit the first file and run again to see the checksum
   error.
6. Write the same "authors with post counts" report twice — N+1 and one
   `LEFT JOIN ... GROUP BY` — and count the queries each issues.

---

[← Course Index](../README.md)
