# Async Safety

## Locks Across .await Points

Holding a lock across an `.await` is one of the most
common async pitfalls.

### Why It's Dangerous

When you `.await`, the current task yields and may resume
on a different thread. If you hold a `std::sync::Mutex`
guard:

- The guard is `!Send` — the future becomes `!Send`,
  preventing `tokio::spawn`
- Even if it compiles (single-threaded runtime), another
  task trying to lock will deadlock

### The Fix: Extract, Await, Apply

```rust
// Bad: guard held across await
async fn update(data: &Mutex<State>) {
    let mut guard = data.lock().unwrap();
    guard.value = fetch_new_value().await;  // Deadlock risk!
}

// Good: scope the lock
async fn update(data: &Mutex<State>) {
    let current = {
        let guard = data.lock().unwrap();
        guard.value.clone()  // Clone what you need
    };  // Lock released here

    let new_value = process(current).await;

    data.lock().unwrap().value = new_value;  // Quick re-lock
}
```

### std::sync::Mutex vs tokio::sync::Mutex

|                 | `std::sync::Mutex`         | `tokio::sync::Mutex`        |
| --------------- | -------------------------- | --------------------------- |
| Across `.await` | No — deadlocks             | Yes — designed for it       |
| Performance     | Faster (no async overhead) | Slower (async coordination) |
| Use when        | Quick operations, no await | Must hold across await      |
| Poisoning       | Yes                        | No                          |

**Prefer `std::sync::Mutex`** with scoped locks (no await
inside) — it's faster. Only use `tokio::sync::Mutex` when
you genuinely need to hold across an await point, which
is rare.

### Alternative: Eliminate the Lock

Often the best fix is restructuring to avoid the lock
entirely:

```rust
// Instead of shared mutable state with locks:
let data = Arc::new(Mutex::new(State::default()));

// Use a dedicated task with channels:
let (tx, mut rx) = mpsc::channel(32);

// State owner task
tokio::spawn(async move {
    let mut state = State::default();
    while let Some(cmd) = rx.recv().await {
        match cmd {
            Command::Update(value) => state.value = value,
            Command::Get(reply) => { let _ = reply.send(state.clone()); }
        }
    }
});
```

---

## Send and Sync in Async

Futures spawned with `tokio::spawn` must be `Send`
because they may run on different threads. Common issues:

### Non-Send Types Across Await

```rust
// Bad: Rc is !Send
async fn process() {
    let data = Rc::new(vec![1, 2, 3]);
    some_async_op().await;  // Future is !Send because of Rc
    println!("{:?}", data);
}
// tokio::spawn(process()) — won't compile

// Good: use Arc instead
async fn process() {
    let data = Arc::new(vec![1, 2, 3]);
    some_async_op().await;
    println!("{:?}", data);
}
```

### MutexGuard Across Await

```rust
// Bad: MutexGuard is !Send (std) or blocks other tasks (tokio)
async fn bad(data: &Mutex<Vec<i32>>) {
    let guard = data.lock().unwrap();
    do_something().await;  // !Send
    drop(guard);
}

// Good: drop guard before await
async fn good(data: &Mutex<Vec<i32>>) {
    let snapshot = data.lock().unwrap().clone();
    do_something().await;
    // Use snapshot, not guard
}
```

---

## Clone Before Await

When an async block or closure needs data that lives
across an `.await`, clone or move it explicitly:

```rust
// Bad: reference can't live across await in spawned task
let config = &app.config;
tokio::spawn(async move {
    some_op().await;
    use_config(config);  // Borrow doesn't live long enough
});

// Good: clone Arc before spawning
let config = Arc::clone(&app.config);
tokio::spawn(async move {
    some_op().await;
    use_config(&config);  // Owned Arc, lives as long as needed
});
```

### Minimize What You Clone

Don't clone entire structs when you only need one field:

```rust
// Wasteful: cloning everything
let app = app.clone();
tokio::spawn(async move {
    process(&app.config.database_url).await;
});

// Better: clone only what's needed
let db_url = app.config.database_url.clone();
tokio::spawn(async move {
    process(&db_url).await;
});
```

---

## Clippy Lints for Async Safety

Enable these in your project:

```toml
[lints.clippy]
await_holding_lock = "deny"         # MutexGuard held across .await
await_holding_refcell_ref = "deny"  # Ref/RefMut held across .await
```

These catch the most dangerous async anti-patterns at compile time.
