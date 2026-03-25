# Production

## OpenTelemetry Integration

### Setup with `tracing-opentelemetry`

```rust
use opentelemetry::KeyValue;
use opentelemetry_otlp::WithExportConfig;
use opentelemetry_sdk::{
    trace::{RandomIdGenerator, Sampler},
    Resource,
};
use tracing_opentelemetry::OpenTelemetryLayer;
use tracing_subscriber::{
    layer::SubscriberExt, util::SubscriberInitExt, EnvFilter,
};

fn init_telemetry() -> Result<(), Box<dyn std::error::Error>> {
    let resource = Resource::new(vec![
        KeyValue::new("service.name", env!("CARGO_PKG_NAME")),
        KeyValue::new("service.version", env!("CARGO_PKG_VERSION")),
        KeyValue::new("deployment.environment", "production"),
    ]);

    let tracer_provider = opentelemetry_otlp::new_pipeline()
        .tracing()
        .with_exporter(
            opentelemetry_otlp::new_exporter()
                .tonic()
                .with_endpoint("http://localhost:4317"),
        )
        .with_trace_config(
            opentelemetry_sdk::trace::Config::default()
                .with_sampler(Sampler::AlwaysOn)
                .with_id_generator(RandomIdGenerator::default())
                .with_resource(resource),
        )
        .install_batch(opentelemetry_sdk::runtime::Tokio)?;

    let otel_layer = OpenTelemetryLayer::new(
        tracer_provider.tracer("my-service"),
    );

    tracing_subscriber::registry()
        .with(EnvFilter::new("info"))
        .with(tracing_subscriber::fmt::layer().json())
        .with(otel_layer)
        .init();

    Ok(())
}

fn shutdown_telemetry() {
    opentelemetry::global::shutdown_tracer_provider();
}
```

### OTel-specific span fields

`tracing-opentelemetry` reserves the `otel.` prefix:

```rust
let span = info_span!(
    "http_request",
    otel.name = "GET /api/orders",
    otel.kind = "server",
    otel.status_code = "ok",
);
```

### Setting span status from results

```rust
#[instrument(err, ret, fields(otel.status_code))]
async fn fallible_operation() -> Result<String, AppError> {
    let result = do_work().await;
    match &result {
        Ok(_) => Span::current().record("otel.status_code", "ok"),
        Err(e) => {
            Span::current().record("otel.status_code", "error");
            error!(error = %e, "Operation failed");
        }
    }
    result
}
```

### Distributed context propagation

For service-to-service calls, inject/extract trace
context via OpenTelemetry propagators:

```rust
use opentelemetry::global;

let mut headers = reqwest::header::HeaderMap::new();
global::get_text_map_propagator(|propagator| {
    propagator.inject_context(
        &tracing::Span::current().context(),
        &mut HeaderInjector(&mut headers),
    );
});
```

---

## File Output

### Non-blocking file writer

Use `tracing-appender` to avoid blocking the async
runtime:

```rust
use tracing_appender::rolling::{RollingFileAppender, Rotation};

fn init_tracing() {
    let file_appender = RollingFileAppender::new(
        Rotation::DAILY,
        "/var/log/my-app",
        "my-app.log",
    );
    let (non_blocking, _guard) = tracing_appender::non_blocking(file_appender);

    tracing_subscriber::registry()
        .with(EnvFilter::new("info"))
        .with(fmt::layer().json().with_writer(non_blocking))
        .init();

    // IMPORTANT: _guard must stay alive for the application's lifetime.
    // When dropped, it flushes remaining logs.
}
```

### Multiple output destinations

Send to stderr (human-readable) and file (JSON)
simultaneously using layers:

```rust
let file_appender = tracing_appender::rolling::daily("/var/log/my-app", "app.log");
let (file_writer, _guard) = tracing_appender::non_blocking(file_appender);

tracing_subscriber::registry()
    .with(EnvFilter::new("info"))
    .with(fmt::layer().pretty().with_writer(std::io::stderr))
    .with(fmt::layer().json().with_writer(file_writer))
    .init();
```

---

## Performance

### Compile-time filtering

Strip disabled levels from the binary entirely:

```toml
tracing = { version = "0.1", features = ["max_level_info"] }
```

See `setup.md` for the full feature table.

### Avoid expensive log arguments

```rust
// Bad: runs even when debug is filtered out
debug!("State: {}", expensive_serialization(&state));

// Good: deferred evaluation via structured fields
debug!(state = ?state, "Current state");

// Good: guard truly expensive work
if tracing::enabled!(tracing::Level::DEBUG) {
    let summary = compute_expensive_summary(&data);
    debug!(summary = %summary, "Computed summary");
}
```

### Non-blocking writers

Always use `tracing_appender::non_blocking` for file or
network output. Blocking the async runtime to write logs
defeats the purpose of async.

---

## Testing

### tracing-test

Capture and assert on emitted events in tests:

```toml
[dev-dependencies]
tracing-test = "0.2"
```

```rust
#[cfg(test)]
mod tests {
    use tracing_test::traced_test;

    #[traced_test]
    #[test]
    fn test_logging_output() {
        process_order("ord-123");
        assert!(logs_contain("Order placed"));
        assert!(logs_contain("ord-123"));
    }

    #[traced_test]
    #[tokio::test]
    async fn test_async_logging() {
        handle_request().await;
        assert!(logs_contain("Request completed"));
    }
}
```

`#[traced_test]` sets up a per-test subscriber that
captures output. `logs_contain` checks the captured
output for a substring.

---

## Panic Hook

Log panics as structured events before crashing:

```rust
fn setup_panic_hook() {
    std::panic::set_hook(Box::new(|panic_info| {
        let location = panic_info
            .location()
            .map(|l| format!("{}:{}:{}", l.file(), l.line(), l.column()))
            .unwrap_or_else(|| "unknown".to_string());

        let message = panic_info
            .payload()
            .downcast_ref::<&str>()
            .map(|s| s.to_string())
            .or_else(|| {
                panic_info.payload().downcast_ref::<String>().cloned()
            })
            .unwrap_or_else(|| "Unknown panic".to_string());

        tracing::error!(
            panic.message = %message,
            panic.location = %location,
            "Application panicked"
        );
    }));
}
```

Call `setup_panic_hook()` early in `main()`, after
initializing the tracing subscriber.

---

## Ecosystem Quick Reference

| Crate                             | Purpose                                    |
| --------------------------------- | ------------------------------------------ |
| `tracing`                         | Core API (spans, events, macros)           |
| `tracing-subscriber`              | Subscribers, layers, filtering             |
| `tracing-appender`                | Non-blocking file writers, log rotation    |
| `tracing-opentelemetry`           | OpenTelemetry bridge                       |
| `tracing-log`                     | Bridge `log` events into `tracing`         |
| `tracing-test`                    | Test utilities, `#[traced_test]`           |
| `tracing-bunyan-formatter`        | Bunyan JSON format layer                   |
| `tracing-loki`                    | Ship logs to Grafana Loki                  |
| `tower-http`                      | HTTP middleware with `TraceLayer`           |
| `reqwest-tracing`                 | Auto-instrument outbound HTTP              |
| `opentelemetry`                   | OpenTelemetry API                          |
| `opentelemetry-otlp`              | OTLP exporter (Jaeger, Tempo, Datadog)     |
| `opentelemetry-appender-tracing`  | Bridge tracing events as OTel log records  |
| `secrecy`                         | Secret wrapper with redacted Debug/Display |
| `init-tracing-opentelemetry`      | Opinionated quick setup for tracing + OTel |

---

## Production Readiness Checklist

- [ ] `tracing_subscriber` initialized at the start of `main()`
- [ ] `EnvFilter` with sensible defaults (info for prod, debug for dev)
- [ ] JSON output for production
- [ ] Request IDs generated and propagated via spans
- [ ] Sensitive data protected with `skip`, `Redacted`, or `secrecy`
- [ ] Key business events logged at info level
- [ ] Errors logged with structured fields and context
- [ ] External service calls instrumented (DB, HTTP, queues)
- [ ] `#[instrument]` on important business logic functions
- [ ] HTTP middleware (`TraceLayer`) for automatic request tracing
- [ ] Async spans use `.instrument()`, not `span.enter()`
- [ ] Spawned tasks carry span context via `.instrument(span)`
- [ ] Compile-time level filtering for release builds
- [ ] Non-blocking writers for file/network output
- [ ] Panic hook installed for structured panic logging
- [ ] OpenTelemetry configured if using distributed tracing
- [ ] Graceful shutdown flushes pending spans
- [ ] `RUST_LOG` documented for operators
- [ ] Noisy crates filtered (`hyper=warn`, `h2=warn`, `sqlx=warn`)
