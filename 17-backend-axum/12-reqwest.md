[← Course Index](../README.md)

# 17.12 — HTTP Clients with reqwest

## What You'll Learn

- Why you build one `Client` and clone it, never one per request
- `get`/`post`, `.json()`, `.query()`, `.form()`, headers, timeouts
- Error handling: `error_for_status`, `is_timeout`, `is_connect`, `is_decode`
- Mocking upstream HTTP in tests with `wiremock`, and streaming responses

The outputs on this page are real: this machine had network access, and the
calls go to `jsonplaceholder.typicode.com` and `httpbin.org`. Response ids and
latencies will differ when you run them.

## Coming From Another Language

| Other | reqwest |
|---|---|
| `fetch(url).then(r => r.json())` | `client.get(url).send().await?.json::<T>().await?` |
| `axios.create({ baseURL, timeout })` | `Client::builder().timeout(..).build()?` |
| `requests.Session()` | `Client` — same "reuse the connection pool" idea |
| Go `http.Client` + `json.Decode` | nearly identical |

Two differences from `fetch`. A non-2xx status is **not** an error — you check
or call `error_for_status()`. And `.json::<T>()` deserializes into a concrete
type rather than giving you an untyped object.

## Setup

```bash
cargo add reqwest --features json,query,form,stream
cargo add serde --features derive
```

```toml
reqwest = { version = "0.13.5", features = ["json", "query", "form", "stream"] }
```

Mind the features. In reqwest 0.13, `query` and `form` are **separate features**
from `json` — without them `.query(..)` and `.form(..)` do not exist and the
error is just `no method named query found for struct RequestBuilder`. Also
available: `multipart`, `cookies`, `gzip`, `brotli`, `blocking`.

## Build one `Client`, clone it

```rust
use reqwest::Client;
use std::time::Duration;

let client = Client::builder()
    .timeout(Duration::from_secs(10))
    .connect_timeout(Duration::from_secs(3))
    .user_agent("rust-course/17.12")
    .pool_max_idle_per_host(8)
    .pool_idle_timeout(Duration::from_secs(90))
    .build()?;
```

**Never call `Client::new()` per request.** A `Client` owns a connection pool, a
DNS cache, and the TLS configuration; building one per request means a fresh TCP
handshake and a fresh TLS handshake every time — easily 100ms of added latency
and a risk of exhausting local ports under load.

`Client` is `Arc` internally, so cloning is cheap. Put it in `AppState`
(lesson 17.5):

```rust
#[derive(Clone)]
struct AppState { pool: PgPool, http: Client }
```

| Builder option | Purpose |
|---|---|
| `timeout` | whole request, including body read |
| `connect_timeout` | TCP+TLS only |
| `read_timeout` | between body chunks |
| `user_agent` | default UA header |
| `default_headers` | headers on every request |
| `pool_max_idle_per_host` | kept-alive connections |
| `pool_idle_timeout` | when to close idle ones |
| `redirect(Policy::none())` | don't follow redirects |
| `danger_accept_invalid_certs` | local dev only, never production |

**Always set a `timeout`.** The default is none — a hung upstream hangs your
handler forever, and with no `TimeoutLayer` (lesson 17.7) that hangs the client
too.

## GET and `.json::<T>()`

```rust
#[derive(Debug, Deserialize)]
struct Todo {
    #[serde(rename = "userId")] user_id: u32,
    id: u32,
    title: String,
    completed: bool,
}

let todo: Todo = client.get("https://jsonplaceholder.typicode.com/todos/1")
    .send().await?
    .json().await?;
println!("{:?}", todo);
```

Output:

```
Todo { user_id: 1, id: 1, title: "delectus aut autem", completed: false }
```

Three `await`s do three different things: `.send()` writes the request and reads
the headers, `.json()` reads and parses the body, and the `?`s propagate
separate error types. A 404 with a valid JSON error body would be an `Ok`
response and then a decode error on `.json::<Todo>()` — which is why you check
the status first.

## Inspecting the response

```rust
let res = client.get("https://jsonplaceholder.typicode.com/todos/2").send().await?;
println!("status={} ok={}", res.status(), res.status().is_success());
println!("content-type={:?}", res.headers().get("content-type").and_then(|v| v.to_str().ok()));
let body = res.text().await?;
println!("body={}", body);
```

Output:

```
status=200 OK ok=true
content-type=Some("application/json; charset=utf-8")
body={"userId": 1,"id": 2,"title": "quis ut nam facilis et officia qui","completed": false}
```

`.text()`, `.json::<T>()`, and `.bytes()` all **consume** the response, so take
the status and headers first. `res.url()` gives the final URL after redirects.

## Query parameters

```rust
let res: HttpBinGet = client.get("https://httpbin.org/get")
    .query(&[("page", "2"), ("q", "rust lang")])
    .send().await?
    .json().await?;
println!("url={}", res.url);
println!("args={:?}", res.args);
```

Output:

```
url=https://httpbin.org/get?page=2&q=rust+lang
args={"page": "2", "q": "rust lang"}
```

The space became `+`, correctly encoded without you thinking about it. `.query`
takes anything `Serialize`, so a struct works too — and that is better than
`format!`-ing a URL, which will eventually break on a value containing `&`.

## Headers and auth

```rust
let res: HttpBinGet = client.get("https://httpbin.org/get")
    .header("x-trace-id", "abc-123")
    .bearer_auth("fake-token")
    .send().await?
    .json().await?;
for k in ["X-Trace-Id", "Authorization", "User-Agent"] {
    println!("{k} = {:?}", res.headers.get(k));
}
```

Output:

```
X-Trace-Id = Some("abc-123")
Authorization = Some("Bearer fake-token")
User-Agent = Some("rust-course/17.12")
```

`.bearer_auth(t)` and `.basic_auth(user, Some(pass))` write the `Authorization`
header for you. Put headers that go on **every** request into
`.default_headers(..)` on the builder instead of repeating them.

## POST JSON

```rust
#[derive(Debug, Serialize)]
struct NewPost { title: String, body: String, #[serde(rename = "userId")] user_id: u32 }

#[derive(Debug, Deserialize)]
struct CreatedPost { id: u32, title: String }

let created: CreatedPost = client.post("https://jsonplaceholder.typicode.com/posts")
    .json(&NewPost { title: "Learning Rust".into(), body: "axum + sqlx".into(), user_id: 1 })
    .send().await?
    .json().await?;
println!("{:?}", created);
```

Output:

```
CreatedPost { id: 101, title: "Learning Rust" }
```

`.json(&body)` serializes and sets `content-type: application/json`. The other
body methods: `.body(string_or_bytes)`, `.form(&pairs)`, `.multipart(form)`.

## POST a form

```rust
#[derive(Debug, Deserialize)]
struct FormEcho { form: BTreeMap<String, String> }

let echo: FormEcho = client.post("https://httpbin.org/post")
    .form(&[("name", "raj"), ("role", "dev")])
    .send().await?
    .json().await?;
println!("{:?}", echo.form);
```

Output:

```
{"name": "raj", "role": "dev"}
```

Sets `application/x-www-form-urlencoded`. For a file upload use `multipart`:

```rust
use reqwest::multipart::{Form, Part};

let form = Form::new()
    .text("caption", "my photo")
    .part("file", Part::bytes(bytes).file_name("photo.png").mime_str("image/png")?);
client.post(url).multipart(form).send().await?;
```

## Status checks

A 404 is an `Ok(Response)`. Three ways to handle that.

`error_for_status()` turns 4xx/5xx into an `Err`:

```rust
let res = client.get("https://httpbin.org/status/404").send().await?;
println!("raw status = {}", res.status());
match res.error_for_status() {
    Ok(_) => println!("ok"),
    Err(e) => println!("error_for_status: {} | is_status={} | status={:?}",
        e, e.is_status(), e.status()),
}
```

Output:

```
raw status = 404 Not Found
error_for_status: HTTP status client error (404 Not Found) for url (https://httpbin.org/status/404) | is_status=true | status=Some(404)
```

That lets you chain `?`:

```rust
let todo: Todo = client.get(url).send().await?.error_for_status()?.json().await?;
```

Note it discards the body, so when the upstream returns a useful error payload
use `error_for_status_ref()` first and read the body after.

Or match the status yourself, which is what you want when different codes mean
different things:

```rust
for code in [200_u16, 404, 429, 500] {
    let res = client.get(format!("https://httpbin.org/status/{code}")).send().await?;
    let label = match res.status() {
        StatusCode::OK => "ok",
        StatusCode::NOT_FOUND => "missing",
        StatusCode::TOO_MANY_REQUESTS => "rate limited",
        s if s.is_server_error() => "retry later",
        _ => "other",
    };
    println!("{code} -> {label}");
}
```

Output:

```
200 -> ok
404 -> missing
429 -> rate limited
500 -> retry later
```

Mapping `404` to `Ok(None)` rather than an error is usually right — a missing
upstream resource is a normal outcome, not a failure.

## Classifying errors

`reqwest::Error` has predicates instead of variants.

```rust
let quick = Client::builder().timeout(Duration::from_millis(500)).build()?;
match quick.get("https://httpbin.org/delay/3").send().await {
    Ok(r) => println!("unexpected ok {}", r.status()),
    Err(e) => println!("is_timeout={} is_connect={} msg={}", e.is_timeout(), e.is_connect(), e),
}

match client.get("http://127.0.0.1:1/nope").send().await {
    Ok(r) => println!("unexpected ok {}", r.status()),
    Err(e) => println!("is_connect={} is_timeout={} msg={}", e.is_connect(), e.is_timeout(), e),
}

match client.get("https://httpbin.org/html").send().await?.json::<Todo>().await {
    Ok(t) => println!("unexpected {:?}", t),
    Err(e) => println!("is_decode={} msg={}", e.is_decode(), e),
}
```

Output:

```
is_timeout=true is_connect=false msg=error sending request for url (https://httpbin.org/delay/3)
is_connect=true is_timeout=false msg=error sending request for url (http://127.0.0.1:1/nope)
is_decode=true msg=error decoding response body for url (https://httpbin.org/html)
```

| Predicate | Means | Retry? |
|---|---|---|
| `is_timeout()` | deadline hit | yes, with backoff |
| `is_connect()` | TCP/TLS/DNS failed | yes |
| `is_status()` | 4xx/5xx after `error_for_status` | only 5xx and 429 |
| `is_decode()` | body did not match the type | no — a bug or a contract change |
| `is_redirect()` | redirect policy hit | no |
| `is_body()` | sending the body failed | maybe |

The top-level `Display` is terse; `e.source()` has the detail, which is what
`anyhow` prints when you add context:

```rust
let res = client.get(url).send().await
    .with_context(|| format!("calling {url}"))?;
```

## Retries with backoff

Retry **idempotent** requests only, with exponential backoff and a cap.

```rust
async fn with_retry(client: &Client, url: &str, attempts: u32) -> Result<StatusCode, reqwest::Error> {
    let mut delay = Duration::from_millis(100);
    for i in 1..=attempts {
        match client.get(url).send().await {
            Ok(r) if r.status().is_server_error() && i < attempts => {
                println!("  attempt {i}: {} -> retrying in {:?}", r.status(), delay);
            }
            Ok(r) => return Ok(r.status()),
            Err(e) if e.is_timeout() && i < attempts => {
                println!("  attempt {i}: timeout -> retrying in {:?}", delay);
            }
            Err(e) => return Err(e),
        }
        tokio::time::sleep(delay).await;
        delay *= 2;
    }
    unreachable!()
}

println!("final: {:?}", with_retry(&client, "https://httpbin.org/status/503", 3).await?);
```

Output:

```
  attempt 1: 503 Service Unavailable -> retrying in 100ms
  attempt 2: 503 Service Unavailable -> retrying in 200ms
final: 503
```

Rules:

- **Only retry what is safe to repeat.** GET, PUT, DELETE are idempotent; a bare
  POST is not — retrying "create order" can create two. Add an idempotency key
  if you must retry one.
- **Retry 5xx, 429, timeouts, and connect errors. Never retry 4xx** — the
  request is wrong and will stay wrong.
- **Honour `Retry-After`** on a 429 before falling back to your own delay.
- **Add jitter** (`delay + random(0..delay/2)`) so a fleet of clients does not
  synchronize into a thundering herd.
- **Cap total attempts and total time**, or a retry loop inside a handler blows
  past your own response deadline.

Rather than hand-rolling, `reqwest-retry` (with `reqwest-middleware`) or
`tower::retry` gives you this as a layer, including jitter.

## Streaming responses

For a large download, don't buffer the whole body.

```rust
use futures_util::StreamExt;

let mut stream = client.get("https://httpbin.org/stream/3").send().await?.bytes_stream();
let mut chunks = 0;
let mut total = 0;
while let Some(c) = stream.next().await {
    let c = c?;
    chunks += 1;
    total += c.len();
}
println!("{chunks} chunk(s), {total} bytes total");
```

Output:

```
4 chunk(s), 720 bytes total
```

Needs `features = ["stream"]` and the `futures-util` crate for `StreamExt`.
Writing to a file as it arrives keeps memory flat regardless of size:

```rust
let mut file = tokio::fs::File::create("out.bin").await?;
let mut stream = res.bytes_stream();
while let Some(chunk) = stream.next().await {
    tokio::io::AsyncWriteExt::write_all(&mut file, &chunk?).await?;
}
```

Use this whenever the body could exceed a few megabytes. `.bytes()` on a 2GB
download allocates 2GB.

## Mocking upstream HTTP in tests

```bash
cargo add --dev wiremock
```

`wiremock` starts a real HTTP server on a random port and you point the client
at it — no trait abstraction, no mock injection. Keep the base URL a parameter,
and the production code is unchanged:

```rust
async fn fetch_rate(client: &Client, base_url: &str, cur: &str) -> Result<Rate, reqwest::Error> {
    client
        .get(format!("{base_url}/rates"))
        .query(&[("base", cur)])
        .header("x-api-key", "k123")
        .send().await?
        .error_for_status()?
        .json().await
}
```

The happy path, asserting what was sent as well as what came back:

```rust
use wiremock::matchers::{header, method, path, query_param};
use wiremock::{Mock, MockServer, ResponseTemplate};

#[tokio::test]
async fn returns_parsed_rate() {
    let server = MockServer::start().await;

    Mock::given(method("GET"))
        .and(path("/rates"))
        .and(query_param("base", "EUR"))
        .and(header("x-api-key", "k123"))
        .respond_with(ResponseTemplate::new(200).set_body_json(serde_json::json!({
            "base": "EUR", "usd": 1.09
        })))
        .expect(1)
        .mount(&server)
        .await;

    let rate = fetch_rate(&Client::new(), &server.uri(), "EUR").await.unwrap();
    assert_eq!(rate, Rate { base: "EUR".into(), usd: 1.09 });
}
```

`.expect(1)` is verified when the `MockServer` is dropped: if the request never
arrived, or arrived twice, the test fails. That makes the mock an assertion on
the request, not just a canned response.

The failure paths are the reason to use a mock at all — you cannot ask a real
API to return a 500 on demand.

```rust
#[tokio::test]
async fn propagates_upstream_500() {
    let server = MockServer::start().await;
    Mock::given(method("GET"))
        .respond_with(ResponseTemplate::new(500))
        .mount(&server).await;

    let err = fetch_rate(&Client::new(), &server.uri(), "EUR").await.unwrap_err();
    assert!(err.is_status());
    assert_eq!(err.status().map(|s| s.as_u16()), Some(500));
}

#[tokio::test]
async fn slow_upstream_times_out() {
    let server = MockServer::start().await;
    Mock::given(method("GET"))
        .respond_with(ResponseTemplate::new(200).set_delay(Duration::from_secs(2)))
        .mount(&server).await;

    let client = Client::builder().timeout(Duration::from_millis(200)).build().unwrap();
    let err = fetch_rate(&client, &server.uri(), "EUR").await.unwrap_err();
    assert!(err.is_timeout());
}
```

And a request matching no mock gets a 404, which catches a typo'd path:

```rust
#[tokio::test]
async fn no_matching_mock_gives_404() {
    let server = MockServer::start().await;
    Mock::given(method("GET"))
        .and(path("/something-else"))
        .respond_with(ResponseTemplate::new(200))
        .mount(&server).await;

    let err = fetch_rate(&Client::new(), &server.uri(), "EUR").await.unwrap_err();
    assert_eq!(err.status().map(|s| s.as_u16()), Some(404));
}
```

```console
$ cargo test --test wiremock_demo
running 4 tests
test no_matching_mock_gives_404 ... ok
test propagates_upstream_500 ... ok
test returns_parsed_rate ... ok
test slow_upstream_times_out ... ok

test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.24s
```

Matchers worth knowing: `method`, `path`, `path_regex`, `query_param`,
`header`, `header_exists`, `body_json`, `body_string`, `any`. Response helpers:
`set_body_json`, `set_body_string`, `set_body_bytes`, `insert_header`,
`set_delay`.

`MockServer::start()` picks a free port, so tests run in parallel without
coordination — and each test gets its own server, so mounted mocks never leak
between them. Lesson 17.13 uses the same technique inside full axum integration
tests.

## Calling an upstream from a handler

```rust
#[derive(Clone)]
struct AppState { http: Client }

async fn proxy(State(st): State<AppState>, Path(id): Path<u32>) -> Result<Json<Todo>, AppError> {
    let todo: Todo = st.http
        .get(format!("https://jsonplaceholder.typicode.com/todos/{id}"))
        .send().await
        .map_err(|e| AppError::Upstream(e.to_string()))?
        .error_for_status()
        .map_err(|e| match e.status() {
            Some(StatusCode::NOT_FOUND) => AppError::NotFound("todo", id.to_string()),
            _ => AppError::Upstream(e.to_string()),
        })?
        .json().await
        .map_err(|e| AppError::Upstream(e.to_string()))?;
    Ok(Json(todo))
}
```

Two points. The client comes from state, built once at startup. And an upstream
failure maps to **your** error taxonomy (lesson 17.6) — an upstream 500 should
probably be a 502 Bad Gateway from you, not a 500, and an upstream timeout a 504.
Never leak an upstream error message to your clients; log it and return your own.

Also set the client's `timeout` shorter than your own `TimeoutLayer`, so you fail
with a sensible error instead of being cut off mid-request.

## Quick Reference

| Need | Form |
|---|---|
| Build once | `Client::builder().timeout(d).build()?`, store in state |
| GET | `client.get(url).send().await?` |
| POST JSON | `.post(url).json(&body)` |
| POST form | `.post(url).form(&pairs)` |
| Query string | `.query(&[("k","v")])` or `.query(&struct)` |
| Header | `.header("k", "v")` |
| Bearer / basic | `.bearer_auth(t)` / `.basic_auth(u, Some(p))` |
| Parse body | `.json::<T>().await?` |
| Raw body | `.text().await?` / `.bytes().await?` |
| Status | `res.status()`, `.is_success()` |
| 4xx/5xx as error | `.error_for_status()?` |
| Classify | `e.is_timeout()`, `is_connect()`, `is_status()`, `is_decode()` |
| Stream | `.bytes_stream()` + `StreamExt::next` (`stream` feature) |
| Upload | `.multipart(Form::new().part(..))` (`multipart` feature) |
| Mock in tests | `wiremock::MockServer` + `Mock::given(..).respond_with(..)` |
| Assert the call happened | `.expect(1)` on the mock |

## Common Mistakes

**`Client::new()` per request.** A new connection pool and TLS handshake each
time. Build one, clone it.

**No `timeout` on the client.** The default is unlimited; a hung upstream hangs
your handler.

**Treating a 404 as an error automatically.** It is an `Ok(Response)`. Call
`error_for_status()` or match the status.

**Reading the status after `.json()`.** The body methods consume the response.
Status and headers first.

**Retrying a POST.** Not idempotent — you may create two of something. Retry
GET/PUT/DELETE, or use an idempotency key.

**Retrying 4xx.** The request is wrong; repeating it changes nothing.

**`format!`-ing a URL with user data.** Use `.query()` so values are encoded.

**Missing the `query` or `form` feature in reqwest 0.13.** The methods just do
not exist and the error does not mention features.

**`.bytes()` on a huge download.** Buffers the whole body. Use `.bytes_stream()`.

**Leaking an upstream error message to your clients.** Log it; return your own
502/504.

## Exercises

1. Build one `Client` with a 5-second timeout and fetch
   `https://jsonplaceholder.typicode.com/users/1` into a struct with `name` and
   `email`. Print both.
2. POST to `https://httpbin.org/post` with `.json()` and print the `json` field
   of the echo so you can see exactly what was sent.
3. Request `https://httpbin.org/status/500` and handle it three ways: raw status
   match, `error_for_status()`, and `?` through an `AppError`.
4. Set a 300ms timeout, call `https://httpbin.org/delay/2`, and assert
   `e.is_timeout()`. Then raise the timeout and confirm it succeeds.
5. Write `with_retry` that retries on 5xx and timeouts with jitter, caps at four
   attempts, and honours a `Retry-After` header. Test it against
   `https://httpbin.org/status/503`.
6. Write a `wiremock` test suite for a function calling an upstream: one test for
   200 with a JSON body, one for 500, one for a timeout via `set_delay`, and one
   asserting the request carried the right query param and header with
   `.expect(1)`.

---

[← Course Index](../README.md)
