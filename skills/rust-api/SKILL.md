---
name: rust-api
description: >
  Rust API design, naming conventions, and documentation standards. Use when designing public
  APIs, implementing builder patterns, choosing #[must_use]/#[non_exhaustive], following Rust
  naming conventions (as_/to_/into_ prefixes), or writing doc comments with examples. Also
  use for library design decisions like common trait implementations and serde feature gating.
---

# API Design

## Core Question

**What does the caller need, and what should the compiler prevent?**

Every API decision flows from this:

- What is the minimum the caller must provide?
- What mistakes can the type system catch at compile time?
- What is the cost of each operation, and does the name
  communicate it?

If the caller can misuse your API without a compiler error,
the API needs work.

---

## API → Design Question

| Symptom                          | Don't Just Say           | Ask Instead                                 |
| -------------------------------- | ------------------------ | ------------------------------------------- |
| Constructor with 8 parameters    | "Use a builder"          | Which parameters are required vs optional?  |
| Caller ignores return value      | "Add must_use"           | Is ignoring this value ever correct?        |
| Adding enum variant breaks users | "It's a breaking change" | Should this enum be `#[non_exhaustive]`?    |
| Method named `get_name()`        | "Remove the get\_"       | Does this do more than return a field?      |
| `as_string()` allocates          | "Rename to to_string()"  | What is the actual cost of this conversion? |

---

## Quick Decisions

| Scenario                            | Use                               | Why                                  |
| ----------------------------------- | --------------------------------- | ------------------------------------ |
| Many optional fields in constructor | Builder pattern                   | Self-documenting, flexible           |
| Required + optional fields          | Typestate builder                 | Compiler enforces required fields    |
| All fields have sensible defaults   | `#[derive(Default)]`              | Works with `..Default::default()`    |
| Return value must not be ignored    | `#[must_use]`                     | Compiler warns on silent drop        |
| Builder struct or method chain      | `#[must_use]` on type + methods   | Prevents accidental drop             |
| Public enum that may grow           | `#[non_exhaustive]`               | Add variants without breaking change |
| Public struct that may grow         | `#[non_exhaustive]` + constructor | Add fields without breaking change   |
| Adding methods to external types    | Extension trait (`TypeExt`)       | Works around orphan rules            |
| Public type minimum traits          | `Debug, Clone, PartialEq`         | Basic ecosystem interop              |
| Serde in a library crate            | Feature flag, not hard dep        | Users who don't need it don't pay    |
| Free reference conversion           | `as_` prefix                      | Signals O(1), no allocation          |
| Allocating conversion               | `to_` prefix                      | Signals cost                         |
| Ownership-consuming conversion      | `into_` prefix                    | Signals self is consumed             |
| Simple field accessor               | No `get_` prefix                  | `name()` not `get_name()`            |
| Boolean-returning method            | `is_`/`has_`/`can_` prefix        | Reads naturally in conditions        |

---

## Builder Pattern

Choose the right builder variant based on your
requirements:

### Decision

| Variant                     | When                                     | `build()` returns |
| --------------------------- | ---------------------------------------- | ----------------- |
| **Infallible**              | All fields have defaults                 | `T`               |
| **Fallible**                | Validation can fail at runtime           | `Result<T, E>`    |
| **Typestate**               | Required fields enforced at compile time | `T`               |
| **Consuming** (`mut self`)  | Most common, simple chaining             | Depends           |
| **Borrowing** (`&mut self`) | Builder reused for multiple instances    | Depends           |

### Infallible Builder

```rust
#[derive(Default)]
#[must_use = "builders do nothing unless you call build()"]
pub struct WidgetBuilder {
    color: Option<Color>,
    size: Option<Size>,
}

impl WidgetBuilder {
    pub fn color(mut self, color: Color) -> Self {
        self.color = Some(color);
        self
    }

    pub fn build(self) -> Widget {
        Widget {
            color: self.color.unwrap_or(Color::Black),
            size: self.size.unwrap_or(Size::Medium),
        }
    }
}
```

### Typestate Builder (compile-time required fields)

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
```

---

## Common Traits Checklist

Derive these for every public type unless you have a
reason not to:

| Type Category                  | Derive                                                           |
| ------------------------------ | ---------------------------------------------------------------- |
| **Minimum (all public types)** | `Debug, Clone, PartialEq`                                        |
| **ID / key types**             | `Debug, Clone, Copy, PartialEq, Eq, Hash`                        |
| **Small value types**          | `Debug, Clone, Copy, PartialEq, Default`                         |
| **Config / options**           | `Debug, Clone, PartialEq, Default`                               |
| **Error types**                | `Debug, Clone, PartialEq, Eq`                                    |
| **HashMap keys**               | Add `Eq, Hash`                                                   |
| **BTreeMap keys**              | Add `Eq, Ord, PartialOrd`                                        |
| **Serde support**              | `#[cfg_attr(feature = "serde", derive(Serialize, Deserialize))]` |

Implement manually when derive does the wrong thing
(e.g., case-insensitive equality, redacting sensitive
fields in Debug).

---

## Naming Quick Reference

### Conversion Prefixes

| Prefix  | Cost               | Ownership     | Example                                      |
| ------- | ------------------ | ------------- | -------------------------------------------- |
| `as_`   | Free O(1)          | `&self -> &U` | `as_str()`, `as_bytes()`, `as_slice()`       |
| `to_`   | Allocates/computes | `&self -> U`  | `to_string()`, `to_vec()`, `to_lowercase()`  |
| `into_` | Consumes self      | `self -> U`   | `into_inner()`, `into_bytes()`, `into_vec()` |

### Accessor Naming

| Pattern             | Name                                     | Not                            |
| ------------------- | ---------------------------------------- | ------------------------------ |
| Simple field access | `name()`, `len()`                        | `get_name()`, `get_len()`      |
| Fallible lookup     | `get()`, `get_mut()`                     | `find()` (unless searching)    |
| Boolean check       | `is_empty()`, `has_key()`, `can_write()` | `empty()`, `key_exists()`      |
| Setter              | `set_name(value)`                        | `name(value)` (unless builder) |

### Iterator Methods

| Method        | Yields   | Ownership           |
| ------------- | -------- | ------------------- |
| `iter()`      | `&T`     | Borrows collection  |
| `iter_mut()`  | `&mut T` | Mutably borrows     |
| `into_iter()` | `T`      | Consumes collection |

Iterator type names match their method: `iter()` ->
`Iter`, `into_iter()` -> `IntoIter`, `keys()` -> `Keys`.

### General Rules

| Element              | Convention              | Example              | Not                  |
| -------------------- | ----------------------- | -------------------- | -------------------- |
| Types, traits, enums | `UpperCamelCase`        | `HttpServer`         | `HTTPServer`         |
| Enum variants        | `UpperCamelCase`        | `NotFound`           | `NOT_FOUND`          |
| Functions, methods   | `snake_case`            | `parse_json()`       | `parseJSON()`        |
| Constants, statics   | `SCREAMING_SNAKE_CASE`  | `MAX_RETRIES`        | `maxRetries`         |
| Lifetimes            | Short lowercase         | `'a`, `'de`, `'src`  | `'input_lifetime`    |
| Type params          | Single uppercase        | `T`, `E`, `K`, `V`   | `ElementType`        |
| Acronyms             | Treat as words          | `Uuid`, `HttpClient` | `UUID`, `HTTPClient` |
| Crate names          | No `-rs`/`-rust` suffix | `json-parser`        | `json-parser-rs`     |

---

## Usage Scenarios

### Scenario 1: Designing a Library Config Type

You need a `Config` struct with 3 required fields and 5
optional fields.

1. Use a builder with typestate for the 3 required fields
2. Add `#[must_use]` to the builder type
3. Derive `Debug, Clone, PartialEq, Default` on `Config`
4. Add `#[non_exhaustive]` if the struct is public and may
   gain fields
5. Gate serde behind a feature flag
6. Document with `# Examples` showing builder usage

### Scenario 2: Adding Conversion Methods to a Newtype

You have `struct Email(String)`:

```rust
impl Email {
    /// Returns the email as a string slice. O(1), no allocation.
    pub fn as_str(&self) -> &str {
        &self.0
    }

    /// Returns a new lowercase version. Allocates.
    pub fn to_lowercase(&self) -> Email {
        Email(self.0.to_lowercase())
    }

    /// Consumes the Email, returning the inner String.
    pub fn into_string(self) -> String {
        self.0
    }

    pub fn is_valid(&self) -> bool {
        self.0.contains('@')
    }
}
```

### Scenario 3: Extending an External Type

You need hex encoding for byte slices:

```rust
pub trait ByteSliceExt {
    fn as_hex(&self) -> String;
}

impl ByteSliceExt for [u8] {
    fn as_hex(&self) -> String {
        self.iter().map(|b| format!("{b:02x}")).collect()
    }
}
```

Import `use my_crate::ByteSliceExt;` to use. Name the
trait with `Ext` suffix.

---

## Reference Index

| Reference                                       | Read When                                                                                                             |
| ----------------------------------------------- | --------------------------------------------------------------------------------------------------------------------- |
| [patterns.md](references/patterns.md)           | Implementing builders, choosing #[must_use]/#[non_exhaustive], extension traits, Default, common traits, serde gating |
| [naming.md](references/naming.md)               | Naming methods, types, conversions, iterators, or crate names                                                         |
| [documentation.md](references/documentation.md) | Writing doc comments, examples, error/panic/safety sections, intra-doc links, Cargo.toml metadata                     |

---

## Cross-References

| Need                                     | Skill          |
| ---------------------------------------- | -------------- |
| Error types for Result-returning APIs    | rust-errors    |
| Newtype patterns, typestate, PhantomData | rust-types     |
| Trait design, generics, dispatch         | rust-types     |
| Testing doc examples                     | rust-quality   |
| Clippy lints for API quality             | rust-quality   |
| Ownership decisions in API signatures    | rust-ownership |
