# Documentation Reference

## Document All Public Items

Use `///` for every public struct, enum, function, trait,
constant, and field. Use `//!` for module-level and
crate-level documentation.

Enable the lint to catch misses:

```rust
#![warn(missing_docs)]
```

### What to Document

| Item          | Required Content                       |
| ------------- | -------------------------------------- |
| Structs       | Purpose, when to use, example          |
| Struct fields | What the field represents              |
| Enums         | When to use, how variants relate       |
| Enum variants | What state/case it represents          |
| Functions     | What it does, parameters, return value |
| Traits        | The contract and expected behavior     |
| Constants     | What the value represents and why      |

---

## Module Documentation with `//!`

Place `//!` at the top of the file. It documents the
module itself, not the next item. This is the first thing
users see in `cargo doc`.

````rust
//! Authentication and authorization utilities.
//!
//! This module provides multiple authentication strategies:
//!
//! - [`JwtAuth`] - JSON Web Token based authentication
//! - [`SessionAuth`] - Cookie-based session authentication
//!
//! # Examples
//!
//! ```
//! use my_crate::auth::{JwtAuth, Authenticator};
//!
//! let auth = JwtAuth::new("secret-key");
//! let token = auth.generate_token(&user)?;
//! # Ok::<(), my_crate::Error>(())
//! ```

use std::collections::HashMap;
````

For `lib.rs`, the `//!` block becomes the crate-level
documentation shown on the crate root page.

---

## # Examples Section

Include runnable examples for every public function and
type. Doc examples are compiled and tested by
`cargo test --doc`, keeping them in sync with the code.

### Bad: No Examples

```rust
/// Parses a string into a Foo.
pub fn parse(s: &str) -> Result<Foo, Error> { /* ... */ }
```

### Good: Runnable Example

````rust
/// Parses a string into a Foo.
///
/// # Examples
///
/// ```
/// use my_crate::parse;
///
/// let foo = parse("hello")?;
/// assert_eq!(foo.name(), "hello");
/// # Ok::<(), my_crate::Error>(())
/// ```
pub fn parse(s: &str) -> Result<Foo, Error> { /* ... */ }
````

Show multiple examples when behavior varies (success case,
error case, edge case).

---

## # Errors Section

Every function returning `Result` must document when and
why it fails.

### Bad: No Error Documentation

```rust
/// Opens a file and reads its contents.
pub fn read_file(path: &Path) -> Result<String, Error> { /* ... */ }
```

### Good: Specific Error Conditions

```rust
/// Opens a file and reads its contents as a UTF-8 string.
///
/// # Errors
///
/// Returns an error if:
/// - The file does not exist ([`Error::NotFound`])
/// - The process lacks read permission ([`Error::PermissionDenied`])
/// - The file contains invalid UTF-8 ([`Error::InvalidUtf8`])
pub fn read_file(path: &Path) -> Result<String, Error> { /* ... */ }
```

Link to specific error variants with intra-doc links so
users can click through to the variant documentation.

---

## # Panics Section

Document every condition under which a function panics.
Users need this to ensure preconditions are met and to
avoid calling panicking functions in contexts where panics
are unacceptable.

### Bad: Undocumented Panic

```rust
/// Returns the element at the given index.
pub fn get(&self, index: usize) -> &T {
    &self.data[index]  // Panics on out-of-bounds, not documented
}
```

### Good: Documented Panic with Alternative

```rust
/// Returns the element at the given index.
///
/// # Panics
///
/// Panics if `index >= self.len()`.
///
/// For a non-panicking version, use [`try_get`](Self::try_get) which
/// returns `Option<&T>`.
pub fn get(&self, index: usize) -> &T {
    &self.data[index]
}
```

Common panic conditions to document: index out of bounds,
division by zero, `.unwrap()` / `.expect()` calls,
`assert!` failures.

---

## # Safety Section

Required for all `unsafe fn` declarations and
`unsafe trait` definitions. Document exactly what the
caller must guarantee.

### Bad: No Safety Contract

```rust
/// Reads a value from a raw pointer.
pub unsafe fn read_ptr<T>(ptr: *const T) -> T {
    ptr.read()
}
```

### Good: Explicit Safety Requirements

```rust
/// Reads a value from a raw pointer.
///
/// # Safety
///
/// The caller must ensure that:
/// - `ptr` is valid for reads of `size_of::<T>()` bytes
/// - `ptr` is properly aligned for type `T`
/// - `ptr` points to a properly initialized value of type `T`
/// - The memory referenced by `ptr` is not mutated during this call
pub unsafe fn read_ptr<T>(ptr: *const T) -> T {
    ptr.read()
}
```

For `unsafe` blocks inside safe functions, use
`// SAFETY:` comments:

```rust
pub fn get(&self, index: usize) -> Option<&T> {
    if index < self.len {
        // SAFETY: index < len guarantees this access is in bounds.
        Some(unsafe { self.data.get_unchecked(index) })
    } else {
        None
    }
}
```

---

## Use `?` in Examples, Not `.unwrap()`

Doc examples model best practices. Use `?` for error
propagation.

### Bad: Teaching unwrap

````rust
/// # Examples
///
/// ```
/// let config = Config::from_file("config.toml").unwrap();
/// ```
````

### Good: Using ? with Hidden Wrapper

````rust
/// # Examples
///
/// ```
/// # fn main() -> Result<(), Box<dyn std::error::Error>> {
/// let config = Config::from_file("config.toml")?;
/// println!("{:?}", config);
/// # Ok(())
/// # }
/// ```
````

Or with the trailing `Ok` pattern (Rust 2021+):

````rust
/// ```
/// # use my_crate::{Config, Error};
/// let config = Config::from_file("config.toml")?;
/// # Ok::<(), Error>(())
/// ```
````

Exception: `.unwrap()` is acceptable for values known to
be infallible at compile time (e.g.,
`Regex::new(r"^\d+$").unwrap()` with a literal pattern).

---

## Hide Setup Code with `#`

Lines starting with `#` (hash space) in doc examples are
compiled but hidden from rendered documentation. Use this
to keep examples focused.

### Bad: Setup Buries the Point

````rust
/// # Examples
///
/// ```
/// use my_crate::{Processor, Config, Item};
/// use std::sync::Arc;
///
/// let config = Config {
///     batch_size: 100,
///     timeout_ms: 5000,
///     retry_count: 3,
/// };
/// let processor = Processor::new(Arc::new(config));
/// let items = vec![Item::new("a"), Item::new("b")];
///
/// let results = processor.process_batch(&items)?;
/// assert!(results.all_succeeded());
/// # Ok::<(), my_crate::Error>(())
/// ```
````

### Good: Hidden Setup, Visible API

````rust
/// # Examples
///
/// ```
/// # use my_crate::{Processor, Config, Item, Error};
/// # use std::sync::Arc;
/// # let config = Config { batch_size: 100, timeout_ms: 5000, retry_count: 3 };
/// # let processor = Processor::new(Arc::new(config));
/// # let items = vec![Item::new("a"), Item::new("b")];
/// let results = processor.process_batch(&items)?;
/// assert!(results.all_succeeded());
/// # Ok::<(), Error>(())
/// ```
````

Do NOT hide setup when the setup IS the example (e.g.,
documenting a builder).

### `no_run` and `ignore`

- ` ```no_run ` — Compiles but does not execute (for code
  that starts servers, writes files, etc.)
- ` ```ignore ` — Neither compiled nor executed (for
  pseudocode or incomplete examples)

---

## Intra-Doc Links

Use `[TypeName]` syntax to create clickable,
compiler-verified links in docs.

### Link Syntax

| Syntax           | Target                 | Example                 |
| ---------------- | ---------------------- | ----------------------- |
| `[Name]`         | Item in scope          | `[Vec]`, `[Option]`     |
| `[path::Name]`   | Qualified path         | `[std::vec::Vec]`       |
| `[Self::method]` | Method on current type | `[Self::new]`           |
| `[Type::method]` | Method on another type | `[String::new]`         |
| `[text](path)`   | Custom link text       | `[see here](Self::len)` |

### Disambiguation

When a name refers to multiple items, add a suffix:

| Suffix    | Item       |
| --------- | ---------- |
| `fn@`     | Function   |
| `mod@`    | Module     |
| `struct@` | Struct     |
| `enum@`   | Enum       |
| `trait@`  | Trait      |
| `type@`   | Type alias |

Example: `[Error](struct@Error)` vs `[Error](trait@Error)`

### Verify Links in CI

```bash
RUSTDOCFLAGS="-D warnings" cargo doc --no-deps
```

Or in `Cargo.toml`:

```toml
[lints.rustdoc]
broken_intra_doc_links = "deny"
```

---

## Cargo.toml Metadata

Fill metadata for published crates. This appears on
crates.io and affects discoverability.

### Required for Publishing

| Field                       | Purpose                                       |
| --------------------------- | --------------------------------------------- |
| `name`                      | Crate name                                    |
| `version`                   | Semver version                                |
| `license` or `license-file` | SPDX identifier (e.g., `"MIT OR Apache-2.0"`) |
| `description`               | One-line summary, max 256 characters          |

### Recommended

| Field           | Purpose              | Example                          |
| --------------- | -------------------- | -------------------------------- |
| `repository`    | Source code link     | `"https://github.com/user/repo"` |
| `documentation` | Docs link            | `"https://docs.rs/my-crate"`     |
| `readme`        | Path to README       | `"README.md"`                    |
| `keywords`      | Search terms (max 5) | `["http", "async", "client"]`    |
| `categories`    | crates.io categories | `["network-programming"]`        |
| `rust-version`  | MSRV                 | `"1.70"`                         |
| `edition`       | Rust edition         | `"2021"`                         |

### Bad

```toml
[package]
name = "my-crate"
version = "0.1.0"
edition = "2021"
```

### Good

```toml
[package]
name = "my-crate"
version = "0.1.0"
edition = "2021"
rust-version = "1.70"
description = "A fast, ergonomic HTTP client for Rust"
license = "MIT OR Apache-2.0"
repository = "https://github.com/user/my-crate"
documentation = "https://docs.rs/my-crate"
readme = "README.md"
keywords = ["http", "client", "async"]
categories = ["network-programming", "web-programming::http-client"]
```

Verify before publishing:

```bash
cargo package --list   # See what will be included
cargo publish --dry-run  # Check metadata completeness
```
