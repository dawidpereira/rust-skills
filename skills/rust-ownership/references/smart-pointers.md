# Smart Pointers

## Arc: Thread-Safe Shared Ownership

`Arc<T>` (Atomic Reference Counted) enables multiple owners
across thread boundaries. `Arc::clone()` is cheap — it
increments an atomic counter, not the data.

```rust
use std::sync::Arc;
use std::thread;

let config = Arc::new(AppConfig::load());

let handles: Vec<_> = (0..4).map(|_| {
    let config = Arc::clone(&config);  // Cheap: atomic increment
    thread::spawn(move || {
        process_with(&config);
    })
}).collect();

for handle in handles {
    handle.join().unwrap();
}
```

### Arc with Interior Mutability

For shared mutable state across threads, combine `Arc` with `Mutex` or `RwLock`:

```rust
use std::sync::{Arc, Mutex};

let counter = Arc::new(Mutex::new(0));

let handles: Vec<_> = (0..10).map(|_| {
    let counter = Arc::clone(&counter);
    thread::spawn(move || {
        let mut count = counter.lock().unwrap();
        *count += 1;
    })
}).collect();
```

### Performance

`Arc::clone()` costs one atomic increment — fast, but not
free. Avoid cloning in tight loops where the reference can
be borrowed instead:

```rust
// Bad: atomic increment per iteration
for item in &items {
    let shared = Arc::clone(&data);
    process(shared, item);
}

// Good: borrow the inner data
for item in &items {
    process(&data, item);  // Just a reference, no atomic ops
}
```

---

## Rc: Single-Thread Shared Ownership

`Rc<T>` is `Arc<T>` without atomic operations — faster,
but restricted to a single thread. The compiler enforces
this: `Rc` is `!Send + !Sync`.

```rust
use std::rc::Rc;

// Tree with shared nodes
struct Node {
    value: i32,
    children: Vec<Rc<Node>>,
}

let shared_leaf = Rc::new(Node { value: 1, children: vec![] });
let parent_a = Node { value: 2, children: vec![Rc::clone(&shared_leaf)] };
let parent_b = Node { value: 3, children: vec![Rc::clone(&shared_leaf)] };
```

**Don't use Arc when Rc suffices** — atomic operations
have overhead even when there's no contention:

```rust
// Bad: unnecessary atomic overhead in single-threaded code
let data = Arc::new(build_tree());

// Good: Rc when you know it's single-threaded
let data = Rc::new(build_tree());
```

### Decision Guide

| Scenario | Use |
|----------|-----|
| Shared ownership, multi-thread | `Arc<T>` |
| Shared ownership, single-thread | `Rc<T>` |
| Shared + mutable, multi-thread | `Arc<Mutex<T>>` or `Arc<RwLock<T>>` |
| Shared + mutable, single-thread | `Rc<RefCell<T>>` |
| Unique ownership, heap-allocated | `Box<T>` |
| No shared ownership needed | Move or borrow |

---

## Box: Heap Allocation and Large Enum Variants

`Box<T>` puts data on the heap. Primary uses: large types,
recursive types, and trait objects.

### Boxing Large Enum Variants

An enum's size equals its largest variant. One large
variant bloats every instance:

```rust
// Bad: Image variant forces all Messages to be ~1032 bytes
enum Message {
    Text(String),              // 24 bytes
    Image([u8; 1024]),         // 1024 bytes — every Message pays this cost!
    Quit,                      // 0 bytes
}

// Good: Box the large variant
enum Message {
    Text(String),              // 24 bytes
    Image(Box<[u8; 1024]>),   // 8 bytes (pointer)
    Quit,                      // 0 bytes
}
// Now Message is ~24 bytes
```

Check sizes with:

```rust
assert_eq!(std::mem::size_of::<Message>(), 32);  // Catch regressions
```

**Clippy lint:** `clippy::large_enum_variant` catches this automatically.

**When to box variants:**
- Variant is >3x the size of other variants
- Enum is stored in collections or passed frequently
- The large variant is rarely used

### Recursive Types

Recursive types require boxing because the compiler can't determine their size:

```rust
// Won't compile: infinite size
enum List<T> {
    Cons(T, List<T>),
    Nil,
}

// Works: Box provides indirection
enum List<T> {
    Cons(T, Box<List<T>>),
    Nil,
}
```

### Pattern Matching with Boxed Variants

```rust
match message {
    Message::Text(s) => println!("{s}"),
    Message::Image(data) => render(&*data),  // Deref to access inner
    Message::Quit => break,
}
```
