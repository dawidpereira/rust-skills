# Tracing Setup

## tracing vs log

`tracing` is the modern standard (Tokio project, 387M+
downloads). It provides structured events, spans, and
native async support. The `log` crate is a lightweight
facade without spans or structured context.

**Libraries** should use `tracing` with the `log` feature
so consumers using either crate see events:

```toml
[dependencies]
tracing = { version = "0.1", features = ["log"] }
```

**Applications** should use `tracing` directly and bridge
`log`-based dependencies via `tracing-log` or the `log`
feature.

---

## Cargo.toml Dependencies

```toml
[dependencies]
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = [
    "env-filter",
    "json",
    "fmt",
] }

# Optional: file output with rotation
tracing-appender = "0.2"

# Optional: OpenTelemetry
tracing-opentelemetry = "0.25"
opentelemetry = "0.24"
opentelemetry_sdk = { version = "0.24", features = ["rt-tokio"] }
opentelemetry-otlp = "0.17"
```

---

## Basic Initialization

The simplest setup — respects `RUST_LOG`, human-readable
output to stdout:

```rust
fn main() {
    tracing_subscriber::fmt::init();
    tracing::info!("Application started");
}
```

This defaults to `INFO` level if `RUST_LOG` is not set.

---

## Production Initialization (Layered)

Build a subscriber from composable layers on a `Registry`:

```rust
use tracing_subscriber::{
    fmt, layer::SubscriberExt, util::SubscriberInitExt, EnvFilter,
};

fn init_tracing() {
    let env_filter = EnvFilter::try_from_default_env()
        .unwrap_or_else(|_| EnvFilter::new("info,my_app=debug"));

    let fmt_layer = fmt::layer()
        .json()
        .with_target(true)
        .with_thread_ids(true)
        .with_file(true)
        .with_line_number(true)
        .flatten_event(true);

    tracing_subscriber::registry()
        .with(env_filter)
        .with(fmt_layer)
        .init();
}
```

---

## Dev vs Production Presets

Switch output format based on environment:

```rust
fn init_tracing(is_production: bool) {
    let env_filter = EnvFilter::try_from_default_env()
        .unwrap_or_else(|_| {
            if is_production {
                EnvFilter::new("info")
            } else {
                EnvFilter::new("debug,hyper=warn,tower_http=info")
            }
        });

    let registry = tracing_subscriber::registry().with(env_filter);

    if is_production {
        registry.with(fmt::layer().json()).init();
    } else {
        registry.with(fmt::layer().pretty()).init();
    }
}
```

---

## RUST_LOG Syntax

`EnvFilter` parses `RUST_LOG` to control verbosity at
runtime without recompiling.

### Basic patterns

```bash
# Global level
RUST_LOG=info

# Module-specific
RUST_LOG=warn,my_app=debug,my_app::db=trace

# With target filtering
RUST_LOG=my_app::api=debug,sqlx=warn,tower_http=debug

# Span-level filtering
RUST_LOG="my_app[process_order]=trace"
```

### Common presets

```bash
# Development: verbose app, quiet dependencies
RUST_LOG="debug,hyper=warn,h2=warn,tower=info,sqlx=warn"

# Production: info baseline, debug for specific modules
RUST_LOG="info,my_app::payments=debug"

# Troubleshooting: trace a specific module
RUST_LOG="warn,my_app::auth=trace"
```

### EnvFilter with fallback

Always provide a fallback when `RUST_LOG` is not set:

```rust
let filter = EnvFilter::try_from_default_env()
    .unwrap_or_else(|_| EnvFilter::new("info"));
```

---

## Compile-Time Level Filtering

Strip disabled levels entirely from the binary using
Cargo features — zero cost, not even a branch:

```toml
[dependencies]
tracing = { version = "0.1", features = ["max_level_info"] }
```

Available features:

| Feature            | Keeps                         |
| ------------------ | ----------------------------- |
| `max_level_error`  | error only                    |
| `max_level_warn`   | warn + error                  |
| `max_level_info`   | info + warn + error           |
| `max_level_debug`  | everything except trace       |
| `max_level_trace`  | everything (default)          |

Recommended: `max_level_info` for release builds,
`max_level_trace` (default) for debug builds.

For profile-specific filtering:

```toml
[profile.release.package.tracing]
features = ["max_level_info"]
```

---

## Core Concepts

### Events

Single point in time — something happened:

```rust
info!(user_id = %user.id, "User logged in");
warn!(retry_count = 3, "Connection unstable");
error!(error = %e, "Payment failed");
```

### Spans

Period of time — context that propagates to child
events and spans:

```rust
let span = info_span!("process_order", order_id = %order.id);
let _guard = span.enter();
info!("Validating order");  // carries order_id
info!("Charging payment");  // carries order_id
```

### Subscribers and Layers

- **Subscriber** consumes spans and events (writes to
  stdout, sends to OTel).
- **Layer** is a composable processing unit stacked on
  a `Registry`.
- **Registry** manages span storage and coordinates
  multiple layers.

Multiple layers on one registry lets you send formatted
output to stderr and structured JSON to a file
simultaneously.
