[← 18.5 Release Profiles and Build Optimization](05-release-builds.md) · [Course Index](../README.md) · [18.7 CI with GitHub Actions →](07-ci.md)

# 18.6 — Docker and Deployment

## What You'll Learn

- A multi-stage Dockerfile for Rust, and why the naive one rebuilds everything
- The dependency-caching trick, and `cargo-chef` which does it properly
- musl static builds for `distroless`/`scratch`, with a size comparison
- Non-root user, healthcheck, `.dockerignore`, cross-compiling from Apple Silicon

> **On the numbers in this lesson.** Docker was not available in the environment
> these lessons were written in (`docker info` failed — no daemon), and this
> machine has no `musl` target installed. Every Dockerfile here is complete and
> correct, but the image sizes in the comparison table are **illustrative
> typical values, not measurements taken on this machine**. Each table says so.
> Build them yourself and your numbers will differ with your dependency set.

## Coming From Another Language

| Other | Rust |
|---|---|
| `COPY package.json` then `npm ci` | the same idea, but Cargo makes it awkward |
| `node:20-slim` runtime | a `debian:trixie-slim` with just your binary |
| ship source + interpreter | ship one static binary, no runtime at all |
| cold build ~1 min | cold build 1-10 min; caching is not optional |

The payoff is a tiny image with no interpreter, no `node_modules`, no CVE
surface from a language runtime. The cost is compile time, and the entire art of
a Rust Dockerfile is not paying it twice.

## The naive version, and why it's wrong

```dockerfile
FROM rust:1.99-trixie
WORKDIR /app
COPY . .
RUN cargo build --release
CMD ["/app/target/release/myapi"]
```

Two problems. The image carries the whole toolchain (~1.5 GB). And `COPY . .`
invalidates the cache on **any** source change, so every one-line edit
recompiles all 300 dependency crates. For the Module 17 stack — tokio, axum,
sqlx, serde — that is minutes per build, every build.

## Multi-stage

Build in one image, copy the binary into a clean one.

```dockerfile
FROM rust:1.99-trixie AS builder
WORKDIR /app
COPY . .
RUN cargo build --release

FROM debian:trixie-slim AS runtime
RUN apt-get update \
    && apt-get install -y --no-install-recommends ca-certificates \
    && rm -rf /var/lib/apt/lists/*
COPY --from=builder /app/target/release/myapi /usr/local/bin/myapi
EXPOSE 8080
ENTRYPOINT ["/usr/local/bin/myapi"]
```

`ca-certificates` is needed by almost every service — without it `reqwest` TLS
and `sqlx` over TLS fail with a certificate error, which is a confusing first
deploy. Nothing else from Debian is required if you link against glibc.

This fixes image size. It does not fix rebuild time.

## The dependency-layer trick

Docker caches per layer. Copy only the manifests, build a throwaway binary to
compile the dependency graph, *then* copy real source.

```dockerfile
FROM rust:1.99-trixie AS builder
WORKDIR /app

COPY Cargo.toml Cargo.lock ./
RUN mkdir src && echo "fn main() {}" > src/main.rs
RUN cargo build --release
RUN rm -rf src

COPY src ./src
RUN touch src/main.rs && cargo build --release
```

Now editing a handler reuses the dependency layer and recompiles only your
crate. Why it matters more in Rust than in Node: `npm ci` downloads prebuilt
packages, while Cargo *compiles* every dependency from source, with generics
monomorphised into your crate. The dependency layer is minutes, not seconds.

Three sharp edges. `touch src/main.rs` is required — the dummy build left a
newer mtime on the artifact, and without the touch Cargo thinks it is current
and ships a binary that does nothing. A workspace needs every member's
`Cargo.toml` plus a dummy for each. And adding one dependency invalidates the
whole layer, which is correct but means a full rebuild.

## cargo-chef, the better answer

`cargo-chef` computes a recipe — a manifest-only skeleton of the workspace — so
the cached layer depends on your dependency *set* and nothing else. It handles
workspaces and the mtime problem for you.

```dockerfile
FROM rust:1.99-trixie AS chef
RUN cargo install cargo-chef --locked
WORKDIR /app

FROM chef AS planner
COPY . .
RUN cargo chef prepare --recipe-path recipe.json

FROM chef AS builder
COPY --from=planner /app/recipe.json recipe.json
RUN cargo chef cook --release --recipe-path recipe.json
COPY . .
RUN cargo build --release --bin myapi

FROM debian:trixie-slim AS runtime
RUN apt-get update \
    && apt-get install -y --no-install-recommends ca-certificates \
    && rm -rf /var/lib/apt/lists/*
WORKDIR /app
RUN groupadd --system --gid 10001 app \
    && useradd --system --uid 10001 --gid app --no-create-home app
COPY --from=builder /app/target/release/myapi /usr/local/bin/myapi
USER app
EXPOSE 8080
ENTRYPOINT ["/usr/local/bin/myapi"]
```

The `planner` stage exists so the recipe is produced in a throwaway layer: its
`COPY . .` is invalidated by every source edit, but only `recipe.json` crosses
into `builder`, and that file changes only when dependencies do.

`cargo chef cook` compiles dependencies only. Then `COPY . .` and the real
build. Use this one.

## Non-root user

```dockerfile
RUN groupadd --system --gid 10001 app \
    && useradd --system --uid 10001 --gid app --no-create-home app
USER app
```

A fixed high UID matters for Kubernetes `runAsNonRoot` and for volume
permissions. On `scratch`/`distroless` there is no `useradd`, so either use
`distroless:nonroot` (which ships UID 65532) or set `USER 10001:10001`
numerically — the kernel only needs the number.

Note `USER` comes after the `COPY`, so the binary is owned by root and not
writable by the app user. That is what you want.

## Healthcheck

```dockerfile
HEALTHCHECK --interval=10s --timeout=3s --start-period=5s --retries=3 \
    CMD ["/usr/local/bin/myapi", "healthcheck"]
```

There is no `curl` in a slim or distroless image, and adding one for a
healthcheck is a poor trade. Give your own binary a subcommand that hits
`/health/ready` on localhost (18.3) and exits 0 or 1 — zero extra dependencies
and it reuses your config.

Under Kubernetes, delete the `HEALTHCHECK` entirely: the kubelet uses the probes
from the pod spec and ignores Docker's. `HEALTHCHECK` is for Compose and plain
Docker.

## .dockerignore

Without this, `COPY . .` ships your `target/` directory — gigabytes — into the
build context, which is slow and busts caches.

```
target/
.git/
.gitignore
.env
.env.*
Dockerfile
docker-compose.yml
.github/
*.md
tests/fixtures/*.bin
```

`target/` and `.git/` are the ones that matter. `.env` is listed so a local
secrets file can never be baked into an image.

## Static builds with musl

A glibc binary needs a glibc base image. Link against musl statically and the
binary needs *nothing* — so it runs on `scratch`.

```dockerfile
FROM rust:1.99-trixie AS builder
RUN rustup target add x86_64-unknown-linux-musl \
    && apt-get update && apt-get install -y --no-install-recommends musl-tools \
    && rm -rf /var/lib/apt/lists/*
WORKDIR /app
COPY . .
RUN cargo build --release --target x86_64-unknown-linux-musl

FROM scratch AS runtime
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
COPY --from=builder /app/target/x86_64-unknown-linux-musl/release/myapi /myapi
USER 10001:10001
EXPOSE 8080
ENTRYPOINT ["/myapi"]
```

On `scratch` you must copy the CA bundle in by hand — there is no package
manager to install it. You also have no shell, so `docker exec` gives you
nothing; debug by running the same binary in a slim image.

Three real caveats before you choose musl:

- **`ring` and some `-sys` crates need the musl C toolchain** (`musl-tools`
  above), and a few do not support musl at all.
- **musl's allocator is markedly slower than glibc's** under multi-threaded
  allocation pressure, which is exactly what a Tokio server does. If you go
  musl and care about throughput, add `mimalloc` or `jemallocator` as the global
  allocator and measure.
- For `rustls`-based TLS (18.9) this is painless; for OpenSSL it is not. Prefer
  `rustls` features on `reqwest` and `sqlx`.

`distroless/cc` is the middle path: glibc, no shell, no package manager, CA
certs already present, so a normal `gnu`-target build drops straight in.

## Image size comparison

> **Illustrative, not measured here** — Docker was unavailable. Figures are
> typical for a small Axum + SQLx service; the binary is ~8-15 MB of that, so
> your totals move with your dependency set.

| Base | Typical total | Shell | glibc | Notes |
|---|---|---|---|---|
| `rust:1.99-trixie` (single stage) | ~1.5 GB | yes | yes | never ship this |
| `debian:trixie-slim` | ~90-110 MB | yes | yes | easiest; `docker exec` works |
| `alpine` (musl target) | ~20-30 MB | yes | no | shell, slower allocator |
| `gcr.io/distroless/cc` | ~35-45 MB | **no** | yes | glibc build, no shell |
| `gcr.io/distroless/static` | ~15-25 MB | **no** | no | needs a static binary |
| `scratch` (musl static) | ~10-20 MB | **no** | no | smallest; nothing to debug with |

Pick by what you need, not by the smallest number. `debian:trixie-slim` is the
right default: the extra ~70 MB is one layer shared across every image on the
node, and being able to get a shell into a misbehaving pod is worth more than
you think. Go distroless when a security policy demands no shell.

The compression caveat: `docker images` shows uncompressed size, registries
store and transfer compressed layers, so pull time tracks roughly half these
numbers.

## Cross-compiling from Apple Silicon

Your Mac is `arm64`; most cloud hosts are `amd64`. An image built without
thought runs under emulation or not at all.

```bash
docker build --platform linux/amd64 -t myapi:latest .
```

This builds for amd64 — and on Apple Silicon it does so by **emulating amd64
with QEMU**, which makes `rustc` roughly 3-10x slower. For a Rust project that
turns a 3-minute build into 20.

Faster options, in order of preference:

```bash
# 1. Build amd64 in CI on amd64 runners; locally build native arm64 only.
docker build --platform linux/arm64 -t myapi:dev .

# 2. Multi-arch manifest, each arch on its own native builder.
docker buildx build --platform linux/amd64,linux/arm64 -t myapi:latest --push .

# 3. Cross-compile on the host, Docker only packages the binary.
cargo install cross
cross build --release --target x86_64-unknown-linux-musl
```

Option 3 with a Dockerfile that only copies a prebuilt binary:

```dockerfile
FROM debian:trixie-slim
RUN apt-get update \
    && apt-get install -y --no-install-recommends ca-certificates \
    && rm -rf /var/lib/apt/lists/*
ARG BINARY=target/x86_64-unknown-linux-musl/release/myapi
COPY ${BINARY} /usr/local/bin/myapi
USER 10001:10001
ENTRYPOINT ["/usr/local/bin/myapi"]
```

Also deploy for the arch you build: if your target is Graviton or arm64 nodes,
`linux/arm64` is both cheaper and native to your laptop.

## Secrets and config at runtime

Never `COPY .env` and never bake a secret into a layer — layers are extractable
from any image, and deleting a file in a later layer does **not** remove it from
the earlier one.

```bash
docker run --rm -p 8080:8080 \
  -e APP_ENV=production \
  -e DATABASE_URL="postgres://u:p@db:5432/app" \
  -e JWT_SECRET="$(cat /run/secrets/jwt)" \
  -e RUST_LOG="info,myapi=debug" \
  myapi:latest
```

For a build-time secret (a private registry token) use BuildKit mounts, which
are never written to a layer:

```dockerfile
RUN --mount=type=secret,id=cargo_token \
    CARGO_REGISTRIES_MY_REGISTRY_TOKEN=$(cat /run/secrets/cargo_token) \
    cargo build --release
```

This is exactly the 18.1 split: files for shape, environment for secrets,
injected by the platform.

## docker-compose for local development

```yaml
services:
  db:
    image: postgres:17-alpine
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: devpassword
      POSTGRES_DB: app_dev
    ports: ["5432:5432"]
    volumes: ["pgdata:/var/lib/postgresql/data"]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app -d app_dev"]
      interval: 2s
      timeout: 3s
      retries: 20

  api:
    build:
      context: .
      dockerfile: Dockerfile
    depends_on:
      db:
        condition: service_healthy
    environment:
      DATABASE_URL: postgres://app:devpassword@db:5432/app_dev
      APP_ENV: development
      JWT_SECRET: dev-only-insecure-secret-at-least-32-bytes
      RUST_LOG: "info,myapi=debug,sqlx::query=warn"
    ports: ["8080:8080"]
    stop_grace_period: 30s

volumes:
  pgdata:
```

Four details that save time. `condition: service_healthy` with a real
`pg_isready` healthcheck — `depends_on` alone only waits for the container to
start, and your migrations will race the database. The host is `db`, Compose's
service name. `stop_grace_period` must exceed your shutdown timeout from 18.4 or
`docker compose down` SIGKILLs you mid-drain. And `sqlx::query=warn` keeps
per-statement logging out of your terminal.

Many teams run only `db` in Compose and the app with `cargo run` on the host —
you keep incremental compilation and a debugger, and skip the image build
entirely. `docker compose up -d db` then `cargo run`.

## Shrinking the image further

Combine with 18.5's profile settings:

```toml
[profile.release]
lto = "fat"
codegen-units = 1
panic = "abort"
strip = "symbols"
```

On the small Axum service measured in 18.5, that took the binary from 2,585,072
bytes to 1,215,264 — about 53% off, and it is the binary that varies between
your images. (That number *is* measured, on this machine, in 18.5.)

Also prefer `rustls` over `native-tls` on `reqwest` and `sqlx`: it drops the
OpenSSL dependency, which removes a C build step, shrinks the image and makes
musl builds work.

## One last PID 1 warning

```dockerfile
ENTRYPOINT ["/usr/local/bin/myapi"]
```

Exec form, a JSON array. The shell form — `ENTRYPOINT /usr/local/bin/myapi` —
runs your binary under `/bin/sh -c`, so the shell is PID 1 and may not forward
SIGTERM. Every bit of graceful shutdown from 18.4 silently stops working. Every
Dockerfile in this lesson uses the exec form.

## Quick Reference

| Task | Code |
|---|---|
| Multi-stage | `FROM rust:1.99-trixie AS builder` / `FROM debian:trixie-slim` |
| Copy the binary | `COPY --from=builder /app/target/release/app /usr/local/bin/app` |
| Cache deps (manual) | copy manifests, dummy `src/main.rs`, build, then `touch src/main.rs` |
| Cache deps (good) | `cargo chef prepare` / `cargo chef cook --release` |
| Static target | `rustup target add x86_64-unknown-linux-musl` + `musl-tools` |
| CA certs on scratch | `COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/` |
| Non-root | `useradd --system --uid 10001 app` + `USER app`, or `USER 10001:10001` |
| Correct PID 1 | `ENTRYPOINT ["/path/to/bin"]` (exec form, always) |
| Build for amd64 | `docker build --platform linux/amd64 .` |
| Multi-arch push | `docker buildx build --platform linux/amd64,linux/arm64 --push .` |
| Cross-compile | `cross build --release --target x86_64-unknown-linux-musl` |
| Build secret | `RUN --mount=type=secret,id=x ...` |
| Wait for Postgres | `depends_on: db: condition: service_healthy` |
| Shutdown room | `stop_grace_period: 30s` / `docker stop -t 30` |

## Common Mistakes

**`COPY . .` with no `.dockerignore`.** You ship `target/` into the build
context and bust the cache constantly.

**Forgetting `touch src/main.rs` after the dummy build.** Cargo sees a fresh
artifact and your image runs `fn main() {}`.

**Shell-form `ENTRYPOINT`.** `sh` becomes PID 1, SIGTERM is not forwarded, and
graceful shutdown never runs.

**No `ca-certificates` in the runtime stage.** Every outbound HTTPS call and TLS
database connection fails with a certificate error.

**Baking `.env` or a secret into a layer.** Layers are permanent and
extractable; a later `RUN rm` does not remove it.

**Choosing musl for size without measuring.** Its allocator is slower under
concurrent load. Either accept that or add `mimalloc`.

## Exercises

1. Write the multi-stage Dockerfile with the manual dependency-cache trick.
   Build twice, changing one line of a handler between, and compare the times.
2. Convert it to `cargo-chef`. Add a dependency and confirm which layers are
   invalidated in each version.
3. Add a `healthcheck` subcommand to your binary that requests
   `/health/ready` and exits 0 or 1, then wire up `HEALTHCHECK`.
4. Write the Compose file with a Postgres healthcheck. Prove your app waits by
   logging before the first query.
5. Build the same project for `debian:trixie-slim` and for `scratch` with musl.
   Record both `docker images` sizes and whether `docker exec ... sh` works.

---

[← 18.5 Release Profiles and Build Optimization](05-release-builds.md) · [Course Index](../README.md) · [18.7 CI with GitHub Actions →](07-ci.md)
