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
