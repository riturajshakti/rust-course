[← Course Index](../README.md)

# 18.4 — Graceful Shutdown

## What You'll Learn

- What an abrupt exit does to in-flight requests, measured
- `axum::serve().with_graceful_shutdown()` and a SIGTERM/SIGINT future
- A shutdown timeout so one stuck request can't hang the deploy
- Ordering: drain HTTP, stop background tasks, close the pool

## Coming From Another Language

| Other | Rust |
|---|---|
| `server.close(cb)` (Node) | `axum::serve(..).with_graceful_shutdown(fut)` |
| `process.on('SIGTERM', ..)` | `tokio::signal::unix::signal(SignalKind::terminate())` |
| `signal.signal(SIGTERM, h)` (Python) | same, but `.recv().await` instead of a callback |
| gunicorn handling it for you | you write it; nothing is implicit |

Rust has no default signal handling beyond the OS one: SIGTERM terminates the
process immediately. Every deploy, scale-down and pod eviction sends SIGTERM, so
"immediately" means dropped requests on every single release.

## What abrupt exit costs

A handler that takes 1.5 seconds, with **no** graceful shutdown. SIGTERM arrives
0.4s into the request.

```rust
async fn slow() -> &'static str {
    info!("handler start");
    tokio::time::sleep(Duration::from_millis(1500)).await;
    info!("handler done");
    "finished"
}

axum::serve(listener, app).await.unwrap();
```

Server log:

```
 INFO g2: listening on 3050
 INFO g2: handler start
```

Client:

```
curl exit: 000
curl rc=52
```

`handler done` never printed. curl rc 52 is "empty reply from server" — the
connection died with no response at all. Not a 500, not a retry-able 503: a
truncated TCP stream. Multiply by your request rate times your deploy frequency.

## The same thing, done gracefully

```rust
axum::serve(listener, app)
    .with_graceful_shutdown(shutdown_signal())
    .await
    .unwrap();

info!("server stopped, all connections drained");
```

Same timing — SIGTERM at 0.4s into a 1.5s request:

```
 INFO g1: listening on 3050
 INFO g1: handler start
 INFO g1: received SIGTERM
 INFO g1: shutdown signal received, draining
 INFO g1: handler done
 INFO g1: server stopped, all connections drained
```

Client:

```
finished
```

The signal arrived, the server stopped accepting, the in-flight handler ran to
completion, the client got its response, and `serve()` returned.

## The shutdown signal future

`with_graceful_shutdown` takes a future; whenever it resolves, draining starts.
Listen for both signals, and keep it compiling on Windows.

```rust
async fn shutdown_signal() {
    let ctrl_c = async {
        tokio::signal::ctrl_c().await.expect("ctrl_c handler")
    };

    #[cfg(unix)]
    let terminate = async {
        tokio::signal::unix::signal(tokio::signal::unix::SignalKind::terminate())
            .expect("SIGTERM handler")
            .recv()
            .await;
    };
    #[cfg(not(unix))]
    let terminate = std::future::pending::<()>();

    tokio::select! {
        _ = ctrl_c => info!("received SIGINT"),
        _ = terminate => info!("received SIGTERM"),
    }
    info!("shutdown signal received, draining");
}
```

Needs `tokio = { version = "1.54.1", features = ["signal", ...] }` (included in
`full`).

| Signal | Sent by | Default action |
|---|---|---|
| SIGINT | Ctrl-C in a terminal | terminate |
| SIGTERM | `docker stop`, Kubernetes, `kill` | terminate |
| SIGKILL | `kill -9`, grace period expiry | **unblockable** — you cannot handle it |

`std::future::pending::<()>()` is a future that never completes, which is the
right "this branch can't fire" value for `select!` on non-Unix. The `#[cfg]`
split is necessary because `SignalKind` does not exist on Windows.

Only SIGINT and SIGTERM matter. If SIGKILL arrives you are already out of time —
which is exactly why the grace period below must be shorter than your
orchestrator's.

## What "draining" actually does

The moment the future resolves:

1. the listener stops accepting — **new connections are refused immediately**
2. connections already open are allowed to finish their current request
3. once all of them close, `serve()` returns

Proof of point 1. SIGTERM, then 0.4s later a fresh `curl`:

```
>>> trying a NEW connection during drain:
http_code=000
connection refused (rc=7)
```

rc 7 is "failed to connect". Meanwhile the request that was already in flight
completed normally. That asymmetry is the whole design: shed new load, honour
accepted work.

It also means your load balancer should stop routing to the pod *before* the
SIGTERM — otherwise a few requests hit a closed listener. Kubernetes handles
this by marking the pod unready first, which is why the readiness probe in 18.3
matters here.

## A shutdown timeout

Draining waits for in-flight requests. If one never finishes, you wait forever —
and your orchestrator eventually SIGKILLs you anyway, so you lose the clean
shutdown *and* the time.

Wrap the whole server future.

```rust
let server = axum::serve(listener, app).with_graceful_shutdown(shutdown_signal());

match tokio::time::timeout(Duration::from_secs(3), server).await {
    Ok(Ok(())) => info!("drained cleanly"),
    Ok(Err(e)) => error!("server error: {e}"),
    Err(_) => error!("shutdown timed out after 3s, forcing exit"),
}
info!("exiting");
```

With a handler that sleeps 30 seconds and a 3-second budget:

```
>>> SIGTERM with a 30s request in flight, 3s shutdown budget
server exited after 3s
 INFO g3: listening
 INFO g3: handler start (will take 30s)
 INFO g3: SIGTERM received
ERROR g3: shutdown timed out after 3s, forcing exit
 INFO g3: exiting
```

Measured 3 seconds, not 30. Note the three match arms: `Ok(Ok(()))` is a clean
drain, `Ok(Err(e))` is a server error, `Err(_)` is the timeout. The nesting is
`Result<Result<(), io::Error>, Elapsed>`.

The timeout is a deliberate correctness trade: you abandon a stuck request to
guarantee the process exits. Set it below your orchestrator's grace period so
*you* decide what gets dropped rather than SIGKILL deciding.

## Background tasks: CancellationToken

Spawned tasks are not part of the HTTP server, so draining does not touch them.
A `CancellationToken` from `tokio-util` broadcasts to any number of tasks.

```toml
tokio-util = { version = "0.7.20", features = ["rt"] }
```

```rust
use tokio_util::sync::CancellationToken;

async fn cleanup_worker(token: CancellationToken) {
    let mut tick = tokio::time::interval(Duration::from_millis(400));
    loop {
        tokio::select! {
            _ = tick.tick() => info!("cleanup tick"),
            _ = token.cancelled() => {
                info!("cleanup worker: cancelled, finishing current unit");
                tokio::time::sleep(Duration::from_millis(100)).await;
                info!("cleanup worker: stopped");
                return;
            }
        }
    }
}
```

```rust
let token = CancellationToken::new();
let worker = tokio::spawn(cleanup_worker(token.clone()));
// ... later
token.cancel();
let _ = worker.await;
```

`token.cancelled()` is a future that resolves on cancel and stays resolved, so
latecomers see it too. `token.child_token()` makes a hierarchy — cancelling the
parent cancels children, not the reverse. Lesson 15.4 covers `select!` and
cancellation in general.

## Background tasks: a watch channel

Equivalent with no extra dependency.

```rust
use tokio::sync::watch;

async fn worker(mut rx: watch::Receiver<bool>, id: u32) {
    loop {
        tokio::select! {
            _ = tokio::time::sleep(Duration::from_millis(500)) => info!("worker {id} tick"),
            _ = rx.changed() => { info!("worker {id} stopping"); return; }
        }
    }
}

let (tx, rx) = watch::channel(false);
let w1 = tokio::spawn(worker(rx.clone(), 1));
let w2 = tokio::spawn(worker(rx.clone(), 2));
// ... after the server drains
let _ = tx.send(true);
let _ = tokio::join!(w1, w2);
```

Output:

```
 INFO g5: listening
 INFO g5: worker 1 tick
 INFO g5: worker 2 tick
 INFO g5: SIGTERM
 INFO g5: worker 2 tick
 INFO g5: worker 1 tick
 INFO g5: drained
 INFO g5: worker 2 stopping
 INFO g5: worker 1 stopping
 INFO g5: workers joined
```

Both workers kept ticking during the drain and stopped only after it. That is
the correct order — a worker might be what an in-flight request is waiting on.

Prefer `CancellationToken` when you have more than a couple of tasks or want a
hierarchy; `watch` is fine and dependency-free otherwise. **Never** use
`handle.abort()` for shutdown: it drops the task at its next suspension point,
mid-transaction if that is where it happens to be.

## Ordering

The order is forced by what depends on what:

1. **Stop accepting, drain HTTP.** Requests use the pool and the tasks, so they
   go first.
2. **Cancel background tasks, join them.** Nothing is left to feed them work.
3. **Close the DB pool.** Nothing is left to use it.
4. **Flush anything buffered** — a log appender, a metrics push, a trace
   exporter.

Close the pool earlier and in-flight requests fail with "pool closed". Close it
never and you leak server-side connections until Postgres' idle timeout — which
matters when 50 replicas restart at once against a `max_connections=100` server.

## A complete main()

```rust
#[tokio::main]
async fn main() {
    tracing_subscriber::fmt().with_env_filter("info").init();

    let pool = SqlitePool::connect("sqlite::memory:").await.unwrap();
    let token = CancellationToken::new();
    let worker = tokio::spawn(cleanup_worker(token.clone()));

    let app = Router::new()
        .route("/work", get(handler))
        .with_state(AppState { pool: pool.clone() });
    let listener = tokio::net::TcpListener::bind("127.0.0.1:3052").await.unwrap();
    info!("listening on 3052");

    let server = axum::serve(listener, app).with_graceful_shutdown(shutdown_signal());

    match tokio::time::timeout(Duration::from_secs(10), server).await {
        Ok(Ok(())) => info!("step 1: http server drained"),
        Ok(Err(e)) => error!("server error: {e}"),
        Err(_) => error!("step 1: drain timed out, abandoning in-flight requests"),
    }

    info!("step 2: cancelling background tasks");
    token.cancel();
    match tokio::time::timeout(Duration::from_secs(5), worker).await {
        Ok(_) => info!("step 2: background tasks stopped"),
        Err(_) => error!("step 2: background task did not stop in 5s"),
    }

    info!("step 3: closing database pool");
    pool.close().await;
    info!("step 3: pool closed (is_closed={})", pool.is_closed());

    info!("shutdown complete");
}
```

Run, fire a request at `/work` (800ms sleep plus a query), SIGTERM 0.3s in:

```
 INFO g4: listening on 3052
 INFO g4: cleanup tick
 INFO g4: cleanup tick
 INFO g4: signal: SIGTERM
 INFO g4: cleanup tick
 INFO g4: step 1: http server drained
 INFO g4: step 2: cancelling background tasks
 INFO g4: cleanup worker: cancelled, finishing current unit
 INFO g4: cleanup worker: stopped
 INFO g4: step 2: background tasks stopped
 INFO g4: step 3: closing database pool
 INFO g4: step 3: pool closed (is_closed=true)
 INFO g4: shutdown complete
```

Client:

```
ok 1
```

The request completed with its database query, after the signal, and every
shutdown step ran in order with its own timeout. The `pool.clone()` matters:
`SqlitePool`/`PgPool` are `Arc` inside, so the clone in the state and the
original in `main` are the same pool, and `main` keeps a handle to close.

## Docker and Kubernetes

`docker stop` sends SIGTERM, waits 10 seconds, then SIGKILL. Override with
`docker stop -t 30`.

```yaml
spec:
  terminationGracePeriodSeconds: 30
  containers:
    - name: api
      lifecycle:
        preStop:
          exec: { command: ["sleep", "5"] }
```

The sequence Kubernetes runs: mark the pod unready (load balancers start
removing it), run `preStop`, send SIGTERM, wait `terminationGracePeriodSeconds`,
SIGKILL.

Your budget must fit inside it: `preStop` sleep + your shutdown timeout <
`terminationGracePeriodSeconds`. With the 5s `preStop` and a 10s drain above,
30s is comfortable.

The `preStop` sleep looks like a hack and is the standard fix: endpoint removal
propagates asynchronously, so for a second or two after SIGTERM a proxy may
still send you traffic. Sleeping before SIGTERM lets that settle while you are
still accepting.

One Docker trap that breaks all of this: if your `ENTRYPOINT` is a shell string,
the shell is PID 1 and may not forward SIGTERM to your binary at all. Use the
exec form — `ENTRYPOINT ["/app/server"]` — so your process *is* PID 1. Lesson
18.6 covers the Dockerfile.

## Quick Reference

| Task | Code |
|---|---|
| Graceful serve | `axum::serve(l, app).with_graceful_shutdown(fut).await` |
| Ctrl-C | `tokio::signal::ctrl_c().await` |
| SIGTERM | `signal(SignalKind::terminate())?.recv().await` |
| Never-completing future | `std::future::pending::<()>()` |
| Race two signals | `tokio::select! { _ = a => {}, _ = b => {} }` |
| Bound the drain | `tokio::time::timeout(d, server).await` |
| Broadcast cancel | `CancellationToken::new()` / `.cancel()` / `.cancelled()` |
| Child token | `token.child_token()` |
| Watch alternative | `watch::channel(false)`, `tx.send(true)`, `rx.changed()` |
| Join a task | `handle.await` (never `handle.abort()` for shutdown) |
| Close the pool | `pool.close().await`, check `pool.is_closed()` |
| Docker grace | `docker stop -t 30` |
| K8s grace | `terminationGracePeriodSeconds: 30` |

## Common Mistakes

**No graceful shutdown at all.** Every deploy truncates in-flight responses with
no status code. One line fixes it.

**A drain with no timeout.** One stuck handler and you wait for SIGKILL,
skipping every cleanup step.

**A grace period longer than the orchestrator's.** A 60s timeout under a 30s
`terminationGracePeriodSeconds` means your cleanup never runs.

**Closing the pool before the server drains.** In-flight requests fail with
"pool closed" — you converted dropped connections into 500s.

**`handle.abort()` on background tasks.** It drops the task at an arbitrary
`.await`, possibly mid-transaction. Signal and join instead.

**A shell-form `ENTRYPOINT`.** The shell becomes PID 1 and may swallow SIGTERM,
so your handler never runs. Use the exec form.

## Exercises

1. Add `with_graceful_shutdown` to a server with a 2-second handler. Send
   SIGTERM mid-request and confirm the client gets a full response.
2. Remove it and repeat. Record curl's exit code and explain it.
3. Wrap the server in a 1-second `timeout` and give the handler a 10-second
   sleep. Confirm the process exits after ~1 second.
4. Spawn two background tasks on a `CancellationToken`. Prove they keep running
   during the drain and stop only afterwards.
5. Write the full four-step shutdown with a `PgPool`, logging each step, and
   confirm `pool.is_closed()` is `true` at the end.

---

[← Course Index](../README.md)
