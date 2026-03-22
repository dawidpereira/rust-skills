# Error Handling Patterns

## thiserror: Library Error Types

`thiserror` generates `Error` trait implementations from
enum definitions. Use it for libraries where callers need
to match on specific error conditions.

### Basic Setup

```rust
use thiserror::Error;

#[derive(Error, Debug)]
pub enum ParseError {
    #[error("invalid syntax at line {line}: {message}")]
    Syntax { line: usize, message: String },

    #[error("unexpected end of file")]
    UnexpectedEof,

    #[error("invalid utf-8 encoding")]
    Utf8(#[from] std::str::Utf8Error),

    #[error("io error reading input")]
    Io(#[from] std::io::Error),
}
```

### Key Attributes

| Attribute | Effect |
|-----------|--------|
| `#[error("...")]` | Generates `Display` impl with interpolation |
| `#[from]` | Generates `From<E>` impl — enables `?` conversion |
| `#[source]` | Sets `Error::source()` without generating `From` |
| `#[error(transparent)]` | Delegates `Display` and `source()` to inner error |

### #[from] vs #[source]

- `#[from]`: Automatic conversion + source chain. Use when
  there's a 1:1 mapping.
- `#[source]`: Source chain only, no automatic conversion.
  Use when you need additional context fields.

```rust
#[derive(Error, Debug)]
pub enum ConfigError {
    // #[from]: io::Error automatically converts to ConfigError::Read
    #[error("failed to read config")]
    Read(#[from] std::io::Error),

    // #[source]: manual conversion needed, but preserves cause chain
    #[error("invalid value for '{key}'")]
    InvalidValue {
        key: String,
        #[source]
        cause: ValueError,
    },
}
```

---

## anyhow: Application Error Handling

`anyhow` provides `anyhow::Result<T>` (alias for
`Result<T, anyhow::Error>`) with context chaining. Use for
applications where you don't need typed error matching.

### Context

Add context to explain *what was happening* when the error occurred:

```rust
use anyhow::{Context, Result};

fn load_config(path: &Path) -> Result<Config> {
    let content = std::fs::read_to_string(path)
        .with_context(|| format!("failed to read config from {}", path.display()))?;

    let config: Config = toml::from_str(&content)
        .context("failed to parse config")?;

    Ok(config)
}
```

Use `.context()` for static messages, `.with_context()`
when the message needs formatting (lazy evaluation avoids
allocation on success path).

### bail! and ensure!

```rust
use anyhow::{bail, ensure, Result};

fn validate(config: &Config) -> Result<()> {
    ensure!(config.port > 0, "port must be positive, got {}", config.port);

    if config.workers == 0 {
        bail!("worker count cannot be zero");
    }

    Ok(())
}
```

### Main Function Pattern

```rust
fn main() -> anyhow::Result<()> {
    let config = load_config("app.toml")?;
    run(config)?;
    Ok(())
}

// Displays full error chain:
// Error: failed to initialize
//
// Caused by:
//    0: failed to read config from app.toml
//    1: No such file or directory (os error 2)
```

---

## The ? Operator In Depth

`?` does three things:

1. On `Ok(v)` → unwraps to `v`
2. On `Err(e)` → converts via `From::from(e)` and returns early
3. Works with both `Result` and `Option` (returns `None` for `Option`)

### ? with Option

```rust
fn first_even(numbers: &[i32]) -> Option<i32> {
    let first = numbers.first()?;  // Returns None if empty
    if first % 2 == 0 {
        Some(*first)
    } else {
        None
    }
}
```

### ? in main()

```rust
// With anyhow
fn main() -> anyhow::Result<()> { ... }

// With custom error
fn main() -> Result<(), Box<dyn std::error::Error>> { ... }

// With ExitCode (Rust 1.61+)
fn main() -> ExitCode {
    match run() {
        Ok(()) => ExitCode::SUCCESS,
        Err(e) => {
            eprintln!("Error: {e:#}");
            ExitCode::FAILURE
        }
    }
}
```

---

## Error Chaining

Preserve the causal chain so debugging reveals the full path:

```rust
use thiserror::Error;

#[derive(Error, Debug)]
pub enum AppError {
    #[error("failed to load user {user_id}")]
    LoadUser {
        user_id: u64,
        #[source]
        cause: DbError,
    },
}

// Walking the chain
fn print_error_chain(err: &dyn std::error::Error) {
    eprintln!("Error: {err}");
    let mut source = err.source();
    while let Some(cause) = source {
        eprintln!("  Caused by: {cause}");
        source = cause.source();
    }
}
```

---

## Error Message Conventions

Following Rust standard library convention:

```rust
// Good: lowercase, no trailing punctuation
#[error("connection refused")]
ConnectionRefused,

#[error("invalid header value: {0}")]
InvalidHeader(String),

// Bad: capitalized, has punctuation
#[error("Connection refused.")]
#[error("Invalid header value!")]
```

Exceptions: acronyms and error codes stay uppercase:

```rust
#[error("DNS lookup failed for {host}")]
#[error("HTTP {status} from {url}")]
```

Context reads naturally when chained:

```text
Error: failed to start server
  Caused by: failed to bind address
    Caused by: address already in use (os error 98)
```

---

## Documenting Errors

Every fallible public function should document its error conditions:

```rust
/// Loads configuration from the given path.
///
/// # Errors
///
/// Returns [`ConfigError::NotFound`] if the file doesn't exist.
/// Returns [`ConfigError::Parse`] if the file contains invalid TOML.
pub fn load_config(path: &Path) -> Result<Config, ConfigError> { ... }
```

---

## Don't Silently Ignore Errors

```rust
// Bad: error swallowed silently
let _ = file.sync_all();
if let Err(_) = send_notification() { }

// Good: explicitly document why you're ignoring
let _ = file.sync_all();  // Best-effort flush; data already committed to DB

// Better: log it
if let Err(e) = send_notification() {
    tracing::warn!("notification failed: {e}");
}
```

**Clippy lint:** `clippy::let_underscore_must_use` catches
ignored `#[must_use]` values.
