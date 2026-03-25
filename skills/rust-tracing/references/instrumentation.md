# Instrumentation

## The `#[instrument]` Macro

Automatically creates a span for a function, capturing
arguments as structured fields:

```rust
use tracing::instrument;

#[instrument(
    name = "process_order",
    level = "info",
    skip(db, payment_client),
    fields(
        order.id = %order_id,
        order.status = tracing::field::Empty,
    ),
    err,
    ret,
)]
async fn process_order(
    order_id: &str,
    amount: f64,
    db: &DatabasePool,
    payment_client: &PaymentClient,
) -> Result<OrderResult, OrderError> {
    info!("Starting order processing");
    let result = payment_client.charge(amount).await?;
    tracing::Span::current().record("order.status", "charged");
    Ok(result)
}
```

### Attribute options

| Option       | Purpose                                       | Example                                          |
| ------------ | --------------------------------------------- | ------------------------------------------------ |
| `name`       | Custom span name (defaults to fn name)        | `name = "handle_payment"`                        |
| `level`      | Span level (default: info)                    | `level = "debug"`                                |
| `skip`       | Omit specific args from span fields           | `skip(db_pool, body)`                            |
| `skip_all`   | Omit all args, define fields manually         | `skip_all, fields(user.id = %id)`                |
| `fields`     | Add explicit fields                           | `fields(order.id = %id)`                         |
| `err`        | Log errors via Display when Result is Err     | `err` or `err(Display)`                          |
| `ret`        | Log return value via Debug                    | `ret` or `ret(Display)`                          |

### `Empty` fields — record later

Declare a field as `tracing::field::Empty` and fill it
when the value becomes available:

```rust
#[instrument(fields(result_status = tracing::field::Empty))]
async fn process(data: &Data) -> Result<Output, Error> {
    let output = do_work(data).await?;
    Span::current().record("result_status", "success");
    Ok(output)
}
```

### When to instrument

**DO instrument:**
- Public API entry points (HTTP handlers, gRPC methods)
- Key business logic functions
- External service calls (DB, HTTP clients, queues)
- Functions where timing matters

**DON'T instrument:**
- Small utility / helper functions
- Pure computation with no I/O
- Functions called in tight loops
- Trivial getters/setters

---

## Tower / Axum Middleware

Use `tower_http::trace::TraceLayer` for automatic
per-request spans instead of instrumenting every handler:

```rust
use axum::{Router, routing::get};
use tower_http::trace::TraceLayer;

let app = Router::new()
    .route("/orders", get(list_orders).post(create_order))
    .route("/orders/:id", get(get_order))
    .layer(
        TraceLayer::new_for_http()
            .make_span_with(|request: &http::Request<_>| {
                let request_id = uuid::Uuid::new_v4();
                tracing::info_span!(
                    "http_request",
                    method = %request.method(),
                    uri = %request.uri(),
                    request_id = %request_id,
                )
            })
            .on_response(
                |response: &http::Response<_>,
                 latency: std::time::Duration,
                 _span: &tracing::Span| {
                    tracing::info!(
                        status = response.status().as_u16(),
                        latency_ms = latency.as_millis() as u64,
                        "Response sent"
                    );
                },
            ),
    );
```

This creates a span per request with method, URI, status,
and latency — no `#[instrument]` needed on handlers.

---

## Database (sqlx)

`sqlx` has built-in tracing. Enable in Cargo.toml:

```toml
sqlx = { version = "0.8", features = ["runtime-tokio", "postgres", "tracing"] }
```

Control query verbosity via `RUST_LOG`:

```bash
RUST_LOG="info,sqlx=warn"       # Quiet in production
RUST_LOG="debug,sqlx=debug"     # See all queries in dev
```

---

## HTTP Client (reqwest)

Use `reqwest-tracing` for automatic span creation on
outbound calls:

```toml
reqwest-tracing = "0.5"
reqwest-middleware = "0.3"
```

```rust
use reqwest_middleware::ClientBuilder;
use reqwest_tracing::TracingMiddleware;

let client = ClientBuilder::new(reqwest::Client::new())
    .with(TracingMiddleware::default())
    .build();
```

---

## Correlation & Request IDs

Every incoming request should carry a unique ID that
propagates through all downstream operations.

### Generate at the boundary

```rust
let request_id = Uuid::new_v4();
let span = info_span!(
    "request",
    request.id = %request_id,
    method = %req.method(),
    path = %req.uri().path(),
);
```

### Propagation is automatic

All child spans and events inherit the request ID —
no manual threading needed:

```rust
#[instrument(skip_all, fields(request.id))]
async fn handle_request(req: Request) -> Response {
    let user = get_user(req.user_id).await?;  // carries request.id
    let order = create_order(&user).await?;    // carries request.id
    Ok(Response::new(order))
}
```

### Return in error responses

Include the request ID so clients can reference it in
support tickets:

```json
{
    "error": "Internal server error",
    "request_id": "550e8400-e29b-41d4-a716-446655440000"
}
```

---

## What to Log by Application Layer

### Startup & shutdown

```rust
info!(
    version = env!("CARGO_PKG_VERSION"),
    environment = %config.environment,
    port = config.port,
    "Application starting"
);
info!("Graceful shutdown initiated");
info!(pending_requests = count, "Draining connections");
```

### Authentication & authorization

```rust
info!(user.id = %user_id, method = "oauth2", "User authenticated");
warn!(user.id = %user_id, resource = %path, "Authorization denied");
error!(ip = %remote_addr, attempts = count, "Brute force detected");
```

### Database operations

```rust
debug!(
    query = "find_user_by_email",
    latency_ms = elapsed.as_millis() as u64,
    rows_returned = count,
    "Query executed"
);
warn!(
    query = "update_inventory",
    latency_ms = elapsed.as_millis() as u64,
    "Slow query detected"
);
```

### External service calls

```rust
info!(service = "payment_gateway", action = "charge", "Calling service");
error!(service = "payment_gateway", error = %e, "External call failed");
```

### Background jobs

```rust
info!(job.id = %job_id, job.type = "email_send", "Job started");
info!(job.id = %job_id, duration_ms = elapsed.as_millis() as u64, "Job completed");
error!(job.id = %job_id, error = %e, attempt = retry_count, "Job failed");
```

### Business events

```rust
info!(order.id = %id, order.total = total, items = count, "Order placed");
info!(user.id = %id, plan = %new_plan, "Subscription upgraded");
warn!(inventory.sku = %sku, remaining = qty, "Low stock alert");
```

---

## Timing Pattern

```rust
use std::time::Instant;

async fn timed_operation() {
    let start = Instant::now();
    do_work().await;
    let elapsed = start.elapsed();

    info!(duration_ms = elapsed.as_millis() as u64, "Operation completed");

    if elapsed > Duration::from_secs(5) {
        warn!(duration_ms = elapsed.as_millis() as u64, "Slow operation");
    }
}
```
