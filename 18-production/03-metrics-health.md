[← 18.2 Logging and Tracing](02-tracing.md) · [Course Index](../README.md) · [18.4 Graceful Shutdown →](04-graceful-shutdown.md)

# 18.3 — Metrics and Health Checks

## What You'll Learn

- `metrics` + `metrics-exporter-prometheus`: counters, gauges, histograms
- A `/metrics` endpoint on an Axum router, and a middleware that fills it
- Label cardinality: the one mistake that takes down your metrics backend
- Liveness vs readiness, and a `/health/ready` that actually checks the DB pool

## Coming From Another Language

| Other | Rust |
|---|---|
| `prom-client` (Node) | `metrics` facade + `metrics-exporter-prometheus` |
| `prometheus_client` (Python) | same split: facade, then an exporter |
| registry object passed around | a global recorder, installed once |

Like `tracing`, `metrics` is a **facade**. You call `counter!` anywhere without
threading a registry through your types; an exporter installed at startup
decides where the numbers go. If no recorder is installed the macros compile to
almost nothing.

## The four signals, briefly

Latency, traffic, errors, saturation. Two framings put them to use: **RED** for
request-driven services (Rate, Errors, Duration — what you alert on) and **USE**
for resources (Utilization, Saturation, Errors — what you capacity-plan with).
A web service needs RED on every route and USE on the DB pool.

## Setup

```toml
metrics = "0.24.6"
metrics-exporter-prometheus = "0.18.3"
```

```rust
use metrics_exporter_prometheus::PrometheusBuilder;

let handle = PrometheusBuilder::new().install_recorder().unwrap();
```

`install_recorder()` registers the global recorder and hands back a
`PrometheusHandle` whose `.render()` produces the scrape text. (`install()`
instead spawns its own HTTP listener — we want the endpoint on our own router,
so `install_recorder` is the right call.)

## The three metric types

```rust
use metrics::{counter, gauge, histogram, describe_counter, describe_histogram, Unit};

describe_counter!("http_requests_total", "Total HTTP requests");
describe_histogram!("http_request_duration_seconds", Unit::Seconds, "Request latency");

counter!("app_starts_total").increment(1);
gauge!("db_pool_connections", "state" => "idle").set(3.0);
gauge!("db_pool_connections", "state" => "active").set(2.0);
histogram!("http_request_duration_seconds", "path" => "/hello").record(0.012);
```

| Type | Operations | Use for |
|---|---|---|
| `counter!` | `.increment(n)`, `.absolute(n)` | monotonic totals: requests, errors, bytes |
| `gauge!` | `.set(v)`, `.increment(v)`, `.decrement(v)` | a current value: in-flight, pool size, queue depth |
| `histogram!` | `.record(v)` | a distribution: latencies, payload sizes |

Naming follows Prometheus convention: `snake_case`, a unit suffix
(`_seconds`, `_bytes`, `_total`), and seconds rather than milliseconds.

## A /metrics endpoint on the Axum router

```rust
use axum::{Router, routing::get, extract::State};
use std::sync::Arc;

async fn metrics_handler(State(h): State<Arc<PrometheusHandle>>) -> String {
    h.render()
}

let app = Router::new()
    .route("/hello", get(hello))
    .route("/metrics", get(metrics_handler))
    .with_state(Arc::new(handle));
```

Real `GET /metrics` output after three requests to `/hello`:

```
# TYPE app_starts_total counter
app_starts_total 1

# HELP http_requests_total Total HTTP requests
# TYPE http_requests_total counter
http_requests_total{method="GET",path="/hello",status="200"} 3

# TYPE db_pool_connections gauge
db_pool_connections{state="idle"} 3
db_pool_connections{state="active"} 2

# HELP http_request_duration_seconds Request latency
# TYPE http_request_duration_seconds summary
http_request_duration_seconds{path="/hello",quantile="0"} 0.000007167
http_request_duration_seconds{path="/hello",quantile="0.5"} 0.00001316564144808051
http_request_duration_seconds{path="/hello",quantile="0.9"} 0.00001316564144808051
http_request_duration_seconds{path="/hello",quantile="0.99"} 0.00001316564144808051
http_request_duration_seconds{path="/hello",quantile="1"} 0.000048583
http_request_duration_seconds_sum{path="/hello"} 0.00006891599999999999
http_request_duration_seconds_count{path="/hello"} 3
```

`describe_*` produced the `# HELP` lines; metrics without one still export, just
undocumented.

Note the histogram came out as `summary` with per-process quantiles. Those
**cannot be aggregated across replicas** — averaging p99s from three pods is
meaningless. Ask for real buckets instead.

## Real histogram buckets

```rust
use metrics_exporter_prometheus::{PrometheusBuilder, Matcher};

let handle = PrometheusBuilder::new()
    .set_buckets_for_metric(
        Matcher::Full("http_request_duration_seconds".to_string()),
        &[0.001, 0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0],
    )
    .unwrap()
    .install_recorder()
    .unwrap();
```

Output after recording `0.002, 0.008, 0.03, 0.4, 3.0`:

```
# HELP http_request_duration_seconds Request latency
# TYPE http_request_duration_seconds histogram
http_request_duration_seconds_bucket{path="/users",le="0.001"} 0
http_request_duration_seconds_bucket{path="/users",le="0.005"} 1
http_request_duration_seconds_bucket{path="/users",le="0.01"} 2
http_request_duration_seconds_bucket{path="/users",le="0.025"} 2
http_request_duration_seconds_bucket{path="/users",le="0.05"} 3
http_request_duration_seconds_bucket{path="/users",le="0.1"} 3
http_request_duration_seconds_bucket{path="/users",le="0.25"} 3
http_request_duration_seconds_bucket{path="/users",le="0.5"} 4
http_request_duration_seconds_bucket{path="/users",le="1"} 4
http_request_duration_seconds_bucket{path="/users",le="2.5"} 4
http_request_duration_seconds_bucket{path="/users",le="5"} 5
http_request_duration_seconds_bucket{path="/users",le="+Inf"} 5
http_request_duration_seconds_sum{path="/users"} 3.44
http_request_duration_seconds_count{path="/users"} 5
```

Buckets are cumulative (`le` = less than or equal), which is what lets
Prometheus' `histogram_quantile` work across replicas.

| Matcher | Matches |
|---|---|
| `Matcher::Full("name")` | exactly that metric |
| `Matcher::Prefix("http_")` | all metrics starting with it |
| `Matcher::Suffix("_seconds")` | all metrics ending with it |
| `.set_buckets(&[..])` | the global default for every histogram |

Choose buckets around your actual SLO. If you promise p99 under 200ms, you need
buckets near 0.2 — otherwise the quantile is interpolated from a gap and lies.

## A metrics middleware

Instrumenting every handler by hand does not scale. One layer does it.

```rust
use axum::{extract::{MatchedPath, Request}, middleware::Next, response::Response};
use std::time::Instant;

async fn track(req: Request, next: Next) -> Response {
    let path = req.extensions().get::<MatchedPath>()
        .map(|p| p.as_str().to_owned())
        .unwrap_or_else(|| "unknown".to_owned());
    let method = req.method().clone();
    let start = Instant::now();

    gauge!("http_requests_in_flight").increment(1.0);
    let res = next.run(req).await;
    gauge!("http_requests_in_flight").decrement(1.0);

    let status = res.status().as_u16().to_string();
    let labels = [("method", method.to_string()), ("path", path), ("status", status)];
    counter!("http_requests_total", &labels).increment(1);
    histogram!("http_request_duration_seconds", &labels[..2]).record(start.elapsed().as_secs_f64());
    res
}
```

```rust
let app = Router::new()
    .route("/users/{id}", get(user))
    .layer(middleware::from_fn(track))
    .route("/metrics", get(metrics_ep))
    .with_state(handle);
```

After four requests to `/users/1..4` and one to a non-existent route:

```
# TYPE http_requests_total counter
http_requests_total{method="GET",path="/users/{id}",status="200"} 4
http_requests_total{method="GET",path="unknown",status="404"} 1

# TYPE http_requests_in_flight gauge
http_requests_in_flight 0

# TYPE http_request_duration_seconds histogram
http_request_duration_seconds_bucket{method="GET",path="/users/{id}",le="0.005"} 4
...
http_request_duration_seconds_count{method="GET",path="/users/{id}"} 4
```

**`MatchedPath` is the whole point.** Four requests with four different ids
produced **one** series labelled `/users/{id}`, not four. `req.uri().path()`
would have produced four — and a million in production.

`/metrics` is registered *after* `.layer(track)`, so scrapes don't pollute the
numbers. The gauge is back to `0` because every request finished.

## Label cardinality

A time series exists for every unique combination of label values. The exporter
holds them all in memory, and so does Prometheus.

| Label value | Distinct values | Verdict |
|---|---|---|
| `path = "/users/{id}"` | ~dozens of routes | fine |
| `method = "GET"` | ~8 | fine |
| `status = "200"` | ~15 | fine |
| `user_id = "48213"` | millions | **never** |
| `request_id = <uuid>` | one per request | **never** |
| `error = <message>` | unbounded | **never** |
| `email`, raw URL with query | unbounded | **never** |

Multiply, don't add: method x path x status is 8 x 40 x 15 = 4,800 series, which
is nothing. Add `user_id` and it is 4,800 x your user count, which will OOM the
process. The rule: a label value must come from a small, fixed set you could
write down.

High-cardinality identifiers belong in **logs and traces** (18.2), which are
designed for them. Metrics are for aggregates.

## Pool and business metrics

```rust
gauge!("db_pool_connections", "state" => "total").set(pool.size() as f64);
gauge!("db_pool_connections", "state" => "idle").set(pool.num_idle() as f64);

counter!("orders_created_total").increment(1);
counter!("payment_failures_total", "reason" => "card_declined").increment(1);
histogram!("order_value_dollars").record(amount);
```

Pool saturation is the USE signal that explains most mystery latency: if
`idle` sits at zero while `total` is at `max_connections`, every request is
queueing for a connection and no amount of application tuning helps.

Note `reason` is a closed set of your own enum variants, not an error string.

## Liveness vs readiness

These answer different questions and must not be the same endpoint.

| | Liveness | Readiness |
|---|---|---|
| Question | is the process wedged? | can it serve traffic right now? |
| Checks | nothing — just that we respond | dependencies: DB pool, cache, migrations |
| On failure | the orchestrator **kills and restarts** | traffic is **routed away**, process left alone |
| Must not | touch the database | lie about dependencies |

The classic outage: readiness logic in the liveness probe. Postgres hiccups,
every replica reports unhealthy, Kubernetes restarts them all at once, and now
you have a thundering herd on a database that was about to recover.

## Liveness

```rust
async fn liveness() -> &'static str { "ok" }
```

That is correct and complete. If the process can run that handler, the event
loop is alive; restarting it would not help with anything else.

## Readiness that checks the pool

```rust
use axum::{http::StatusCode, response::{IntoResponse, Json}};
use serde_json::json;
use std::time::Duration;

async fn readiness(State(s): State<AppState>) -> impl IntoResponse {
    let db = tokio::time::timeout(
        Duration::from_secs(2),
        sqlx::query_scalar::<_, i32>("SELECT 1").fetch_one(&s.pool),
    )
    .await;

    match db {
        Ok(Ok(_)) => (
            StatusCode::OK,
            Json(json!({
                "status": "ready",
                "checks": { "database": "ok" },
                "pool": { "size": s.pool.size(), "idle": s.pool.num_idle() }
            })),
        ),
        Ok(Err(e)) => (
            StatusCode::SERVICE_UNAVAILABLE,
            Json(json!({ "status": "not_ready", "checks": { "database": e.to_string() } })),
        ),
        Err(_) => (
            StatusCode::SERVICE_UNAVAILABLE,
            Json(json!({ "status": "not_ready", "checks": { "database": "timed out after 2s" } })),
        ),
    }
}
```

Healthy pool:

```
GET /health/ready -> 200 OK
{"checks":{"database":"ok"},"pool":{"idle":0,"size":1},"status":"ready"}
```

Pointed at a database that isn't there (`connect_lazy` to a dead host, 1s
acquire timeout):

```
GET /health/live -> 200 OK
ok

GET /health/ready -> 503 Service Unavailable
{"checks":{"database":"pool timed out while waiting for an open connection"},"status":"not_ready"}
```

Liveness stayed 200 while readiness went 503 — exactly the split you want. The
process keeps running, stops receiving traffic, and recovers on its own when the
database returns.

Three details that matter. The outer `tokio::time::timeout` is essential: without
it a hung connection makes the probe itself hang and the orchestrator reads a
timeout as a failure of the wrong kind. `SELECT 1` and nothing heavier — probes
run every few seconds on every replica. And a `503` body that names the failing
dependency saves you a debugging session.

## Returning 503 correctly

Axum needs a `StatusCode` in the tuple to override the default 200. Both match
arms must return the same type, so `Json<Value>` in all of them — a `Json<T>` of
two different `T`s will not compile.

A tidier version with the response type named explicitly:

```rust
type HealthResponse = (StatusCode, Json<serde_json::Value>);

async fn readiness(State(s): State<AppState>) -> HealthResponse { /* ... */ }
```

For several dependencies, check them concurrently and report each:

```rust
let (db, cache) = tokio::join!(check_db(&s.pool), check_redis(&s.redis));
let ready = db.is_ok() && cache.is_ok();
let code = if ready { StatusCode::OK } else { StatusCode::SERVICE_UNAVAILABLE };
```

Decide per dependency whether it is *required*. A broken cache you can fall back
through should not take you out of the load balancer; a broken primary database
should.

## Routing and exposure

```rust
let app = Router::new()
    .route("/health/live", get(liveness))
    .route("/health/ready", get(readiness))
    .route("/metrics", get(metrics_handler))
    .nest("/api", api_routes().layer(middleware::from_fn(track)))
    .with_state(state);
```

Keep `/metrics` and `/health` off the authenticated, rate-limited, traced path:
probes every two seconds would otherwise dominate your logs and your metrics.
`/metrics` should not be reachable from the public internet — bind it to an
internal port or block it at the ingress. It leaks your route list, traffic
volumes and error rates.

## Kubernetes probes

```yaml
livenessProbe:
  httpGet: { path: /health/live, port: 8080 }
  initialDelaySeconds: 5
  periodSeconds: 10
  failureThreshold: 3
readinessProbe:
  httpGet: { path: /health/ready, port: 8080 }
  initialDelaySeconds: 2
  periodSeconds: 5
  failureThreshold: 2
startupProbe:
  httpGet: { path: /health/live, port: 8080 }
  periodSeconds: 2
  failureThreshold: 30
```

Liveness gets a *longer* period and a *higher* threshold than readiness: being
slow to kill a pod is cheap, killing a healthy one is not. The `startupProbe`
suspends the other two until the app is up, which lets a slow migration run
without a restart loop.

The Prometheus side:

```yaml
annotations:
  prometheus.io/scrape: "true"
  prometheus.io/port: "8080"
  prometheus.io/path: "/metrics"
```

## OpenTelemetry in one line

For distributed tracing across services — one trace id following a request
through your API, a worker and a third-party call — add `opentelemetry` and
`tracing-opentelemetry`, which exports the spans you already create with
`#[instrument]` to a collector; OTel can carry metrics too, but the
Prometheus exporter above is simpler if metrics are all you need.

## Quick Reference

| Task | Code |
|---|---|
| Install recorder | `PrometheusBuilder::new().install_recorder()?` |
| Render scrape text | `handle.render()` |
| Counter | `counter!("name", "k" => "v").increment(1)` |
| Gauge set | `gauge!("name").set(3.0)` |
| Gauge delta | `gauge!("n").increment(1.0)` / `.decrement(1.0)` |
| Histogram | `histogram!("name").record(secs)` |
| Document | `describe_counter!("n", "help text")` |
| Global buckets | `.set_buckets(&[..])?` |
| Per-metric buckets | `.set_buckets_for_metric(Matcher::Full(..), &[..])?` |
| Route template as label | `req.extensions().get::<MatchedPath>()` |
| Pool gauges | `pool.size()`, `pool.num_idle()` |
| 503 from a handler | `(StatusCode::SERVICE_UNAVAILABLE, Json(..))` |
| Bound a probe | `tokio::time::timeout(d, fut).await` |

## Common Mistakes

**`user_id` or a request id as a label.** Unbounded cardinality OOMs the
exporter and the scraper. Those belong in traces.

**`req.uri().path()` instead of `MatchedPath`.** One series per id. Use the
route template.

**The same handler for liveness and readiness.** A database blip restarts every
replica simultaneously. Liveness must check nothing.

**A readiness probe with no timeout.** A hung connection hangs the probe. Wrap
it in `tokio::time::timeout`.

**Trusting per-process quantiles.** The default `summary` output cannot be
aggregated across replicas. Configure buckets.

## Exercises

1. Install the Prometheus recorder, increment a counter with two labels, and
   print `handle.render()`. Confirm the `# TYPE` line.
2. Add buckets `[0.01, 0.1, 1.0]` for one metric only, using `Matcher::Full`.
   Record five values and check the cumulative counts by hand.
3. Write the `track` middleware and prove that `/items/1` and `/items/2`
   collapse to a single `path="/items/{id}"` series.
4. Write `/health/ready` against a `PgPool` pointed at a port with nothing on
   it. Confirm you get 503 and that `/health/live` still returns 200.
5. Extend readiness to check two dependencies with `tokio::join!`, where one is
   optional: report its state in the body but keep the status 200.

---

[← 18.2 Logging and Tracing](02-tracing.md) · [Course Index](../README.md) · [18.4 Graceful Shutdown →](04-graceful-shutdown.md)
