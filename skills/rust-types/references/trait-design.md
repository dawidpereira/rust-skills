# Trait Design

## Sealed Traits

Sealed traits prevent external implementations while still
being usable. This lets you add methods in future versions
without breaking changes.

```rust
mod private {
    pub trait Sealed {}
}

pub trait MyTrait: private::Sealed {
    fn method(&self) -> String;
}

// Only your crate can implement Sealed, so only your crate can implement MyTrait
pub struct TypeA;
impl private::Sealed for TypeA {}
impl MyTrait for TypeA {
    fn method(&self) -> String { "A".to_string() }
}

pub struct TypeB;
impl private::Sealed for TypeB {}
impl MyTrait for TypeB {
    fn method(&self) -> String { "B".to_string() }
}
```

Document sealed traits clearly:

```rust
/// This trait is sealed and cannot be implemented outside of this crate.
pub trait DatabaseDriver: private::Sealed { ... }
```

### Partially Sealed

Allow some methods to be implemented externally, seal
others:

```rust
pub trait Plugin: private::Sealed {
    // Implementors must provide this
    fn name(&self) -> &str;

    // Only this crate provides the implementation
    fn register(&self) { ... }
}
```

---

## Extension Traits

Add methods to foreign types using the orphan rule
workaround:

```rust
pub trait StrExt {
    fn is_blank(&self) -> bool;
    fn truncate_to(&self, max: usize) -> &str;
}

impl StrExt for str {
    fn is_blank(&self) -> bool {
        self.trim().is_empty()
    }

    fn truncate_to(&self, max: usize) -> &str {
        if self.len() <= max { self }
        else {
            let mut end = max;
            while !self.is_char_boundary(end) { end -= 1; }
            &self[..end]
        }
    }
}

// Usage: only visible where StrExt is imported
use crate::StrExt;
"  hello  ".is_blank();  // false
"hello world".truncate_to(5);  // "hello"
```

Convention: use `Ext` suffix. Ecosystem examples:

- `itertools::Itertools` — extends `Iterator`
- `futures::StreamExt` — extends `Stream`
- `tokio::io::AsyncReadExt` — extends `AsyncRead`

---

## Object Safety

A trait is object-safe (usable as `dyn Trait`) when:

- No methods return `Self`
- No methods have generic type parameters
- No `Self: Sized` bound on the trait itself
- All methods take `self`, `&self`, or `&mut self`

```rust
// Object-safe
trait Draw {
    fn draw(&self);
    fn bounds(&self) -> Rect;
}

let shapes: Vec<Box<dyn Draw>> = vec![
    Box::new(Circle::new(10.0)),
    Box::new(Rectangle::new(5.0, 3.0)),
];

// NOT object-safe (returns Self)
trait Clonable {
    fn duplicate(&self) -> Self;  // Can't use as dyn Clonable
}
```

### Workaround: Associated Types or Box

```rust
// Make it object-safe with Box return
trait Handler {
    fn handle(&self, req: Request) -> Box<dyn Future<Output = Response>>;
}

// Or use associated types
trait Parser {
    type Output;
    fn parse(&self, input: &str) -> Self::Output;
}
```

---

## Common Trait Implementations

Implement these traits eagerly for public types:

| Trait | When | Why |
|-------|------|-----|
| `Debug` | Always | Required for error messages, logging |
| `Clone` | If duplication makes sense | Needed for many APIs |
| `PartialEq` / `Eq` | If comparison is meaningful | Testing, HashMap keys (Eq) |
| `Hash` | If used as map keys | Required for HashMap/HashSet |
| `Default` | If there's a sensible default | Works with `unwrap_or_default()` |
| `Display` | For user-facing types | Error messages, logging |
| `Send` / `Sync` | Auto-derived | Don't implement manually unless you must |
| `Serialize` / `Deserialize` | Behind feature flag | Don't force serde on all users |

```rust
#[derive(Debug, Clone, PartialEq, Eq, Hash)]
pub struct Config {
    pub name: String,
    pub port: u16,
}

impl Default for Config {
    fn default() -> Self {
        Config {
            name: "app".to_string(),
            port: 8080,
        }
    }
}

impl fmt::Display for Config {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "{}:{}", self.name, self.port)
    }
}
```

### Serde Behind Feature Flag

Don't force serde dependency on all users:

```toml
[features]
serde = ["dep:serde"]

[dependencies]
serde = { version = "1", features = ["derive"], optional = true }
```

```rust
#[derive(Debug, Clone)]
#[cfg_attr(feature = "serde", derive(serde::Serialize, serde::Deserialize))]
pub struct Config { ... }
```

---

## Supertraits

Express trait dependencies:

```rust
// Formatter requires Debug
trait Formatter: Debug {
    fn format(&self) -> String;
}

// Any type implementing Formatter must also implement Debug
// This is enforced at compile time
```

Useful for ensuring types have prerequisite capabilities
without repeating bounds everywhere.
