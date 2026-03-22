# API Patterns Reference

## Builder Pattern

Use builders when a type has many optional parameters or
complex initialization.

### When to Use

- Constructor would have 4+ parameters
- Several parameters are optional with sensible defaults
- Construction may require validation

### Bad: Constructor with Positional Arguments

```rust
let client = Client::new(
    "https://api.example.com",
    30,       // What is this?
    true,     // And this?
    None,
    Some("token"),
);
```

### Good: Builder with Named Methods

```rust
#[derive(Default)]
#[must_use = "builders do nothing unless you call build()"]
pub struct ClientBuilder {
    base_url: Option<String>,
    timeout: Option<Duration>,
    max_retries: u32,
    auth_token: Option<String>,
}

impl ClientBuilder {
    pub fn new() -> Self {
        Self::default()
    }

    pub fn base_url(mut self, url: impl Into<String>) -> Self {
        self.base_url = Some(url.into());
        self
    }

    pub fn timeout(mut self, timeout: Duration) -> Self {
        self.timeout = Some(timeout);
        self
    }

    pub fn max_retries(mut self, n: u32) -> Self {
        self.max_retries = n;
        self
    }

    pub fn build(self) -> Result<Client, BuilderError> {
        let base_url = self.base_url.ok_or(BuilderError::MissingBaseUrl)?;
        Ok(Client {
            base_url,
            timeout: self.timeout.unwrap_or(Duration::from_secs(30)),
            max_retries: self.max_retries,
            auth_token: self.auth_token,
        })
    }
}

let client = ClientBuilder::new()
    .base_url("https://api.example.com")
    .timeout(Duration::from_secs(10))
    .max_retries(5)
    .build()?;
```

### Typestate Builder

Enforce required fields at compile time. Only expose
`build()` when all required fields are set.

```rust
pub struct NoUrl;
pub struct HasUrl(String);

pub struct ClientBuilder<Url> {
    url: Url,
    timeout: Option<Duration>,
}

impl ClientBuilder<NoUrl> {
    pub fn new() -> Self {
        Self { url: NoUrl, timeout: None }
    }

    pub fn url(self, url: String) -> ClientBuilder<HasUrl> {
        ClientBuilder { url: HasUrl(url), timeout: self.timeout }
    }
}

impl ClientBuilder<HasUrl> {
    pub fn build(self) -> Client {
        Client { url: self.url.0, timeout: self.timeout }
    }
}

// Won't compile: ClientBuilder::new().build()
// Compiles:      ClientBuilder::new().url("...".into()).build()
```

---

## #[must_use]

Mark types and functions where ignoring the return value
is likely a bug.

### When to Apply

| Apply to | Reason |
|----------|--------|
| Builder types | Dropped builder does nothing |
| Builder methods returning `Self` | Dropped return loses configuration |
| Pure functions (no side effects) | Discarding result is pointless |
| Functions returning `Result` | Errors must be handled |
| Futures | Futures do nothing unless polled |
| RAII guards / locks | Immediately dropped guard has no effect |

### Bad: Silent Drop

```rust
let request = RequestBuilder::new("https://example.com");
request.timeout(Duration::from_secs(30));  // Dropped! No warning.
request.header("Auth", "Bearer token");    // Dropped! No warning.
let response = request.send();             // Sends with no timeout or headers
```

### Good: Compiler Warning on Drop

```rust
#[must_use = "builders do nothing unless consumed"]
pub struct RequestBuilder { /* ... */ }

impl RequestBuilder {
    #[must_use = "builder methods return modified builder - chain or assign"]
    pub fn timeout(mut self, duration: Duration) -> Self {
        self.timeout = Some(duration);
        self
    }
}
```

Mark the type itself with `#[must_use]` to cover all
methods returning `Self`. Add descriptive messages
explaining why the value matters.

### Clippy Lints

```toml
[lints.clippy]
must_use_candidate = "warn"
return_self_not_must_use = "warn"
```

---

## #[non_exhaustive]

Prevent downstream code from exhaustively matching enums
or constructing structs with literal syntax. Allows adding
variants/fields in minor versions.

### When to Use

- Public enums that may gain variants (especially error
  types)
- Public structs that may gain fields
- Any type in a library API that is likely to evolve

### When NOT to Use

- Internal types (not `pub`)
- Enums that are logically complete
  (`Ordering`: `Less, Equal, Greater`)
- Types in binary crates (no downstream users)

### Bad: Adding a Variant Breaks Users

```rust
pub enum ErrorKind {
    NotFound,
    PermissionDenied,
    TimedOut,
}

// Downstream: exhaustive match breaks when you add a variant
match err {
    ErrorKind::NotFound => { /* ... */ }
    ErrorKind::PermissionDenied => { /* ... */ }
    ErrorKind::TimedOut => { /* ... */ }
}
```

### Good: Forward-Compatible Enum

```rust
#[non_exhaustive]
pub enum ErrorKind {
    NotFound,
    PermissionDenied,
    TimedOut,
}

// Downstream must include wildcard
match err {
    ErrorKind::NotFound => { /* ... */ }
    ErrorKind::PermissionDenied => { /* ... */ }
    ErrorKind::TimedOut => { /* ... */ }
    _ => { /* handle future variants */ }
}
```

### Non-Exhaustive Structs

```rust
#[non_exhaustive]
pub struct Config {
    pub name: String,
    pub value: i32,
}

impl Config {
    pub fn new(name: impl Into<String>, value: i32) -> Self {
        Config { name: name.into(), value }
    }
}

// Downstream cannot use struct literal syntax, must use constructor
```

---

## Extension Traits

Add methods to external types by defining a trait and
implementing it.

### Convention

Name the trait `TypeExt` where `Type` is what you are
extending. The trait must be imported to use the methods
(scoped extension).

### Bad: Cannot Impl on External Type

```rust
impl Vec<u8> {
    fn as_hex(&self) -> String { /* error: cannot define inherent impl */ }
}
```

### Good: Extension Trait

```rust
pub trait ByteSliceExt {
    fn as_hex(&self) -> String;
    fn is_ascii_printable(&self) -> bool;
}

impl ByteSliceExt for [u8] {
    fn as_hex(&self) -> String {
        self.iter().map(|b| format!("{b:02x}")).collect()
    }

    fn is_ascii_printable(&self) -> bool {
        self.iter().all(|b| b.is_ascii_graphic() || b.is_ascii_whitespace())
    }
}

// Usage: must import the trait
use my_crate::ByteSliceExt;
let hex = b"hello".as_hex();
```

Ecosystem examples: `itertools::Itertools`,
`futures::StreamExt`, `tokio::io::AsyncReadExt`,
`anyhow::Context`.

---

## Default Implementation

Implement `Default` when a type has sensible zero/empty/
initial values.

### Derive vs Manual

Use `#[derive(Default)]` when field defaults (0, false,
empty string, empty vec) are correct. Implement manually
when custom values are needed.

```rust
impl Default for Connection {
    fn default() -> Self {
        Connection {
            host: "localhost".to_string(),
            port: 8080,
            timeout: Duration::from_secs(30),
        }
    }
}
```

Do NOT implement `Default` when some fields have no
sensible default (e.g., a user ID). Use a constructor
instead.

### Integration Points

- `Option::unwrap_or_default()`
- Struct update syntax:
  `Config { retries: 5, ..Default::default() }`
- Generic bounds: `T: Default`
- Builder initialization:
  `ServerBuilder::default().host("0.0.0.0").build()`

---

## Common Trait Derivation

### Minimum for All Public Types

```rust
#[derive(Debug, Clone, PartialEq)]
pub struct MyType { /* ... */ }
```

### Manual Implementations

Implement manually when:

- **PartialEq** needs custom logic (case-insensitive
  comparison)
- **Hash** must be consistent with custom PartialEq
- **Debug** should redact sensitive data

```rust
impl Debug for Password {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "Password([REDACTED])")
    }
}
```

---

## Serde Feature Gating

Make serde optional for library crates. Not all users need
serialization.

### Bad: Hard Dependency

```toml
[dependencies]
serde = { version = "1.0", features = ["derive"] }
```

### Good: Feature Flag

```toml
[dependencies]
serde = { version = "1.0", features = ["derive"], optional = true }

[features]
default = []
serde = ["dep:serde"]
```

```rust
#[derive(Debug, Clone)]
#[cfg_attr(feature = "serde", derive(serde::Serialize, serde::Deserialize))]
#[cfg_attr(docsrs, doc(cfg(feature = "serde")))]
pub struct Config {
    pub name: String,
    pub value: i32,
}
```

Users opt in:
`my_crate = { version = "1.0", features = ["serde"] }`

### When Serde Should Be Required

Only when your crate IS about serialization (config file
parser, API client, data format library). For everything
else, make it optional.

### Testing Features

```bash
cargo test                 # Without serde
cargo test --features serde  # With serde
cargo test --all-features    # All combinations
```
