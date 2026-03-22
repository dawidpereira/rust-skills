# Borrowing Patterns

## Prefer References Over Cloning

Cloning allocates new memory and copies data. Borrowing is
free. Before reaching for `.clone()`, ask whether a
reference would work.

```rust
// Bad: unnecessary clone
fn process(data: &String) {
    let local = data.clone();
    println!("{}", local);
}

// Good: just borrow
fn process(data: &str) {
    println!("{}", data);
}
```

In loops, the cost multiplies — N iterations means N allocations:

```rust
// Bad: clone per iteration
for item in items {
    let copy = item.clone();
    handle(&copy);
}

// Good: borrow directly
for item in items {
    handle(item);
}
```

### When Clone Is Acceptable

Clone is the right choice when you genuinely need a separate owned copy:

```rust
// Storing owned data (HashMap values, struct fields)
cache.insert(key.to_string(), value.to_string());

// Sending across threads (need 'static)
let owned = config.clone();
std::thread::spawn(move || use_config(owned));

// Copy types (i32, bool, f64) — these are free
let x: i32 = 42;
let y = x;  // Copy, not clone
```

**Detection:** Clippy catches many unnecessary clones
with `redundant_clone` and `clone_on_copy`.

---

## Accept Slices, Not Owned Collections

Functions should accept `&[T]` not `&Vec<T>`, and `&str`
not `&String`. This makes the API more flexible through
deref coercion.

```rust
// Bad: only accepts &Vec<i32>
fn sum(numbers: &Vec<i32>) -> i32 {
    numbers.iter().sum()
}

// Good: accepts Vec, array, slice, or any &[i32]
fn sum(numbers: &[i32]) -> i32 {
    numbers.iter().sum()
}
```

The same applies to strings:

```rust
// Bad: only accepts &String
fn greet(name: &String) {
    println!("Hello, {name}");
}

// Good: accepts String, &str, string literals, Cow<str>
fn greet(name: &str) {
    println!("Hello, {name}");
}
```

**Deref coercion chain:**
- `&String` → `&str`
- `&Vec<T>` → `&[T]`
- `&Box<T>` → `&T`
- `&PathBuf` → `&Path`

For maximum flexibility, use `impl AsRef<str>` or `impl AsRef<[T]>`:

```rust
fn read_file(path: impl AsRef<Path>) -> io::Result<String> {
    std::fs::read_to_string(path.as_ref())
}

// Accepts &str, String, PathBuf, &Path...
read_file("config.toml");
read_file(PathBuf::from("/etc/config"));
```

---

## Use Cow for Conditional Ownership

`Cow<'a, T>` (Clone-on-Write) holds either a borrowed
reference or an owned value. It allocates only when
mutation is needed.

```rust
use std::borrow::Cow;

fn normalize_path(path: &str) -> Cow<'_, str> {
    if path.contains("//") {
        // Need to modify — allocates
        Cow::Owned(path.replace("//", "/"))
    } else {
        // No change needed — zero allocation
        Cow::Borrowed(path)
    }
}
```

This is powerful when most inputs pass through unchanged:

```rust
fn format_error(msg: &str, code: Option<u32>) -> Cow<'_, str> {
    match code {
        Some(c) => Cow::Owned(format!("[E{c:04}] {msg}")),
        None => Cow::Borrowed(msg),
    }
}
```

**When to use Cow:**

| Scenario | Use Cow? |
|----------|----------|
| Most inputs pass through unchanged | Yes |
| Always need to modify | No — return `String` |
| Never modify | No — return `&str` |
| Collecting into a container | Maybe — depends on mutation frequency |

Evidence: ripgrep uses `Cow` extensively in path handling
to avoid allocations in the common case.

---

## Make Clone Explicit

`Clone` signals that duplication has a cost. Unlike `Copy`
(which is implicit and free for small types), `Clone`
typically involves heap allocation. Make it visible.

```rust
// Bad: hidden cost
let config = get_config().clone();
let name = user.name.clone();
let items = list.clone();

// Good: use references where possible
let config = get_config();  // Do we really need to own this?
let name = &user.name;      // Just reading it
process(&list);             // Pass by reference
```

When you do need `Clone`, consider `clone_from()` to reuse existing allocations:

```rust
let mut buffer = String::with_capacity(1024);
for line in lines {
    buffer.clone_from(line);  // Reuses buffer's allocation
    process(&buffer);
}
```

**Alternatives to cloning:**

| Instead of Clone | Consider |
|-----------------|----------|
| Read-only access | `&T` reference |
| Conditional modification | `Cow<'a, T>` |
| Shared ownership | `Rc<T>` or `Arc<T>` |
| Passing to a function | Pass by value if caller is done with it |

---

## Derive Copy for Small Types

For small, trivial types (≤16 bytes), implement `Copy` to
enable implicit, free duplication.

```rust
// Good: small, all fields are Copy
#[derive(Debug, Clone, Copy, PartialEq)]
struct Point {
    x: f64,
    y: f64,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
struct Color {
    r: u8,
    g: u8,
    b: u8,
    a: u8,
}
```

**Copy requirements:**
- All fields must implement `Copy`
- No custom `Drop` implementation
- No heap-allocated data (`String`, `Vec`, `Box` are not `Copy`)

**Size guidelines:**
- ≤16 bytes → implement `Copy`
- 17-64 bytes → consider case by case (if passed frequently, maybe)
- \>64 bytes → don't implement `Copy`

---

## Move Large Data Instead of Cloning

For large types (hundreds of bytes), moving is free but
cloning is expensive. Design APIs to take ownership
instead of requiring clones.

```rust
// Bad: forces caller to clone large struct
fn process(data: &LargeStruct) {
    let owned = data.clone();  // Expensive!
    store(owned);
}

// Good: take ownership, caller moves
fn process(data: LargeStruct) {
    store(data);  // Zero-cost move
}
```

For very large types passed frequently, consider boxing:

```rust
// Move cost is always 8 bytes (pointer size)
fn process(data: Box<LargeStruct>) {
    // ...
}
```

---

## Lifetime Elision Patterns

Rely on elision when the rules cover your case — explicit
lifetimes add noise without value.

```rust
// Bad: unnecessary annotations
fn first_word<'a>(s: &'a str) -> &'a str { ... }
fn longest_line<'a>(s: &'a str) -> &'a str { ... }

// Good: elision handles these
fn first_word(s: &str) -> &str { ... }
fn longest_line(s: &str) -> &str { ... }
```

**When explicit lifetimes are required:**

```rust
// Multiple input references — compiler can't guess which output borrows from
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() { x } else { y }
}

// Struct holding a reference
struct Parser<'a> {
    input: &'a str,
}

// Multiple distinct lifetimes
fn split<'a, 'b>(input: &'a str, delimiter: &'b str) -> Vec<&'a str> { ... }
```

Use anonymous lifetime `'_` when the compiler needs a hint
but the specific lifetime doesn't matter:

```rust
impl fmt::Display for Wrapper<'_> {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "{}", self.inner)
    }
}
```
