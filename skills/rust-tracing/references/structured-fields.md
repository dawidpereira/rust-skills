# Structured Fields

## Field Formatting Sigils

```rust
// %value — Display trait
info!(user = %user.name, "Greeting user");

// ?value — Debug trait
debug!(config = ?app_config, "Loaded configuration");

// No sigil — primitives implementing tracing::Value
info!(count = 42, active = true, "Current state");
```

| Sigil | Trait     | Use For                                    |
| ----- | --------- | ------------------------------------------ |
| `%`   | `Display` | User-facing strings, IDs, formatted values |
| `?`   | `Debug`   | Structs, enums, complex types              |
| none  | `Value`   | Integers, bools, `&str`                    |

---

## Naming Conventions

Use `snake_case` for field names and dot-separated
namespaces for related fields:

```rust
// Request context
info!(
    request.id = %request_id,
    request.method = %method,
    request.path = %path,
    response.status = status.as_u16(),
    response.latency_ms = elapsed.as_millis() as u64,
    "Request completed"
);

// Business context
info!(
    order.id = %order_id,
    order.total = total,
    customer.id = %customer_id,
    "Order processed"
);
```

Consistent naming across the codebase makes logs
queryable in aggregation tools.

---

## Structured Fields vs String Interpolation

```rust
// Bad: unstructured, hard to query
info!("User {} logged in from {}", user_id, ip_address);

// Good: structured, queryable, machine-parseable
info!(user_id = %user_id, ip = %ip_address, "User logged in");
```

Structured fields let log aggregators filter and group
by field values. String interpolation buries data inside
a message string that requires regex to extract.

---

## Error Logging Patterns

### Include the error chain

```rust
error!(
    error = %e,
    error.source = ?e.source(),
    "Operation failed"
);
```

### anyhow / eyre errors

The `:#` alternate format shows the full error chain:

```rust
error!(error = %err, "Request failed: {err:#}");
```

### Level by recoverability

```rust
match result {
    Ok(val) => info!(result = ?val, "Success"),
    Err(e) if e.is_transient() => {
        warn!(error = %e, "Transient failure, will retry");
    }
    Err(e) => error!(error = %e, "Permanent failure"),
}
```

---

## Sensitive Data Protection

### Never log secrets

Passwords, tokens, API keys, credit card numbers, and
PII must never appear in logs.

### Use `skip` on `#[instrument]`

```rust
#[instrument(skip(password, token))]
async fn authenticate(
    username: &str,
    password: &str,
    token: &str,
) -> Result<Session> {
    // ...
}
```

### Redacted wrapper type

A newtype that always displays `[REDACTED]`:

```rust
use std::fmt;

pub struct Redacted<T>(pub T);

impl<T> fmt::Display for Redacted<T> {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "[REDACTED]")
    }
}

impl<T> fmt::Debug for Redacted<T> {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "[REDACTED]")
    }
}

info!(password = %Redacted(&password), "Login attempt");
```

### Masked wrapper type

Show partial values (e.g., last 4 digits):

```rust
pub struct Masked(pub String);

impl fmt::Display for Masked {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        if self.0.len() > 4 {
            write!(f, "****{}", &self.0[self.0.len() - 4..])
        } else {
            write!(f, "****")
        }
    }
}

info!(card = %Masked(card_number.to_string()), "Processing payment");
// Output: card=****4242
```

### The `secrecy` crate

Battle-tested approach with automatic memory zeroization
on drop:

```toml
secrecy = { version = "0.10", features = ["serde"] }
```

```rust
use secrecy::{Secret, ExposeSecret};

struct Config {
    db_password: Secret<String>,
    api_key: Secret<String>,
}

// Debug output: Secret([REDACTED String])
// Access only when needed:
let conn = format!(
    "postgres://user:{}@host/db",
    config.db_password.expose_secret()
);
```

---

## Conditional Span Enrichment

Declare fields as `Empty` and fill them when available:

```rust
use tracing::Span;

fn enrich_span(span: &Span, user: Option<&User>) {
    if let Some(user) = user {
        span.record("user.id", user.id.to_string().as_str());
        span.record("user.role", user.role.as_str());
    }
}
```

This is useful when context is not available at span
creation but becomes known during processing.
