# Type-Driven Design

## Newtypes for Type Safety

Wrap primitives in newtypes to prevent mixing semantically
different values:

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub struct UserId(u64);

#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub struct OrderId(u64);

// Compiler catches argument swaps
fn get_order(user: UserId, order: OrderId) -> Order { ... }

// Won't compile: get_order(order_id, user_id)
```

### Validated Newtypes (Parse, Don't Validate)

Validate at construction time. Once the type exists, it's
valid — no re-checking needed.

```rust
pub struct Email(String);

impl Email {
    pub fn new(s: &str) -> Result<Self, ValidationError> {
        if s.contains('@') && s.len() >= 3 {
            Ok(Email(s.to_string()))
        } else {
            Err(ValidationError::InvalidEmail(s.to_string()))
        }
    }

    pub fn as_str(&self) -> &str { &self.0 }
}

// This function trusts the type — no validation needed
fn send_email(to: &Email, subject: &str, body: &str) { ... }
```

Common validated types: `Email`, `NonEmptyString`, `Port`,
`Percentage`, `Url`.

### Serde Integration

```rust
use serde::{Deserialize, Serialize};

#[derive(Debug, Clone, Serialize)]
#[serde(transparent)]
pub struct UserId(u64);

impl<'de> Deserialize<'de> for Email {
    fn deserialize<D: Deserializer<'de>>(d: D) -> Result<Self, D::Error> {
        let s = String::deserialize(d)?;
        Email::new(&s).map_err(serde::de::Error::custom)
    }
}
```

---

## Enums for Mutually Exclusive States

Use enums instead of boolean flags or optional fields to make
invalid states unrepresentable:

```rust
// Bad: what does active=true, suspended=true mean?
struct Account {
    active: bool,
    suspended: bool,
    ban_reason: Option<String>,
}

// Good: exactly one state at a time
enum AccountStatus {
    Active,
    Suspended { until: DateTime<Utc> },
    Banned { reason: String },
    Closed,
}

struct Account {
    status: AccountStatus,
    // ...
}
```

Pattern matching ensures you handle every state:

```rust
match account.status {
    AccountStatus::Active => process_order(),
    AccountStatus::Suspended { until } => {
        if Utc::now() > until { reactivate() }
        else { deny("account suspended") }
    }
    AccountStatus::Banned { reason } => deny(&reason),
    AccountStatus::Closed => deny("account closed"),
}
```

---

## Option and Result as Type-Level Contracts

### Option<T>: Absence is Explicit

```rust
// Bad: sentinel values
fn find_user(id: u64) -> User { ... }  // What if not found? Panic? Default?

// Good: absence is part of the type
fn find_user(id: u64) -> Option<User> { ... }

// Composable
let name = find_user(42)
    .map(|u| u.name.clone())
    .unwrap_or_else(|| "unknown".to_string());
```

### Result<T, E>: Failure is Explicit

```rust
// Bad: returns Option, losing error information
fn parse_config(s: &str) -> Option<Config> { ... }

// Good: Result preserves what went wrong
fn parse_config(s: &str) -> Result<Config, ConfigError> { ... }
```

---

## PhantomData: Type-Level Markers

`PhantomData<T>` expresses type relationships without
runtime data (zero-size):

```rust
use std::marker::PhantomData;

// Type-safe handles
struct Handle<T> {
    id: u64,
    _marker: PhantomData<T>,
}

struct User;
struct Order;

let user_handle: Handle<User> = Handle { id: 1, _marker: PhantomData };
let order_handle: Handle<Order> = Handle { id: 2, _marker: PhantomData };

// Can't mix them even though both contain u64
fn get_user(h: Handle<User>) -> User { ... }
// get_user(order_handle);  // Compile error!
```

### Common Uses

| Pattern | PhantomData Type | Purpose |
|---------|-----------------|---------|
| Type-safe handles | `PhantomData<T>` | Distinguish handle types |
| Lifetime marker | `PhantomData<&'a T>` | Express borrowing relationship |
| Variance control | `PhantomData<fn() -> T>` | Covariant marker |
| FFI type safety | `PhantomData<T>` | Associate opaque pointer with Rust type |

---

## Typestate Pattern

Encode state machine transitions in the type system. Invalid
transitions become compile errors.

```rust
// States are types
struct Disconnected;
struct Connected;
struct Authenticated;

struct Connection<State> {
    addr: String,
    _state: PhantomData<State>,
}

impl Connection<Disconnected> {
    fn new(addr: &str) -> Self {
        Connection { addr: addr.to_string(), _state: PhantomData }
    }

    fn connect(self) -> Result<Connection<Connected>, io::Error> {
        // ... establish connection
        Ok(Connection { addr: self.addr, _state: PhantomData })
    }
}

impl Connection<Connected> {
    fn authenticate(self, token: &str) -> Result<Connection<Authenticated>, AuthError> {
        // ... authenticate
        Ok(Connection { addr: self.addr, _state: PhantomData })
    }
}

impl Connection<Authenticated> {
    fn query(&self, sql: &str) -> Result<Rows, QueryError> {
        // Only available after authentication
        todo!()
    }
}

// Can't call query on a non-authenticated connection — won't compile
```

### Builder Typestate

Enforce required fields at compile time:

```rust
struct NoName;
struct HasName(String);

struct RequestBuilder<N> {
    url: String,
    name: N,
}

impl RequestBuilder<NoName> {
    fn new(url: &str) -> Self {
        RequestBuilder { url: url.to_string(), name: NoName }
    }

    fn name(self, name: &str) -> RequestBuilder<HasName> {
        RequestBuilder { url: self.url, name: HasName(name.to_string()) }
    }
}

impl RequestBuilder<HasName> {
    fn build(self) -> Request {
        Request { url: self.url, name: self.name.0 }
    }
}

// RequestBuilder::new("url").build()  // Compile error: no build() on NoName
// RequestBuilder::new("url").name("test").build()  // Works!
```

---

## The Never Type

`!` (never type) for functions that don't return normally:

```rust
fn exit_with_error(msg: &str) -> ! {
    eprintln!("Fatal: {msg}");
    std::process::exit(1);
}

// ! coerces to any type, useful in match arms
let value: u32 = match result {
    Ok(v) => v,
    Err(e) => exit_with_error(&e.to_string()),  // ! coerces to u32
};
```

On stable Rust, use `std::convert::Infallible` for types
that can't be constructed:

```rust
impl FromStr for AlwaysValid {
    type Err = Infallible;  // Parsing never fails
    fn from_str(s: &str) -> Result<Self, Infallible> {
        Ok(AlwaysValid(s.to_string()))
    }
}
```

---

## Avoid Stringly-Typed APIs

Strings accept any value — typos and invalid formats
compile fine.

```rust
// Bad: stringly-typed
fn set_log_level(level: &str) { ... }
set_log_level("wraning");  // Typo compiles fine!

// Good: enum
enum LogLevel { Error, Warn, Info, Debug, Trace }
fn set_log_level(level: LogLevel) { ... }
// set_log_level(LogLevel::Wraning)  // Compile error!
```

Parse strings to types at system boundaries:

```rust
impl FromStr for LogLevel {
    type Err = ParseError;
    fn from_str(s: &str) -> Result<Self, Self::Err> {
        match s.to_lowercase().as_str() {
            "error" => Ok(LogLevel::Error),
            "warn" | "warning" => Ok(LogLevel::Warn),
            _ => Err(ParseError::InvalidLevel(s.to_string())),
        }
    }
}
```
