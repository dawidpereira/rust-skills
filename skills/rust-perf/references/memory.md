# Memory Optimization Reference

## Pre-Allocate with `with_capacity()`

Use `with_capacity()` when the final size is known or
estimable. Avoids repeated reallocations (Vec grows
through 0, 4, 8, 16, 32...).

```rust
// Bad: ~10 reallocations for 1000 elements
let mut results = Vec::new();
for i in 0..1000 {
    results.push(process(i));
}

// Good: zero reallocations
let mut results = Vec::with_capacity(1000);
for i in 0..1000 {
    results.push(process(i));
}

// Good: collect() uses size_hint automatically
let results: Vec<_> = (0..1000).map(process).collect();
```

Applies to `String::with_capacity()`,
`HashMap::with_capacity()`, and
`VecDeque::with_capacity()`.

For estimated sizes (e.g., ~10% pass a filter), use
`Vec::with_capacity(items.len() / 10)`.

---

## SmallVec — Stack-Allocated Small Collections

`SmallVec<[T; N]>` stores up to N elements inline on the
stack. Falls back to heap when exceeded. Use when most
instances are small.

```rust
use smallvec::{smallvec, SmallVec};

// Bad: always heap-allocates, even for 1-2 elements
fn get_path_components(path: &str) -> Vec<&str> {
    path.split('/').collect()
}

// Good: stack-allocated for typical paths
fn get_path_components(path: &str) -> SmallVec<[&str; 8]> {
    path.split('/').collect()
}
```

Choosing N: measure your actual data distribution. Common
values: 4 for AST children, 8 for path
components/function args, 2-4 for error lists.

Trade-off: SmallVec is slightly larger than Vec (branching
overhead on every operation). Profile to verify benefit.

---

## ArrayVec — Fixed Capacity, No Heap Ever

`ArrayVec<T, N>` guarantees no heap allocation. Panics or
returns Err on overflow. Use for hard upper bounds,
embedded, or `no_std`.

```rust
use arrayvec::ArrayVec;

fn parse_rgb(s: &str) -> ArrayVec<u8, 3> {
    let mut components = ArrayVec::new();
    for part in s.split(',').take(3) {
        components.push(part.parse().unwrap());
    }
    components
}
```

Do not use when size varies widely or capacity would be
large (stack overflow risk).

| Type | Stack | Heap | Use When |
|------|-------|------|----------|
| `Vec<T>` | Never | Always | Unknown size, may grow |
| `SmallVec<[T; N]>` | Up to N | Beyond N | Usually small, occasionally large |
| `ArrayVec<T, N>` | Always | Never | Hard limit, no heap allowed |

---

## ThinVec — Minimal Overhead for Often-Empty Vectors

`ThinVec<T>` uses a single pointer (8 bytes) vs Vec's 24
bytes. Empty ThinVec is a null pointer. Option<ThinVec>
gets niche optimization for free.

```rust
use thin_vec::ThinVec;

struct TreeNode {
    value: i32,
    children: ThinVec<TreeNode>,  // 8 bytes, most leaves are empty
}
```

Use when many instances exist and most are empty (tree
nodes, sparse graphs). Avoid in hot iteration loops —
pointer indirection on every length check.

---

## Box Large Enum Variants

An enum's size equals its largest variant. Box the outlier
to keep the enum small.

```rust
// Bad: every Message is ~1032 bytes, even Quit
enum Message {
    Quit,
    Text(String),
    Image { data: [u8; 1024], width: u32, height: u32 },
}

// Good: Message is ~32 bytes
enum Message {
    Quit,
    Text(String),
    Image(Box<ImageData>),
}
```

Rule of thumb: box variants > 128 bytes when other
variants are much smaller. Enable clippy
`large_enum_variant` lint.

---

## Box<[T]> and Box<str> for Fixed-Size Data

`Box<[T]>` saves 8 bytes per instance vs Vec (no capacity
field). Signals "this data won't grow."

```rust
struct Document {
    paragraphs: Box<[Paragraph]>,  // 16 bytes vs 24 for Vec
}

let paragraphs: Vec<Paragraph> = parse(data);
let doc = Document {
    paragraphs: paragraphs.into_boxed_slice(),
};
```

Same for `Box<str>` vs `String` — saves 8 bytes for
immutable strings. Use when storing millions of fixed
collections.

---

## clone_from() — Reuse Allocations When Cloning

`x = y.clone()` drops x's allocation and creates new. `x.clone_from(&y)` reuses x's buffer.

```rust
// Bad: allocates every iteration
for source in sources {
    buffer = source.clone();
    process(&buffer);
}

// Good: reuses allocation if capacity sufficient
for source in sources {
    buffer.clone_from(source);
    process(&buffer);
}
```

Benefits String, Vec, HashMap, PathBuf. Typically 2-3x faster in loops.

---

## Reuse Collections with clear()

Move collection declarations outside loops. Call `.clear()` to reset length while keeping capacity.

```rust
let mut temp = Vec::new();
for batch in batches {
    temp.clear();  // keeps allocation
    for item in &batch.items {
        temp.push(transform(item));
    }
    results.push(aggregate(&temp));
}
```

Works for Vec, String, HashMap, HashSet. Use BufWriter with reusable formatting buffers for I/O.

---

## write!() Over format!()

`format!()` allocates a new String every call. `write!()` writes into an existing buffer.

```rust
use std::fmt::Write;

// Bad: allocates per iteration (~500ns)
for i in 0..1000 {
    let s = format!("item-{}", i);
    process(&s);
}

// Good: reuses buffer (~50ns)
let mut buf = String::with_capacity(32);
for i in 0..1000 {
    buf.clear();
    write!(&mut buf, "item-{}", i).unwrap();
    process(&buf);
}
```

`format!()` is fine on cold paths (startup, error messages, one-time operations).

---

## Arena Allocators

Arena (bump) allocators allocate by bumping a pointer (~1-2ns vs ~25-50ns for standard). All memory freed at once when the arena drops.

```rust
use bumpalo::Bump;

fn parse<'a>(input: &str, arena: &'a Bump) -> Vec<&'a Node> {
    let mut nodes = Vec::new();
    for token in tokenize(input) {
        nodes.push(arena.alloc(Node::new(token)));
    }
    nodes
}
```

Use for: parsing/AST, request-scoped data, batch processing. Thread-local scratch arenas avoid cross-thread issues. Data cannot outlive the arena without copying out.

---

## Zero-Copy Patterns

Work with references to original data instead of copying.

```rust
// Bad: copies every parsed field
struct Parsed { name: String, value: String }

// Good: references into original string
struct Parsed<'a> { name: &'a str, value: &'a str }

fn parse(input: &str) -> Parsed<'_> {
    let (name, value) = input.split_once('=').unwrap();
    Parsed { name, value }
}
```

Use `Cow<'a, str>` for "borrow when possible, own when necessary." Use `bytes::Bytes` for zero-copy slicing with reference counting.

---

## CompactString for Small String Optimization

`CompactString` stores strings <= 24 bytes inline (no heap). Same size as String but avoids allocation for short strings.

```rust
use compact_str::CompactString;

struct User {
    username: CompactString,  // Most usernames fit inline
    email: CompactString,
}
```

Alternatives: `smartstring` (23 bytes inline), `ecow::EcoString` (16 bytes, O(1) clone). Avoid at API boundaries — forces dependency on callers.

---

## Smaller Integer Types

Use the smallest type that fits. `u8` for 0-255, `u16` for ports/small indices, `u32` for most counts.

```rust
// Bad: 32 bytes per pixel
struct Pixel { r: u64, g: u64, b: u64, a: u64 }

// Good: 4 bytes per pixel
struct Pixel { r: u8, g: u8, b: u8, a: u8 }
```

Order struct fields largest-to-smallest to minimize padding. Use `NonZeroU64` for free Option niche optimization. Use `bitflags` to pack booleans.

---

## Assert Type Sizes

Guard against accidental size growth with compile-time assertions.

```rust
struct Event {
    timestamp: u64,
    kind: EventKind,
    payload: [u8; 32],
}

const _: () = assert!(std::mem::size_of::<Event>() == 48);
```

Add assertions for types stored in large collections, FFI/protocol types, and performance-critical hot-path types. Someone adding a field will trigger a compile error, making size changes intentional.
