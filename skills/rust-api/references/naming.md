# Naming Conventions Reference

## Casing Rules

| Element | Convention | Example | Bad Example |
|---------|------------|---------|-------------|
| Types, traits, enums | `UpperCamelCase` | `HttpServer`, `JsonParser` | `HTTPServer`, `http_server` |
| Enum variants | `UpperCamelCase` | `NotFound`, `InProgress` | `NOT_FOUND`, `not_found` |
| Functions, methods | `snake_case` | `parse_json()`, `send_request()` | `parseJSON()`, `SendRequest()` |
| Variables, parameters | `snake_case` | `user_count`, `max_retries` | `userCount`, `MaxRetries` |
| Modules | `snake_case` | `user_service`, `http_client` | `UserService`, `httpClient` |
| Constants, statics | `SCREAMING_SNAKE_CASE` | `MAX_CONNECTIONS`, `BUFFER_SIZE` | `maxConnections`, `BufferSize` |
| Lifetimes | Short lowercase | `'a`, `'b`, `'de`, `'src` | `'input_lifetime` |
| Type parameters | Single uppercase | `T`, `E`, `K`, `V`, `F` | `ElementType`, `ErrorType` |

All of these are enforced by the compiler or clippy.
Violating them produces warnings.

---

## Acronyms

Treat acronyms as single words — capitalize only the
first letter.

| Good | Bad | Reason |
|------|-----|--------|
| `HttpServer` | `HTTPServer` | Word boundary unclear: `HTTPS`erver? |
| `JsonParser` | `JSONParser` | Consistent CamelCase boundaries |
| `Uuid` | `UUID` | Follows std convention (`std::net::TcpStream`) |
| `TcpIpConnection` | `TCPIPConnection` | Two acronyms: `Tcp` + `Ip` stay clear |
| `parse_json()` | `parse_JSON()` | snake_case = all lowercase |
| `generate_uuid()` | `generate_UUID()` | Acronyms lowercase in snake_case |

Two-letter acronyms (IO, ID) can go either way, but
`Io`, `Id` is preferred for consistency.

---

## Conversion Prefixes

| Prefix | Cost | Signature | Example | Std Example |
|--------|------|-----------|---------|-------------|
| `as_` | Free, O(1) | `&self -> &U` | `as_str()`, `as_inner()` | `String::as_str()`, `Vec::as_slice()` |
| `to_` | Allocates or computes | `&self -> U` | `to_lowercase()`, `to_vec()` | `str::to_string()`, `[T]::to_vec()` |
| `into_` | Consumes self (usually cheap) | `self -> U` | `into_inner()`, `into_bytes()` | `String::into_bytes()`, `Vec::into_boxed_slice()` |

### Rules

- `as_` MUST be free. If it allocates, rename to `to_`.
- `to_` signals "you are paying a cost." Caller should
  consider caching.
- `into_` signals "the original is gone." Caller cannot
  use `self` afterward.

### Bad

```rust
pub fn as_string(&self) -> String {   // Allocates! Should be to_string()
    format!("{}", self.value)
}

pub fn get_inner(self) -> Inner {     // Consumes self! Should be into_inner()
    self.inner
}
```

### Good

```rust
pub fn as_str(&self) -> &str { &self.inner }
pub fn to_string_lossy(&self) -> String { format!("{}", self.value) }
pub fn into_inner(self) -> Inner { self.inner }
```

---

## Getter / Setter Naming

| Pattern | Convention | Example |
|---------|-----------|---------|
| Simple field access | No `get_` prefix | `name()`, `len()`, `capacity()` |
| Fallible access (returns Option) | `get()` is acceptable | `HashMap::get()`, `Vec::get()` |
| Boolean property | `is_`/`has_`/`can_`/`should_` prefix | `is_empty()`, `has_key()` |
| Setter | `set_` prefix | `set_name(value)` |
| Builder method | No prefix, consumes self | `name(mut self, n: String) -> Self` |

### Bad

```rust
fn get_name(&self) -> &str { &self.name }     // Verbose for simple access
fn get_is_active(&self) -> bool { self.active } // Double prefix
fn active(&self) -> bool { self.active }        // Ambiguous: check or activate?
```

### Good

```rust
fn name(&self) -> &str { &self.name }
fn is_active(&self) -> bool { self.active }
fn set_name(&mut self, name: String) { self.name = name; }
```

---

## Boolean Method Prefixes

| Prefix | Use For | Examples |
|--------|---------|---------|
| `is_` | State or property | `is_empty()`, `is_valid()`, `is_some()`, `is_active()` |
| `has_` | Possession or containment | `has_key()`, `has_children()`, `has_remaining()` |
| `can_` | Capability or permission | `can_read()`, `can_write()`, `can_execute()` |
| `should_` | Policy or recommendation | `should_retry()`, `should_cache()` |
| `needs_` | Requirement | `needs_update()`, `needs_auth()` |

Prefer positive form. Use `!is_active()` rather than
creating `is_inactive()`.

---

## Iterator Naming

### The Three Standard Methods

| Method | Receiver | Yields | Type Name |
|--------|----------|--------|-----------|
| `iter()` | `&self` | `&T` | `Iter` |
| `iter_mut()` | `&mut self` | `&mut T` | `IterMut` |
| `into_iter()` | `self` | `T` | `IntoIter` |

### Additional Iterator Methods

| Method | Type Name | Yields |
|--------|-----------|--------|
| `keys()` | `Keys` | `&K` |
| `values()` | `Values` | `&V` |
| `values_mut()` | `ValuesMut` | `&mut V` |
| `drain()` | `Drain` | `T` |
| `chunks()` | `Chunks` | `&[T]` |
| `windows()` | `Windows` | `&[T]` |

### Rules

- Iterator type names MUST match their creation method.
- Implement `IntoIterator` for `T`, `&T`, and `&mut T` to
  enable `for` loop syntax.
- Do NOT use non-standard names like `elements()`,
  `get_iterator()`, or `to_iter()`.

### Bad

```rust
fn elements(&self) -> impl Iterator<Item = &T> { /* ... */ }  // Should be iter()
fn to_iter(self) -> impl Iterator<Item = T> { /* ... */ }     // Should be into_iter()
```

---

## Lifetime Parameters

| Name | Convention | Example |
|------|-----------|---------|
| `'a` | Generic, first lifetime | `fn foo<'a>(x: &'a str)` |
| `'b` | Generic, second lifetime | `fn bar<'a, 'b>(x: &'a T, y: &'b U)` |
| `'de` | Deserialization (serde) | `Deserialize<'de>` |
| `'src` | Source code / input text | `struct Lexer<'src>` |
| `'ctx` | Context reference | `struct Query<'ctx>` |

Prefer lifetime elision when the compiler can infer.
Write `fn name(&self) -> &str` not
`fn name<'a>(&'a self) -> &'a str`.

---

## Type Parameters

| Param | Meaning | Usage |
|-------|---------|-------|
| `T` | Generic type | `Vec<T>`, `Option<T>` |
| `E` | Error | `Result<T, E>` |
| `K` | Key | `HashMap<K, V>` |
| `V` | Value | `HashMap<K, V>` |
| `I` | Input / Item | `Iterator<Item = I>` |
| `O` | Output | `Fn(I) -> O` |
| `F` | Function / Closure | `map<F>(f: F)` |
| `S` | State | `StateMachine<S>` |
| `A` | Allocator | `Vec<T, A>` |
| `R` | Return / Result | `fn() -> R` |

Use single letters. Move complex trait bounds to `where`
clauses.

---

## Crate Names

| Rule | Good | Bad |
|------|------|----|
| No `-rs` or `-rust` suffix | `json-parser` | `json-parser-rs` |
| No `rust-` prefix | `http-client` | `rust-http-client` |
| Descriptive of function | `sqlite-wrapper` | `my-lib-rust` |

You are already on crates.io. The `-rs` suffix adds
nothing.
