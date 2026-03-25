# Docker for Rust

## Multi-Stage Dockerfile (Production-Ready)

```dockerfile
FROM rust:1-slim AS builder
WORKDIR /app

RUN apt-get update && apt-get install -y \
    pkg-config libssl-dev \
    && rm -rf /var/lib/apt/lists/*

COPY Cargo.toml Cargo.lock ./
RUN mkdir src && echo "fn main() {}" > src/main.rs
RUN cargo build --release
RUN rm -rf src target/release/deps/myapp* \
    target/release/myapp*

COPY src ./src
RUN cargo build --release

FROM gcr.io/distroless/cc-debian12
COPY --from=builder /app/target/release/myapp /
EXPOSE 8080
ENTRYPOINT ["/myapp"]
```

The two-phase build trick:
1. Copy only `Cargo.toml` and `Cargo.lock`
2. Build with a dummy `main.rs` to cache dependencies
3. Copy real source and rebuild (only app code compiles)

---

## Builder Base Images

| Image | Size | Use When |
| --- | --- | --- |
| `rust:1-slim` | ~800MB | Default choice, Debian slim |
| `rust:1-bookworm` | ~1.4GB | Need system libs (OpenSSL, etc.) |
| `rust:1-alpine` | ~800MB | Building musl static binaries |

Prefer `rust:1-slim` unless you need specific system
libraries that require the full Debian image.

---

## Runtime Base Images

| Image | Size | Use When |
| --- | --- | --- |
| `scratch` | 0MB | Static binary (musl), no shell needed |
| `gcr.io/distroless/cc-debian12` | ~25MB | Dynamically linked, no shell |
| `debian:bookworm-slim` | ~80MB | Need shell for debugging |
| `alpine:3` | ~8MB | Need shell, small footprint |

**scratch** is ideal for musl-compiled static binaries.
**distroless** is the default for dynamically linked
binaries — no shell, no package manager, minimal
attack surface.

---

## Static Linking with musl

Build a fully static binary with no runtime deps:

```dockerfile
FROM rust:1-alpine AS builder
WORKDIR /app

RUN apk add --no-cache musl-dev

COPY Cargo.toml Cargo.lock ./
RUN mkdir src && echo "fn main() {}" > src/main.rs
RUN cargo build --release
RUN rm -rf src target/release/deps/myapp* \
    target/release/myapp*

COPY src ./src
RUN cargo build --release

FROM scratch
COPY --from=builder /app/target/release/myapp /
ENTRYPOINT ["/myapp"]
```

For cross-compiling musl from a Debian-based image:

```bash
rustup target add x86_64-unknown-linux-musl
cargo build --release --target x86_64-unknown-linux-musl
```

You may need `musl-tools` installed:

```bash
apt-get install -y musl-tools
```

---

## Cross-Compilation with cross

`cross` uses Docker containers with pre-configured
toolchains for each target:

```bash
cargo install cross
cross build --release --target aarch64-unknown-linux-gnu
cross build --release --target x86_64-unknown-linux-musl
```

In CI, use cross for targets that differ from the
runner's architecture:

```yaml
- name: Build for ARM64
  run: |
    cargo install cross
    cross build --release --target aarch64-unknown-linux-gnu
```

---

## .dockerignore

```
target/
.git/
.github/
.gitignore
*.md
LICENSE
.env*
docker-compose*.yml
Dockerfile
.dockerignore
```

Always exclude `target/` — it can be gigabytes and
is rebuilt inside the container anyway.

---

## Layer Caching Optimization

Docker caches each layer. Order instructions from
least to most frequently changing:

```dockerfile
FROM rust:1-slim AS builder
WORKDIR /app

# 1. System deps (rarely change)
RUN apt-get update && apt-get install -y \
    pkg-config libssl-dev \
    && rm -rf /var/lib/apt/lists/*

# 2. Cargo deps (change when Cargo.lock changes)
COPY Cargo.toml Cargo.lock ./
RUN mkdir src && echo "fn main() {}" > src/main.rs
RUN cargo build --release
RUN rm -rf src target/release/deps/myapp* \
    target/release/myapp*

# 3. App source (changes every commit)
COPY src ./src
RUN cargo build --release
```

If you have a workspace with multiple crates, copy
all `Cargo.toml` files in the dependency-cache step:

```dockerfile
COPY Cargo.toml Cargo.lock ./
COPY crates/api/Cargo.toml crates/api/
COPY crates/core/Cargo.toml crates/core/
RUN mkdir -p crates/api/src crates/core/src
RUN echo "fn main() {}" > crates/api/src/main.rs
RUN echo "" > crates/core/src/lib.rs
RUN cargo build --release
```

---

## Health Check and Signal Handling

### Health check in Dockerfile

```dockerfile
FROM gcr.io/distroless/cc-debian12
COPY --from=builder /app/target/release/myapp /
EXPOSE 8080
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
    CMD ["/myapp", "health"]
ENTRYPOINT ["/myapp"]
```

For distroless (no curl), the binary must handle
the health check itself via a subcommand or a
dedicated health endpoint.

### Graceful shutdown

Handle SIGTERM for clean container stops:

```rust
use tokio::signal;

async fn shutdown_signal() {
    signal::ctrl_c()
        .await
        .expect("install CTRL+C handler");
}

// In Axum:
let listener = tokio::net::TcpListener::bind("0.0.0.0:8080")
    .await
    .unwrap();
axum::serve(listener, app)
    .with_graceful_shutdown(shutdown_signal())
    .await
    .unwrap();
```

---

## Docker Compose for Development

```yaml
services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "8080:8080"
    environment:
      - DATABASE_URL=postgres://app:secret@db:5432/myapp
      - RUST_LOG=debug
    depends_on:
      db:
        condition: service_healthy

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: myapp
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app"]
      interval: 5s
      timeout: 3s
      retries: 5

volumes:
  pgdata:
```

For faster iteration, mount source and use
`cargo-watch` instead of rebuilding the image:

```yaml
  app-dev:
    image: rust:1-slim
    working_dir: /app
    volumes:
      - .:/app
      - cargo-cache:/usr/local/cargo/registry
    command: cargo watch -x run
    ports:
      - "8080:8080"
    environment:
      - DATABASE_URL=postgres://app:secret@db:5432/myapp
      - RUST_LOG=debug

volumes:
  cargo-cache:
```
