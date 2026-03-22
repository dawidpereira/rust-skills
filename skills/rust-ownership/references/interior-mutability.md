# Interior Mutability

Interior mutability lets you mutate data through a shared
reference (`&self`). The borrow rules are still enforced
— just at runtime instead of compile time.

## RefCell: Single-Thread Runtime Borrow Checking

`RefCell<T>` moves borrow checking from compile time to
runtime. Use it when the compiler can't prove your borrows
are safe but you know they are.

```rust
use std::cell::RefCell;

struct Cache {
    data: RefCell<HashMap<String, String>>,
}

impl Cache {
    fn get_or_compute(&self, key: &str) -> String {
        // Borrow immutably to check
        if let Some(val) = self.data.borrow().get(key) {
            return val.clone();
        }
        // Borrow mutably to insert
        let val = expensive_compute(key);
        self.data.borrow_mut().insert(key.to_string(), val.clone());
        val
    }
}
```

### Common Pattern: Rc<RefCell<T>>

Shared mutable state in single-threaded code:

```rust
use std::rc::Rc;
use std::cell::RefCell;

let state = Rc::new(RefCell::new(AppState::default()));

let handler_a = {
    let state = Rc::clone(&state);
    move || { state.borrow_mut().count += 1; }
};

let handler_b = {
    let state = Rc::clone(&state);
    move || { println!("Count: {}", state.borrow().count); }
};
```

### Runtime Panics

`RefCell` panics if you violate borrowing rules at runtime
(e.g., borrowing mutably while already borrowed):

```rust
let cell = RefCell::new(42);
let r1 = cell.borrow();      // Immutable borrow
let r2 = cell.borrow_mut();  // PANIC: already borrowed immutably
```

Use `try_borrow()` and `try_borrow_mut()` for fallible borrowing:

```rust
if let Ok(mut val) = cell.try_borrow_mut() {
    *val += 1;
} else {
    // Handle contention gracefully
}
```

---

## Mutex: Thread-Safe Mutual Exclusion

`Mutex<T>` provides exclusive access across threads. Only
one thread can hold the lock at a time.

```rust
use std::sync::{Arc, Mutex};

let shared = Arc::new(Mutex::new(Vec::new()));

let handles: Vec<_> = (0..4).map(|i| {
    let shared = Arc::clone(&shared);
    std::thread::spawn(move || {
        let mut vec = shared.lock().unwrap();
        vec.push(i);
    })
}).collect();

for h in handles { h.join().unwrap(); }
```

### Mutex Poisoning

If a thread panics while holding a lock, the mutex becomes
"poisoned." Subsequent `lock()` calls return `Err`:

```rust
match mutex.lock() {
    Ok(guard) => { /* use guard */ }
    Err(poisoned) => {
        // Recover: get the data despite poisoning
        let guard = poisoned.into_inner();
        // ... use with caution
    }
}
```

### Consider parking_lot::Mutex

For performance-critical code, `parking_lot::Mutex` offers:

- No poisoning (simpler API, no `unwrap()`)
- Smaller size (1 byte vs 40+ bytes for std)
- Fair locking
- Better performance under contention

```rust
use parking_lot::Mutex;

let data = Arc::new(Mutex::new(0));
*data.lock() += 1;  // No unwrap needed
```

---

## RwLock: Multiple Readers, One Writer

`RwLock<T>` allows multiple concurrent readers OR one
exclusive writer. Use when reads significantly outnumber
writes.

```rust
use std::sync::{Arc, RwLock};

let config = Arc::new(RwLock::new(Config::load()));

// Many readers in parallel
for _ in 0..10 {
    let config = Arc::clone(&config);
    thread::spawn(move || {
        let cfg = config.read().unwrap();  // Multiple readers OK
        use_config(&cfg);
    });
}

// Occasional writer (exclusive)
{
    let mut cfg = config.write().unwrap();
    cfg.reload();
}
```

### When RwLock Hurts

`RwLock` has higher overhead than `Mutex` due to tracking
reader count. It's slower when:

- Writes are frequent (>20% of operations)
- Lock hold time is very brief (overhead dominates)
- Single-threaded (use `RefCell` instead)

**Write starvation:** Standard `RwLock` may starve writers
if readers continuously hold the lock.
`parking_lot::RwLock` provides fair scheduling and
upgradeable read locks.

### Cached Computation Pattern

```rust
struct CachedComputation {
    input: RwLock<Vec<f64>>,
    result: RwLock<Option<f64>>,
}

impl CachedComputation {
    fn get_result(&self) -> f64 {
        // Fast path: read lock
        if let Some(result) = *self.result.read().unwrap() {
            return result;
        }
        // Slow path: compute and cache
        let input = self.input.read().unwrap();
        let computed = expensive_compute(&input);
        *self.result.write().unwrap() = Some(computed);
        computed
    }
}
```

---

## Decision Table

| Need | Type | Thread Safety | Runtime Cost |
|------|------|---------------|-------------|
| Mutate through `&self`, single thread | `Cell<T>` (Copy types) | No | Zero |
| Mutate through `&self`, single thread | `RefCell<T>` | No | Borrow tracking |
| Shared mutable state, single thread | `Rc<RefCell<T>>` | No | Ref count + borrow tracking |
| Exclusive mutable access, multi thread | `Mutex<T>` | Yes | Lock acquisition |
| Read-heavy shared access, multi thread | `RwLock<T>` | Yes | Reader count tracking |
| Shared mutable state, multi thread | `Arc<Mutex<T>>` | Yes | Atomic + lock |
| Read-heavy shared state, multi thread | `Arc<RwLock<T>>` | Yes | Atomic + reader tracking |

---

## Locks and Async: A Critical Pitfall

Never hold a `std::sync::Mutex` guard across an `.await`
point — the guard is not `Send`, and even if it compiles,
it can deadlock because the task may resume on a different
thread.

```rust
// Bad: lock held across await
let mut guard = data.lock().unwrap();
*guard = fetch_data().await;  // Task may switch threads here!

// Good: release lock before await
let current = {
    let guard = data.lock().unwrap();
    guard.clone()
};
let new_data = fetch_data().await;
*data.lock().unwrap() = new_data;
```

For async code that genuinely needs to hold a lock across
await:

- Use `tokio::sync::Mutex` (designed for async, but has
  higher overhead)
- Prefer restructuring to avoid holding locks across await
  entirely

See **rust-async** for more async-specific patterns.

**Clippy lints:**

- `clippy::await_holding_lock` — catches `MutexGuard`
  held across `.await`
- `clippy::await_holding_refcell_ref` — catches
  `Ref`/`RefMut` held across `.await`
