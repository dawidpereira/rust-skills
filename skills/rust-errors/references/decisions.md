# Error Handling Decisions

## When to Panic vs Return Result

| Scenario | Use | Reasoning |
|----------|-----|-----------|
| File not found | `Result` | Expected — caller may want to create it or try another path |
| Network timeout | `Result` | Expected — caller retries or shows user message |
| Invalid user input | `Result` | Expected — caller validates and reports |
| Index out of bounds in internal logic | `panic` | Bug — indicates broken invariant |
| Corrupted internal data structure | `panic` | Bug — program state is inconsistent |
| Required env var missing at startup | `panic` (or `Result` from `main`) | Setup failure — can't proceed |
| Test assertion | `panic` / `unwrap` | Test — panics give clear failure messages |
| `Vec::push` fails (OOM) | `panic` (Rust default) | Infrastructure — most programs can't recover |
| Config file malformed | `Result` | Expected — user error, fixable |

### The Rule

**Panic for bugs. Return errors for anything the
user/caller/environment might cause.**

If you find yourself writing
`panic!("this should never happen")`, make sure it
*actually* can't happen — or convert to `Result`.

---

## unwrap vs expect vs ?

```rust
// unwrap: crashes with generic message — acceptable in tests
let val = result.unwrap();

// expect: crashes with YOUR message — acceptable for verified invariants
let val = map.get(&key).expect("key was inserted on line 42");

// ?: propagates to caller — use in production code
let val = result?;
```

### When unwrap/expect IS Acceptable

```rust
// Tests — panic = test failure, which is what we want
#[test]
fn test_parse() {
    let config = Config::parse("valid.toml").unwrap();
    assert_eq!(config.port, 8080);
}

// After validation — you've already checked
let value = input.parse::<u32>().expect("already validated as numeric");

// Compile-time provable — regex, static data
let re = Regex::new(r"^\d{4}-\d{2}-\d{2}$").unwrap();  // Regex is valid by inspection

// Infallible conversions
let num: u32 = u8::MAX.into();  // Always succeeds
```

### When unwrap/expect is NOT Acceptable

```rust
// Bad: user input can fail
let port: u16 = args[1].parse().unwrap();

// Bad: file might not exist
let content = std::fs::read_to_string("config.toml").unwrap();

// Bad: network can fail
let response = client.get(url).send().unwrap();

// All of these should use ? or match/if-let
```

---

## Custom Error Type Design

### When You Need Custom Types

- Library with multiple failure modes callers want to distinguish
- Domain errors that don't map to io::Error or similar
- You want structured error data (not just a message)

### Design Guidelines

1. **One variant per distinct failure mode the caller
   cares about** — not one per internal function
2. **Use `#[non_exhaustive]`** for public enums to allow adding variants later
3. **Implement `Debug`, `Display`, `Error`** (thiserror handles all three)
4. **Keep variants meaningful** — "Other(String)" is a code smell

```rust
use thiserror::Error;

#[derive(Error, Debug)]
#[non_exhaustive]
pub enum AuthError {
    #[error("invalid credentials")]
    InvalidCredentials,

    #[error("token expired at {expired_at}")]
    TokenExpired { expired_at: chrono::DateTime<chrono::Utc> },

    #[error("insufficient permissions: requires {required}")]
    Forbidden { required: String },

    #[error("authentication service unavailable")]
    ServiceUnavailable(#[source] reqwest::Error),
}
```

### Error Type Hierarchy

For larger projects, use a layered approach:

```rust
// Low-level: specific to subsystem
#[derive(Error, Debug)]
pub enum DbError {
    #[error("connection failed")]
    Connection(#[source] sqlx::Error),
    #[error("query timeout after {0:?}")]
    Timeout(Duration),
}

// High-level: what the API consumer sees
#[derive(Error, Debug)]
pub enum ApiError {
    #[error("user not found: {0}")]
    NotFound(UserId),
    #[error("database error")]
    Database(#[from] DbError),
    #[error("invalid request")]
    BadRequest(String),
}
```

---

## Choosing Between Error Handling Approaches

| Approach | Best For | Trade-off |
|----------|----------|-----------|
| `thiserror` enum | Libraries, typed errors | More upfront work, better caller experience |
| `anyhow::Error` | Applications, scripts | Fast to write, callers can't match variants |
| `Box<dyn Error>` | Quick prototypes | Simple, but no type info and allocates |
| Custom `From` impls | Complex conversions | Full control, most verbose |
| `String` errors | Never in real code | No structure, no chaining, no matching |

### Migration Path

Start with `anyhow` in applications. When you find
yourself needing to match on errors, extract a `thiserror`
enum for that boundary:

```rust
// Start: everything is anyhow
fn process() -> anyhow::Result<()> { ... }

// Later: extract typed errors where callers need to match
fn process() -> Result<(), ProcessError> { ... }  // thiserror enum
```

---

## Patterns to Avoid

### Silent Error Swallowing

```rust
// Bad: error disappears
let _ = important_operation();
if let Ok(val) = might_fail() { use(val); }

// Good: at minimum, log or document
if let Err(e) = best_effort_operation() {
    tracing::warn!("non-critical operation failed: {e}");
}
```

### Panic as Control Flow

```rust
// Bad: using panic for expected conditions
fn find_user(id: u64) -> User {
    db.get(id).unwrap_or_else(|| panic!("user {id} not found"))
}

// Good: return the error
fn find_user(id: u64) -> Result<User, UserError> {
    db.get(id).ok_or(UserError::NotFound(id))
}
```

### Overly Generic Errors

```rust
// Bad: caller can't distinguish failures
fn process() -> Result<(), String> { ... }
fn process() -> Result<(), Box<dyn Error>> { ... }

// Good: typed errors for library boundaries
fn process() -> Result<(), ProcessError> { ... }
```

### Too Many Error Variants

If your error enum has 15+ variants, you're probably
modeling internal implementation detail. Group related
failures:

```rust
// Bad: one variant per SQL query
enum DbError { SelectUserFailed, InsertUserFailed, UpdateUserFailed, ... }

// Good: one variant per failure mode
enum DbError { Connection(sqlx::Error), Query(sqlx::Error), Migration(sqlx::Error) }
```

---

## Diagnostic Error Reporting

### miette — Rich Diagnostic Errors

Use `miette` when errors need to point at source code,
configuration files, or user-authored input. Ideal for
CLI tools, compilers, linters, and config parsers.

#### Setup

```toml
[dependencies]
miette = { version = "7", features = ["fancy"] }
thiserror = "2"
```

`miette` works alongside `thiserror`. The `#[diagnostic]`
derive adds rich metadata on top of standard `Error`.

```rust
use miette::{Diagnostic, SourceSpan, NamedSource};
use thiserror::Error;

#[derive(Error, Diagnostic, Debug)]
#[error("invalid value in configuration")]
#[diagnostic(
    code(config::invalid_value),
    help("expected a positive integer, got a string")
)]
pub struct ConfigError {
    #[source_code]
    pub src: NamedSource<String>,

    #[label("this value is not valid")]
    pub span: SourceSpan,
}
```

#### Key Features

| Feature | Attribute / Type | Purpose |
|---------|-----------------|---------|
| Source spans | `#[label("...")]` + `SourceSpan` | Points at the offending text |
| Help text | `#[diagnostic(help("..."))]` | Suggests how to fix the error |
| Error codes | `#[diagnostic(code(...))]` | Machine-readable error identifier |
| Severity | `#[diagnostic(severity(...))]` | Warning vs error vs advice |
| Related | `#[related]` | Attach additional diagnostics |

#### Config Parser Example

```rust
use miette::{miette, NamedSource, SourceSpan, Result};

fn parse_port(
    src: &str,
    filename: &str,
    offset: usize,
    len: usize,
) -> Result<u16> {
    let value = &src[offset..offset + len];
    value.parse::<u16>().map_err(|_| {
        miette!(
            labels = vec![
                miette::LabeledSpan::at(
                    offset..offset + len,
                    "expected a port number (0-65535)",
                )
            ],
            help = "port must be a valid u16 integer",
            "invalid port in {filename}"
        )
        .with_source_code(
            NamedSource::new(filename, src.to_owned())
        )
    })
}
```

#### Using miette::Result in main

```rust
fn main() -> miette::Result<()> {
    let config = load_config("app.toml")?;
    run(config)?;
    Ok(())
}
```

This renders fancy diagnostics to stderr with source
snippets, labels, and help text when the `fancy` feature
is enabled.

---

### color-eyre — Colorized Backtraces

Use `color-eyre` as a drop-in replacement for `eyre`
when developer experience during debugging matters. It
adds colorized backtraces and integrates with `tracing`
for span traces.

#### Setup

```toml
[dependencies]
color-eyre = "0.6"
```

Install the panic and error hooks at startup:

```rust
fn main() -> color_eyre::Result<()> {
    color_eyre::install()?;

    let config = load_config("app.toml")?;
    run(config)?;
    Ok(())
}
```

#### Span Traces with tracing

When `tracing` is active, `color-eyre` captures the
current span stack and includes it in error reports:

```rust
use tracing::instrument;

#[instrument(skip(db))]
fn load_user(db: &Db, user_id: u64)
    -> color_eyre::Result<User>
{
    let row = db.query_one(
        "SELECT * FROM users WHERE id = $1",
        &[&user_id],
    )?;
    Ok(User::from_row(row)?)
}
```

Error output includes both the backtrace and the span
trace showing `load_user{user_id=42}`, making it easier
to reconstruct context.

#### When to Choose color-eyre vs anyhow

| Criterion | anyhow | color-eyre |
|-----------|--------|------------|
| Minimal dependencies | Yes | No (pulls in owo-colors, backtrace) |
| Colorized output | No | Yes |
| Span traces | No | Yes (with tracing) |
| Ecosystem adoption | Wider | Smaller but growing |

Use `anyhow` for libraries and lightweight CLIs. Use
`color-eyre` for applications where developers are the
primary audience for error output.

---

### User-Facing vs Developer-Facing Errors

In web applications and APIs, errors serve two audiences:
developers debugging the system and users consuming the
API. Never leak internal details to users.

#### Pattern: Dual-Layer Error

```rust
use axum::http::StatusCode;
use axum::response::{IntoResponse, Response};
use thiserror::Error;

#[derive(Error, Debug)]
pub enum AppError {
    #[error("database query failed")]
    Database(#[source] sqlx::Error),

    #[error("user {0} not found")]
    NotFound(u64),

    #[error("invalid request: {0}")]
    BadRequest(String),

    #[error("internal error: {0}")]
    Internal(#[from] anyhow::Error),
}

impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        let (status, message) = match &self {
            AppError::NotFound(_) => (
                StatusCode::NOT_FOUND,
                "resource not found".to_owned(),
            ),
            AppError::BadRequest(msg) => (
                StatusCode::BAD_REQUEST,
                msg.clone(),
            ),
            AppError::Database(_)
            | AppError::Internal(_) => {
                tracing::error!(error = %self, "internal error");
                (
                    StatusCode::INTERNAL_SERVER_ERROR,
                    "internal server error".to_owned(),
                )
            }
        };

        (status, message).into_response()
    }
}
```

The `IntoResponse` impl controls what the user sees.
Internal variants log the full error chain (including
source) via `tracing` but return a generic message.

#### Guidelines

- **User-facing message**: generic, safe, actionable
  ("invalid email format", "resource not found")
- **Internal log**: full error chain with context
  (`tracing::error!(error = %self, ...)`)
- **Never expose**: stack traces, SQL queries, file
  paths, internal IDs the user didn't provide

---

### CLI Error Reporting

#### Exit Codes

| Code | Meaning | Convention |
|------|---------|------------|
| 0 | Success | Program completed normally |
| 1 | General error | Catch-all for failures |
| 2 | Usage error | Bad arguments, missing flags |

Scripts and CI pipelines rely on these conventions.
Use `std::process::ExitCode` (Rust 1.61+) for explicit
control:

```rust
use std::process::ExitCode;

fn main() -> ExitCode {
    match run() {
        Ok(()) => ExitCode::SUCCESS,
        Err(e) => {
            eprintln!("error: {e:#}");
            ExitCode::from(1)
        }
    }
}
```

#### stderr vs stdout

- **stdout**: program output (data, results, piped content)
- **stderr**: errors, warnings, progress, diagnostics

This lets users pipe output while still seeing errors:
`my-tool process data.csv > output.json`

#### process::exit vs returning from main

| Approach | Behavior |
|----------|----------|
| `return` from `main` | Runs destructors, flushes buffers |
| `process::exit(1)` | Skips destructors — use only for fatal, unrecoverable situations |

Prefer returning `ExitCode` from `main`. Use
`process::exit` only when you need to bail from deeply
nested code and cleanup is not needed.

#### When to Show Backtraces

- **Default**: show only the error message chain
- **`--verbose` or `RUST_BACKTRACE=1`**: show full
  backtrace for developer debugging
- **Never** show backtraces to end users by default —
  they obscure the actual problem
