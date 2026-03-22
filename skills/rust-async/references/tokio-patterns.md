# Tokio Patterns

## Runtime Configuration

```rust
// Multi-threaded (default) — best for IO-bound servers
#[tokio::main]
async fn main() { ... }

// Current-thread — lighter, good for CLI tools or single-connection apps
#[tokio::main(flavor = "current_thread")]
async fn main() { ... }

// Custom configuration
#[tokio::main(worker_threads = 4)]
async fn main() { ... }

// Manual runtime for fine-grained control
let rt = tokio::runtime::Builder::new_multi_thread()
    .worker_threads(4)
    .max_blocking_threads(32)
    .enable_all()
    .build()?;

rt.block_on(async { ... });
```

**Choosing runtime flavor:**

- Server handling many connections → multi-threaded
- CLI tool, single task → current-thread
- Need multiple runtimes → manual builder

---

## spawn_blocking: CPU-Heavy Work

Blocking the async runtime thread starves other tasks.
Move CPU-intensive or blocking I/O work to the blocking
thread pool.

```rust
// CPU-intensive: JSON parsing, compression, hashing
let result = tokio::task::spawn_blocking(move || {
    serde_json::from_str::<Config>(&large_json)
}).await??;

// Blocking I/O that has no async equivalent
let hash = tokio::task::spawn_blocking(move || {
    let mut hasher = Sha256::new();
    hasher.update(&data);
    hasher.finalize()
}).await?;
```

For parallel CPU work inside `spawn_blocking`, use
Rayon:

```rust
let result = spawn_blocking(move || {
    items.par_iter().map(|item| process(item)).collect::<Vec<_>>()
}).await?;
```

---

## File I/O: tokio::fs

Use `tokio::fs` instead of `std::fs` in async code —
`std::fs` blocks the runtime thread.

```rust
use tokio::fs;
use tokio::io::{AsyncBufReadExt, BufReader};

// Reading
let content = fs::read_to_string("config.toml").await?;

// Writing
fs::write("output.txt", &data).await?;

// Line-by-line reading
let file = fs::File::open("large.log").await?;
let reader = BufReader::new(file);
let mut lines = reader.lines();
while let Some(line) = lines.next_line().await? {
    process(&line);
}
```

`std::fs` is acceptable during startup (before the runtime
is running) or in single-threaded runtimes where there's
nothing else to schedule.

---

## Concurrent Execution

### join! — Run All, Return All

```rust
let (users, orders, config) = tokio::join!(
    fetch_users(),
    fetch_orders(),
    load_config(),
);
// All three run concurrently — total time = slowest one
```

### try_join! — Fail Fast

```rust
let (users, orders) = tokio::try_join!(
    fetch_users(),
    fetch_orders(),
)?;
// Returns Err on first failure, drops the other future
```

### select! — Race, Handle First

```rust
tokio::select! {
    result = fetch_data() => handle_data(result),
    _ = tokio::time::sleep(Duration::from_secs(5)) => {
        println!("Timed out");
    }
    _ = shutdown_signal() => {
        println!("Shutting down");
    }
}
```

`select!` drops unfinished futures. Use `biased;` for
deterministic priority:

```rust
tokio::select! {
    biased;  // Check branches in declaration order (default: fair random to prevent starvation)
    _ = shutdown.cancelled() => break,
    msg = rx.recv() => handle(msg),
}
```

### Dynamic Collections: JoinSet

```rust
use tokio::task::JoinSet;

let mut set = JoinSet::new();

for url in urls {
    set.spawn(async move { fetch(url).await });
}

while let Some(result) = set.join_next().await {
    match result {
        Ok(Ok(data)) => process(data),
        Ok(Err(e)) => eprintln!("Task failed: {e}"),
        Err(e) => eprintln!("Task panicked: {e}"),
    }
}
// All tasks abort when JoinSet is dropped
```

Limit concurrency with a semaphore:

```rust
let semaphore = Arc::new(Semaphore::new(10));  // Max 10 concurrent

for url in urls {
    let permit = semaphore.clone().acquire_owned().await?;
    set.spawn(async move {
        let result = fetch(url).await;
        drop(permit);  // Release slot
        result
    });
}
```

---

## CancellationToken: Graceful Shutdown

```rust
use tokio_util::sync::CancellationToken;

let token = CancellationToken::new();

// Spawn workers with child tokens
for i in 0..4 {
    let child = token.child_token();
    tokio::spawn(async move {
        loop {
            tokio::select! {
                _ = child.cancelled() => {
                    println!("Worker {i} shutting down");
                    break;
                }
                _ = do_work() => {}
            }
        }
    });
}

// Signal shutdown on Ctrl+C
tokio::signal::ctrl_c().await?;
token.cancel();  // All child tokens cancelled
```

Child tokens form a hierarchy — cancelling a parent
cancels all children, but cancelling a child doesn't
affect siblings.
