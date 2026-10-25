[← Course Index](../README.md)

# 18.1 — Configuration and Environment

## What You'll Learn

- `std::env::var` and the `Result` it hands you
- `dotenvy` for `.env` in development, and why shell env still wins
- The `config` crate: defaults, then a file, then environment overrides
- Typed config structs, `#[serde(default)]`, validation, and secret handling

## Coming From Another Language

| Other | Rust |
|---|---|
| Node `process.env.PORT` (undefined if unset) | `env::var("PORT")` → `Result<String, VarError>` |
| Python `os.environ["PORT"]` (raises) | `env::var("PORT")?` |
| Node `dotenv` package | `dotenvy` crate |
| `config`/`pydantic-settings` | `config` crate + a `serde` struct |

The difference is that nothing is implicit. `env::var` returns a `Result`, every
value arrives as a `String`, and you convert it into a typed struct once at
startup. After that the rest of the program reads fields, not strings.

## env::var returns a Result

```rust
use std::env;

fn main() {
    match env::var("DATABASE_URL") {
        Ok(v) => println!("got: {v}"),
        Err(e) => println!("err: {e}"),
    }
    println!("{:?}", env::var("NOPE_NOT_SET"));
}
```

Output with nothing set:

```
err: environment variable not found
Err(NotPresent)
```

Output with `DATABASE_URL=postgres://localhost/mydb`:

```
got: postgres://localhost/mydb
Err(NotPresent)
```

`VarError` has two variants: `NotPresent` and `NotUnicode(OsString)`. The second
only appears when the value is not valid UTF-8; use `env::var_os` if you need
the raw bytes.

## Defaults and parsing

Everything is a `String`, so parsing is a second fallible step.

```rust
let port: u16 = env::var("PORT")
    .unwrap_or_else(|_| "8080".into())
    .parse()
    .unwrap();
println!("port = {port}");

let workers = env::var("WORKERS")
    .ok()
    .and_then(|s| s.parse::<usize>().ok())
    .unwrap_or(4);
println!("workers = {workers}");
```

Output with nothing set:

```
port = 8080
workers = 4
```

The two differ in failure behaviour. `PORT=abc` panics on the first:

```
thread 'main' (13917450) panicked at examples/c1.rs:10:80:
called `Result::unwrap()` on an `Err` value: ParseIntError { kind: InvalidDigit }
```

The `workers` form silently swallows a typo and uses `4`. Neither is what you
want in production — you want a clear message, which is what the rest of this
lesson builds toward.

Catching the parse error yourself:

```rust
match env::var("PORT").unwrap_or_default().parse::<u16>() {
    Ok(p) => println!("parsed {p}"),
    Err(e) => println!("parse failed: {e}"),
}
```

Output with nothing set:

```
parse failed: cannot parse integer from empty string
```

## Listing variables

```rust
for (k, v) in env::vars().filter(|(k, _)| k.starts_with("APP_")) {
    println!("{k}={v}");
}
```

Output with `APP_PORT=9000 APP_NAME=api`:

```
APP_NAME=api
APP_PORT=9000
```

`env::vars()` panics on non-UTF-8 entries; `env::vars_os()` does not.

## dotenvy for development

`.env` in the project root:

```
DATABASE_URL=postgres://dev:dev@localhost/app_dev
JWT_SECRET=dev-only-not-a-real-secret
APP__SERVER__PORT=4000
```

```rust
fn main() {
    match dotenvy::dotenv() {
        Ok(path) => println!("loaded {}", path.display()),
        Err(e) => println!("no .env: {e}"),
    }
    println!("{:?}", std::env::var("DATABASE_URL"));
    println!("{:?}", std::env::var("JWT_SECRET"));
}
```

Output:

```
loaded /private/tmp/.../w18/.env
Ok("postgres://dev:dev@localhost/app_dev")
Ok("dev-only-not-a-real-secret")
```

A variable already in the shell environment is **not** overwritten. Running the
same binary with `DATABASE_URL=postgres://from-shell/x`:

```
loaded /private/tmp/.../w18/.env
Ok("postgres://from-shell/x")
Ok("dev-only-not-a-real-secret")
```

That is the behaviour you want: `.env` is a development convenience, and the
real environment always takes priority. Use `dotenvy::dotenv_override()` if you
genuinely need the opposite.

`.env` goes in `.gitignore`. Commit a `.env.example` with the keys and dummy
values instead.

## The config crate: layered sources

`Cargo.toml`:

```toml
config = "0.15.27"
serde = { version = "1.0.229", features = ["derive"] }
```

`config/default.toml`:

```toml
log_level = "info"

[server]
host = "0.0.0.0"
port = 8080

[database]
url = "postgres://localhost/app"
max_connections = 5
```

`config/production.toml` only states the differences:

```toml
log_level = "warn"

[server]
port = 80

[database]
max_connections = 50
```

```rust
use config::{Config, Environment, File};
use serde::Deserialize;

#[derive(Debug, Deserialize)]
struct ServerConfig {
    host: String,
    port: u16,
}

#[derive(Debug, Deserialize)]
struct DatabaseConfig {
    url: String,
    max_connections: u32,
    #[serde(default = "default_timeout")]
    connect_timeout_secs: u64,
}

fn default_timeout() -> u64 { 30 }

#[derive(Debug, Deserialize)]
struct AppConfig {
    log_level: String,
    server: ServerConfig,
    database: DatabaseConfig,
}

fn load(env: &str) -> Result<AppConfig, config::ConfigError> {
    Config::builder()
        .add_source(File::with_name("config/default"))
        .add_source(File::with_name(&format!("config/{env}")).required(false))
        .add_source(Environment::with_prefix("APP").separator("__"))
        .build()?
        .try_deserialize()
}
```

Later sources win over earlier ones. Output for `development` (no such file, so
only defaults apply):

```
env = development
AppConfig {
    log_level: "info",
    server: ServerConfig {
        host: "0.0.0.0",
        port: 8080,
    },
    database: DatabaseConfig {
        url: "postgres://localhost/app",
        max_connections: 5,
        connect_timeout_secs: 30,
    },
}
```

`connect_timeout_secs` appears in no file — `#[serde(default = "...")]` supplied
it. With `APP_ENV=production`:

```
env = production
AppConfig {
    log_level: "warn",
    server: ServerConfig {
        host: "0.0.0.0",
        port: 80,
    },
    database: DatabaseConfig {
        url: "postgres://localhost/app",
        max_connections: 50,
        connect_timeout_secs: 30,
    },
}
```

`host` and `url` fell through from `default.toml`; the three keys in
`production.toml` replaced their defaults.

## Nested keys from environment variables

`.separator("__")` maps a double underscore to a nesting level, so
`APP__SERVER__PORT` reaches `server.port`.

With `APP_ENV=production APP__SERVER__PORT=9999 APP__LOG_LEVEL=debug`:

```
log_level: "debug",
server: ServerConfig {
    host: "0.0.0.0",
    port: 9999,
},
```

Environment is the last source, so it beats both files. That is what a container
orchestrator sets, and it is the layer you deploy with.

## #[serde(default)] variants

| Attribute | Effect when the key is absent |
|---|---|
| `#[serde(default)]` | `Default::default()` for the field type |
| `#[serde(default = "fn_name")]` | calls `fn_name()` |
| `Option<T>` field | `None`, no attribute needed |
| nothing | hard error |

Without one of these, a missing key is fatal — which is correct for
`database.url` and wrong for `connect_timeout_secs`.

## Config errors are specific

Each of these is real `ConfigError` output from the loader above.

A required field with no value anywhere:

```
ERROR: missing configuration field "jwt_secret"
```

A value that won't parse:

```
ERROR: invalid type: string "notanumber", expected an integer for key `server.port` in the environment
```

Out of range for the declared type:

```
ERROR: invalid type: 64-bit unsigned integer `99999`, expected an unsigned 16 bit integer for key `server.port`
```

A `required(true)` file that doesn't exist:

```
ERROR: configuration file "config/default" not found
```

Note the last line of the second message — `in the environment` tells you which
layer the bad value came from. That is worth a lot at 3am.

## Fail fast at startup

Load and validate before you bind a port or open a pool. A misconfigured service
that exits immediately is far better than one that accepts traffic and then
fails on every request.

```rust
fn main() {
    let config = match load() {
        Ok(c) => c,
        Err(e) => {
            eprintln!("fatal: invalid configuration: {e}");
            std::process::exit(1);
        }
    };
    println!("starting on {}:{}", config.server.host, config.server.port);
}
```

Output, good config:

```
starting on 0.0.0.0:8080
```

Output with `APP__SERVER__PORT=xyz`, exit code 1:

```
fatal: invalid configuration: invalid type: string "xyz", expected an integer for key `server.port` in the environment
```

`std::process::exit(1)` rather than a panic: no backtrace noise, no
`thread 'main' panicked` prefix, just the message an operator needs.

## Secrets: a redacting newtype

The danger with config structs is `{:?}`. One `tracing::info!(?config)` and your
database password is in the log aggregator forever. Wrap secrets in a type whose
`Debug` lies.

```rust
use std::fmt;

#[derive(Deserialize, Clone)]
struct Secret(String);

impl Secret {
    fn expose(&self) -> &str { &self.0 }
}

impl fmt::Debug for Secret {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "[REDACTED]")
    }
}
```

`expose()` is deliberately ugly to type, so it shows up in review.

## A worked AppConfig for a web service

```rust
#[derive(Debug, Deserialize, Clone, Copy, PartialEq)]
#[serde(rename_all = "lowercase")]
enum Env { Development, Staging, Production }

#[derive(Debug, Deserialize)]
struct AppConfig {
    environment: Env,
    database_url: Secret,
    jwt_secret: Secret,
    port: u16,
    #[serde(default)]
    enable_signup: bool,
}
```

```rust
println!("{good:?}");
```

Output:

```
AppConfig { environment: Production, database_url: [REDACTED], jwt_secret: [REDACTED], port: 8080, enable_signup: false }
```

The environment is an `enum`, not a `String` — `match` on it is exhaustive, and
a typo like `APP_ENVIRONMENT=prodcution` fails at load instead of silently
behaving like development.

## Validating beyond types

Types catch shape. Rules need code.

```rust
#[derive(Debug, thiserror::Error)]
enum ConfigError {
    #[error("JWT_SECRET must be at least 32 bytes (got {0})")]
    WeakSecret(usize),
    #[error("DATABASE_URL must start with postgres:// (got {0:?})")]
    BadDatabaseScheme(String),
    #[error("signup cannot be enabled in production")]
    SignupInProduction,
}

impl AppConfig {
    fn validate(&self) -> Result<(), ConfigError> {
        if self.jwt_secret.expose().len() < 32 {
            return Err(ConfigError::WeakSecret(self.jwt_secret.expose().len()));
        }
        let url = self.database_url.expose();
        if !url.starts_with("postgres://") {
            return Err(ConfigError::BadDatabaseScheme(url.chars().take(12).collect()));
        }
        if self.environment == Env::Production && self.enable_signup {
            return Err(ConfigError::SignupInProduction);
        }
        Ok(())
    }
}
```

Output:

```
validate -> Ok(())
JWT_SECRET must be at least 32 bytes (got 5)
DATABASE_URL must start with postgres:// (got "mysql://u:p@")
signup cannot be enabled in production
```

Note the third message: validation can encode policy, not just syntax. And note
the second truncates the URL before putting it in an error — the error string
will be logged, and the full URL has a password in it.

Module 08 covers `thiserror`; 17.10 covers request-level validation, which is a
different problem with the same shape.

## figment as an alternative

`figment` does the same layering with a different API, and is what Rocket uses.

```rust
use figment::{Figment, providers::{Env, Format, Serialized, Toml}};

let cfg: Result<Cfg, _> = Figment::from(Serialized::defaults(Cfg::default()))
    .merge(Toml::file("config/default.toml"))
    .merge(Env::prefixed("APP_"))
    .extract();
```

Output with `APP_PORT=7000`:

```
Ok(
    Cfg {
        host: "0.0.0.0",
        port: 7000,
        log_level: "info",
    },
)
```

It takes defaults from a `Default` impl rather than a file, which some people
prefer. Either crate is fine; pick one.

## Per-environment profiles

```
config/
  default.toml      committed, safe values, no secrets
  development.toml  committed, local ports and a local DB
  staging.toml      committed
  production.toml   committed, no secrets
.env                gitignored, local secrets only
```

The rule: files hold shape and non-secret differences, environment holds secrets
and anything the deploy target decides. In production the environment comes from
your platform's secret store — Kubernetes `Secret`, AWS Secrets Manager, Vault —
injected as environment variables, so your Rust code does not change at all
between local and deployed.

Never commit a secret. If one lands in git, rotate it; deleting the commit does
not help, the value is in the reflog and in every clone.

## Quick Reference

| Task | Code |
|---|---|
| Read a var | `std::env::var("K")` → `Result<String, VarError>` |
| Read non-UTF-8 | `env::var_os("K")` → `Option<OsString>` |
| With default | `env::var("K").unwrap_or_else(\|_\| "d".into())` |
| Set for the process | `unsafe { env::set_var("K", "v") }` |
| Iterate | `env::vars()` / `env::vars_os()` |
| Load `.env` | `dotenvy::dotenv()?` |
| `.env` overrides shell | `dotenvy::dotenv_override()?` |
| Layered load | `Config::builder().add_source(..).build()?.try_deserialize()` |
| Optional file | `File::with_name("x").required(false)` |
| Env source | `Environment::with_prefix("APP").separator("__")` |
| Optional key | `#[serde(default)]` or `Option<T>` |
| Optional with value | `#[serde(default = "f")]` |
| Exit on bad config | `eprintln!(..); std::process::exit(1)` |

## Common Mistakes

**`env::var("X").unwrap()` in a handler.** It panics on request number one in a
new environment. Read config once at startup into a struct.

**Putting secrets in `config/production.toml`.** The file is committed. Secrets
belong in the environment, injected by your platform.

**A `String` field for everything.** `port: String` means every user re-parses
it and a typo surfaces at the point of use. Declare `u16` and let `serde` reject
garbage at startup.

**Deriving `Debug` on a config with raw secrets.** One `{:?}` leaks it. Use a
redacting newtype.

**Forgetting `.env` is development-only.** It is not loaded in your container
unless you copy it in, which you should not. Set real environment variables.

## Exercises

1. Write a `fn port() -> u16` reading `PORT`, defaulting to `3000`, that returns
   a clear error message rather than panicking on `PORT=abc`.
2. Build a `config/` directory with `default.toml` and `test.toml`, and a loader
   that picks the file from `APP_ENV`. Prove the `test.toml` values override.
3. Add a `rate_limit: Option<u32>` field and a `max_body_bytes: usize` field
   defaulting to `1_048_576`. Show both behave correctly when absent.
4. Wrap `jwt_secret` in a redacting `Secret` newtype and print the whole config
   with `{:#?}` to confirm nothing leaks.
5. Add a `validate()` that rejects `port < 1024` unless `environment` is
   `Development`. Make `main` exit with code 1 and a one-line message.

---

[← Course Index](../README.md)
