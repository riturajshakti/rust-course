[← Course Index](../README.md)

# 18.2 — Logging and Tracing

## What You'll Learn

- Why `tracing` beats `log` for async services: spans survive `.await`
- `tracing_subscriber::fmt()`, `EnvFilter`, and `RUST_LOG` syntax
- Events with structured fields, and `#[instrument]` on handlers
- JSON output, request ids, and `TraceLayer` from `tower-http`

## Coming From Another Language

| Other | Rust |
|---|---|
| `console.log` / `print` | `info!("...")` — never use `println!` in a service |
| `winston` / `pino` structured fields | `info!(user_id = 7, "msg")` |
| `logging.getLogger(__name__)` | targets are module paths, automatic |
| `LOG_LEVEL=debug` | `RUST_LOG=debug` (with per-module directives) |

Two crates exist. `log` is the older facade — level plus a string. `tracing` adds
**spans**: a region of work that every event inside it inherits context from.
For an async service, that distinction is the whole game.

## Why spans matter in async

With plain log lines, two concurrent requests interleave and you cannot tell
which line belongs to which. A span attaches to the *future*, not the thread, so
it follows the task across every `.await` and suspension.

```rust
use tracing::{info, instrument};

#[instrument(fields(request_id = %format!("req-{id}")))]
async fn handle(id: u32, delay: u64) {
    info!("received");
    tokio::time::sleep(std::time::Duration::from_millis(delay)).await;
    info!("db done");
    tokio::time::sleep(std::time::Duration::from_millis(delay)).await;
    info!("responded");
}

#[tokio::main]
async fn main() {
    tracing_subscriber::fmt()
        .with_env_filter(EnvFilter::from_default_env())
        .with_ansi(false)
        .without_time()
        .init();

    let a = tokio::spawn(handle(1, 30));
    let b = tokio::spawn(handle(2, 10));
    let _ = tokio::join!(a, b);
}
```

Output with `RUST_LOG=info`:

```
 INFO handle{id=1 delay=30 request_id=req-1}: t4: received
 INFO handle{id=2 delay=10 request_id=req-2}: t4: received
 INFO handle{id=2 delay=10 request_id=req-2}: t4: db done
 INFO handle{id=2 delay=10 request_id=req-2}: t4: responded
 INFO handle{id=1 delay=30 request_id=req-1}: t4: db done
 INFO handle{id=1 delay=30 request_id=req-1}: t4: responded
```

The lines are completely out of order — request 2 started second and finished
first — yet every line carries its own `request_id`. Grep one id and you get
exactly one request's story. That is the reason to use `tracing`.

A thread-local in a `log`-based setup cannot do this: the task moves between
worker threads at every `.await`.

## Setup

```toml
tracing = { version = "0.1.44", features = ["attributes"] }
tracing-subscriber = { version = "0.3.23", features = ["env-filter", "json"] }
```

The shortest possible init:

```rust
tracing_subscriber::fmt::init();
```

```rust
use tracing::{debug, error, info, trace, warn};

fn main() {
    tracing_subscriber::fmt::init();

    trace!("entering");
    debug!("cache lookup");
    info!("server started");
    warn!("pool near capacity");
    error!("query failed");
}
```

Output with **no** `RUST_LOG` set:

```
2026-10-25T16:58:21.567591Z ERROR t1: query failed
```

Only `ERROR`. `fmt::init()` defaults to `ERROR` when `RUST_LOG` is absent — a
common surprise. Set a default explicitly in real code (shown below).

With `RUST_LOG=debug`:

```
2026-10-25T16:58:21.890815Z DEBUG t1: cache lookup
2026-10-25T16:58:21.891003Z  INFO t1: server started
2026-10-25T16:58:21.891032Z  WARN t1: pool near capacity
2026-10-25T16:58:21.891053Z ERROR t1: query failed
```

With `RUST_LOG=trace`, `entering` appears too. Levels are ordered
`TRACE < DEBUG < INFO < WARN < ERROR`, and a directive enables its level **and
everything above it**.

## The builder

```rust
use tracing_subscriber::EnvFilter;

tracing_subscriber::fmt()
    .with_env_filter(EnvFilter::from_default_env())
    .with_ansi(false)
    .with_target(true)
    .without_time()
    .init();
```

| Method | Effect |
|---|---|
| `.with_env_filter(..)` | use `RUST_LOG` directives instead of a flat level |
| `.with_ansi(false)` | no colour codes — what you want when piping to a file |
| `.with_target(true)` | show the module path (`t2:`) |
| `.without_time()` | drop the timestamp (your log platform adds one) |
| `.with_timer(fmt::time::uptime())` | seconds since start instead of a clock |
| `.with_thread_ids(true)` | include the worker thread |
| `.with_line_number(true)` | file and line of the event |
| `.json()` | one JSON object per line |
| `.compact()` / `.pretty()` | denser / multi-line human format |

A sane production default that still respects `RUST_LOG`:

```rust
let filter = EnvFilter::try_from_default_env()
    .unwrap_or_else(|_| EnvFilter::new("info,tower_http=debug,sqlx=warn"));
tracing_subscriber::fmt().with_env_filter(filter).init();
```

## Structured fields on events

Fields go before the message, as `name = value`.

```rust
let user = User { id: 7, email: "raj@example.com".into() };
let path = "/api/users/7";

info!(user_id = user.id, path, "request handled");
info!(status = 200, duration_ms = 12.4, cached = false, "response sent");
info!(?user, "debug formatting with ?");
info!(%path, "display formatting with %");
info!(user.id = user.id, user.email = %user.email, "dotted field names");
warn!(retries = 3, max = 3, "giving up");
info!(message = "explicit message field", code = 42);
```

Output:

```
 INFO t2: request handled user_id=7 path="/api/users/7"
 INFO t2: response sent status=200 duration_ms=12.4 cached=false
 INFO t2: debug formatting with ? user=User { id: 7, email: "raj@example.com" }
 INFO t2: display formatting with % path=/api/users/7
 INFO t2: dotted field names user.id=7 user.email=raj@example.com
 INFO t2: giving up retries=3 max=3
 INFO t2: explicit message field code=42
```

The three sigils:

| Form | Meaning |
|---|---|
| `field = value` | the value's native type (`i64`, `f64`, `bool`, `&str`) |
| `?value` or `field = ?v` | record via `Debug` |
| `%value` or `field = %v` | record via `Display` |
| bare `path` | shorthand for `path = path` |

Structured fields are not cosmetic: a JSON backend turns each into a queryable
column. `info!("user {} failed", id)` gives you a string to regex;
`info!(user_id = id, "failed")` gives you a field to filter on.

## Spans

`info_span!` creates one; `.enter()` makes it current for the scope of the guard.

```rust
let span = info_span!("request", request_id = "req-abc123", method = "GET");
let _g = span.enter();
info!("start");

let inner = info_span!("db");
{
    let _g2 = inner.enter();
    info!(query = "SELECT 1", "executing");
}
info!("done");
```

Output:

```
 INFO request{request_id="req-abc123" method="GET"}: t3: start
 INFO request{request_id="req-abc123" method="GET"}:db: t3: executing query="SELECT 1"
 INFO request{request_id="req-abc123" method="GET"}: t3: done
```

Nested spans join with `:`, and the inner event inherits the outer fields. The
guard must be bound to a name — `let _ = span.enter()` drops it immediately and
the span is never current. Use `let _g = ...`.

**Never hold an `.enter()` guard across an `.await`.** It is tied to the thread,
and the task can move. Use `#[instrument]` or `.instrument(span)` instead.

## Spans vs events

| | Span | Event |
|---|---|---|
| Represents | a period of work | a moment |
| Created by | `info_span!`, `#[instrument]` | `info!`, `warn!`, ... |
| Has | a name, fields, enter/exit times | a message, fields, a timestamp |
| Analogy | the stack frame | the log line |

Events inside a span inherit the whole ancestor chain's fields.

## #[instrument]

On a function, it opens a span named after the function with every argument as a
field.

```rust
#[instrument]
async fn fetch_user(id: u64) -> String {
    info!("querying");
    tokio::time::sleep(std::time::Duration::from_millis(5)).await;
    info!("got row");
    format!("user{id}")
}

#[instrument(skip(password), fields(user_id))]
async fn login(email: &str, password: &str) -> bool {
    tracing::Span::current().record("user_id", 42);
    info!("checking credentials");
    !password.is_empty() && !email.is_empty()
}

#[instrument(name = "handler", level = "info", ret)]
fn add(a: i32, b: i32) -> i32 { a + b }
```

Output, all three called inside the outer `request` span:

```
 INFO request{request_id="req-abc123" method="GET"}:fetch_user{id=7}: t3: querying
 INFO request{request_id="req-abc123" method="GET"}:fetch_user{id=7}: t3: got row
 INFO request{request_id="req-abc123" method="GET"}:login{email="raj@example.com" user_id=42}: t3: checking credentials
 INFO request{request_id="req-abc123" method="GET"}:handler{a=2 b=3}: t3: return=5
```

Three things to read off that output. `fetch_user`'s span persisted across the
`sleep().await`. `login`'s `password` is absent because of `skip`, and
`user_id` was declared empty then filled with `record`. `add` was renamed and
`ret` logged its return value.

| Argument | Effect |
|---|---|
| `skip(a, b)` | omit those arguments (use for secrets and big values) |
| `skip_all` | omit every argument |
| `fields(k = v, k2)` | add fields; bare name declares an empty one to `record` later |
| `name = "x"` | override the span name |
| `level = "debug"` | span level (default `INFO` from `#[instrument]`) |
| `ret` | log the return value on exit |
| `err` | log at `ERROR` if the function returns `Err` |

`skip` matters: `#[instrument]` on a handler taking a `Request` or a password
will happily record the whole thing. Default to `skip_all` plus explicit
`fields(..)` on anything handling user input.

## Span lifecycle events

By default a span produces no line of its own — only its events do. Turn that on
with `FmtSpan`.

```rust
use tracing_subscriber::fmt::format::FmtSpan;

tracing_subscriber::fmt()
    .with_span_events(FmtSpan::NEW | FmtSpan::CLOSE)
    .init();
```

Output for `work(3)`:

```
 INFO work{n=3}: t6: new
DEBUG work{n=3}: t6: inner
 INFO work{n=3}: t6: close
```

`FmtSpan::NONE`, `NEW`, `ENTER`, `EXIT`, `CLOSE`, `ACTIVE`, `FULL` are the
options. `CLOSE` alone is the useful one in production; `FULL` is extremely
noisy in async code because a span enters and exits at every `.await`.

## RUST_LOG syntax

Directives are comma-separated, `target=level`, and the most specific match
wins. All of these are real output from the same binary with modules `t6::db`
and `t6::http`.

`RUST_LOG=info` — everything at info and above:

```
 INFO t6: top
 INFO t6::db: db info
 INFO t6::http: http info
```

`RUST_LOG=info,t6::db=debug` — raise one module:

```
 INFO t6: top
DEBUG t6::db: db debug
 INFO t6::db: db info
 INFO t6::http: http info
```

`RUST_LOG=warn,t6=debug` — quiet globally, verbose for your own crate. This is
the directive to use when a dependency is flooding you:

```
 INFO t6: top
DEBUG t6::db: db debug
 INFO t6::db: db info
DEBUG t6::http: http debug
 INFO t6::http: http info
```

`RUST_LOG=t6[work]=debug` — only events inside a span named `work`:

```
 INFO work{n=3}: t6: new
DEBUG work{n=3}: t6: inner
 INFO work{n=3}: t6: close
```

| Directive | Meaning |
|---|---|
| `debug` | global level |
| `my_crate=debug` | one crate |
| `my_crate::db=trace` | one module |
| `warn,my_crate=debug` | global floor plus an exception |
| `my_crate[span_name]=debug` | events inside that span |
| `my_crate[{user_id=7}]` | spans with that field value |
| `off` | nothing |

Useful real-world starting point for the Module 17 stack:

```
RUST_LOG=info,my_api=debug,tower_http=debug,sqlx::query=warn
```

`sqlx::query` logs every statement at `INFO` by default, which is great locally
and far too much in production.

## JSON output for production

Log aggregators want one JSON object per line, not a human-formatted string.

```rust
tracing_subscriber::fmt()
    .json()
    .flatten_event(true)
    .with_current_span(true)
    .with_span_list(false)
    .with_env_filter(EnvFilter::from_default_env())
    .init();
```

```rust
#[instrument(fields(request_id = "req-9f2"))]
async fn handle(user_id: u64) {
    info!(status = 200, duration_ms = 7.5, "request complete");
}
```

Output (one line, shown expanded):

```json
{
    "timestamp": "2026-10-25T16:58:53.132376Z",
    "level": "INFO",
    "message": "request complete",
    "status": 200,
    "duration_ms": 7.5,
    "target": "t5",
    "span": {
        "request_id": "req-9f2",
        "user_id": 42,
        "name": "handle"
    }
}
```

`status` and `duration_ms` are real JSON fields you can aggregate on, and
`span.request_id` ties every line of the request together. `flatten_event(true)`
lifts event fields to the top level; without it they nest under `"fields"`.
`with_span_list(true)` adds the full ancestor chain as an array, which is more
complete and much larger.

Pick the format from config, not a `cfg!`:

```rust
if config.log_json {
    tracing_subscriber::fmt().json().with_env_filter(filter).init();
} else {
    tracing_subscriber::fmt().with_env_filter(filter).init();
}
```

## TraceLayer from tower-http

One layer instruments every request. Cross-reference 17.7 for `tower` layers in
general.

```rust
use tower_http::trace::TraceLayer;

let app = Router::new()
    .route("/users/{id}", get(get_user))
    .layer(TraceLayer::new_for_http());
```

Output with `RUST_LOG="info,tower_http=debug"` for one 200 and one 404:

```
DEBUG request{method=GET uri=/users/7 version=HTTP/1.1}: tower_http::trace::on_request: started processing request
 INFO request{method=GET uri=/users/7 version=HTTP/1.1}:get_user{id=7}: t9: loading user
DEBUG request{method=GET uri=/users/7 version=HTTP/1.1}: tower_http::trace::on_response: finished processing request latency=0 ms status=200
DEBUG request{method=GET uri=/users/7 version=HTTP/1.1}: tower_http::trace::on_eos: end of stream stream_duration=0 ms
DEBUG request{method=GET uri=/nope version=HTTP/1.1}: tower_http::trace::on_request: started processing request
DEBUG request{method=GET uri=/nope version=HTTP/1.1}: tower_http::trace::on_response: finished processing request latency=0 ms status=404
```

The handler's `#[instrument]` span nests inside the layer's `request` span, and
you get latency and status for free.

## A request id on every line

Override the span the layer makes.

```rust
use axum::http::Request;

let app = Router::new()
    .route("/users/{id}", get(get_user))
    .layer(
        TraceLayer::new_for_http().make_span_with(|req: &Request<_>| {
            info_span!(
                "http",
                request_id = %uuid::Uuid::new_v4(),
                method = %req.method(),
                path = %req.uri().path(),
            )
        }),
    );
```

Output for two requests:

```
DEBUG http{request_id=816a6410-bc41-4534-b30d-3df5345e44cd method=GET path=/users/7}: tower_http::trace::on_request: started processing request
 INFO http{request_id=816a6410-bc41-4534-b30d-3df5345e44cd method=GET path=/users/7}:get_user{id=7}: t10: loading user
DEBUG http{request_id=816a6410-bc41-4534-b30d-3df5345e44cd method=GET path=/users/7}: tower_http::trace::on_response: finished processing request latency=0 ms status=200
DEBUG http{request_id=30edb172-caba-42ca-b4c9-cb10156c5bd4 method=GET path=/users/9}: tower_http::trace::on_request: started processing request
 INFO http{request_id=30edb172-caba-42ca-b4c9-cb10156c5bd4 method=GET path=/users/9}:get_user{id=9}: t10: loading user
```

Use `path()` rather than the full `uri()` — a query string often contains tokens
and PII. In real services you read an incoming `X-Request-Id` header first and
generate a UUID only when it is absent, so the id matches your load balancer's.

## The log-to-tracing bridge

Dependencies that use the `log` crate are captured automatically —
`tracing-subscriber` enables `tracing-log` by default.

```rust
log::warn!("this came from the log crate");
log::info!(target: "legacy_dep", "an old dependency logging");
```

Output:

```
 WARN t7: this came from the log crate
 INFO legacy_dep: an old dependency logging
```

They become ordinary events, filterable by `RUST_LOG` and nested in whatever
span is current. No bridge code needed.

## Choosing levels

| Level | Use for | In production |
|---|---|---|
| `ERROR` | the request failed and someone must look | on, pages someone |
| `WARN` | degraded but handled: retry, fallback, near a limit | on |
| `INFO` | one line per significant state change or request | on |
| `DEBUG` | the decisions inside a request | off, on per-module when debugging |
| `TRACE` | loop bodies, every row | off |

The failure mode is `INFO` for things that happen per-item rather than per-
request. A loop over 10,000 rows logging at `INFO` costs real money in your
aggregator and buries the lines that matter.

## What not to log

Never: passwords, password hashes, JWTs or session tokens, API keys, full
database URLs (they carry the password), credit card numbers, full request
bodies on auth endpoints.

Careful with: email addresses, IPs, user names, full URLs with query strings.
These are PII under GDPR and similar regimes, and logs are retained and widely
readable. Prefer a `user_id` to an email.

Mechanical defences: the redacting `Secret` newtype from 18.1, `skip` on
`#[instrument]`, and never `?request` on a whole request object.

```rust
#[instrument(skip_all, fields(user_id = %claims.sub))]
async fn update_profile(claims: Claims, Json(body): Json<UpdateProfile>) { }
```

`skip_all` then opting fields back in is the safe default. The opposite — listing
what to skip — fails the moment someone adds a field.

## Quick Reference

| Task | Code |
|---|---|
| Init, simple | `tracing_subscriber::fmt::init()` (defaults to ERROR!) |
| Init with filter | `fmt().with_env_filter(EnvFilter::from_default_env()).init()` |
| Filter with fallback | `EnvFilter::try_from_default_env().unwrap_or_else(\|_\| EnvFilter::new("info"))` |
| Event | `info!(field = v, "message")` |
| Debug field | `info!(?value)` |
| Display field | `info!(%value)` |
| Create a span | `info_span!("name", k = v)` |
| Enter a span (sync) | `let _g = span.enter();` |
| Span across await | `#[instrument]` or `fut.instrument(span)` |
| Current span | `tracing::Span::current()` |
| Fill a declared field | `Span::current().record("k", v)` |
| Span open/close lines | `.with_span_events(FmtSpan::CLOSE)` |
| JSON output | `.json().flatten_event(true)` |
| HTTP layer | `TraceLayer::new_for_http()` |
| Custom HTTP span | `TraceLayer::new_for_http().make_span_with(\|req\| ..)` |

## Common Mistakes

**`tracing_subscriber::fmt::init()` with no `RUST_LOG`.** You get `ERROR` only
and conclude logging is broken. Supply a fallback filter.

**Holding `span.enter()` across `.await`.** The guard is thread-bound; the task
is not. Spans leak into unrelated work. Use `#[instrument]`.

**`let _ = span.enter();`** drops the guard instantly, so the span is never
current. Bind it: `let _g = span.enter();`.

**`#[instrument]` on a handler taking a password or token.** Every argument
becomes a field. Use `skip_all` and name the safe fields.

**Logging in a hot loop at `INFO`.** Per-item logging at a level you ship is how
you get a million lines from one request. Use `DEBUG`, or log a summary after
the loop.

## Exercises

1. Set up a subscriber that reads `RUST_LOG` but falls back to
   `info,sqlx=warn`. Prove both paths with and without the variable set.
2. Instrument an async `fn charge(user_id: u64, card: &str)` so `card` never
   appears in output. Verify with `RUST_LOG=info`.
3. Spawn three tasks with different sleep durations inside distinct spans. Show
   the lines interleave but each keeps its own span fields.
4. Switch your subscriber to `.json()` and pipe the output through `jq` to
   select only lines where `status` is 500.
5. Add a `TraceLayer` whose span reads an `X-Request-Id` header if present and
   generates a UUID otherwise. Confirm both cases from `curl`.

---

[← Course Index](../README.md)
