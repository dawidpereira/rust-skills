# Threads and Parallelism

## When to Use What

| Work Type                  | Reach For               | Why                                       |
| -------------------------- | ----------------------- | ----------------------------------------- |
| CPU-bound batch processing | `rayon::par_iter`       | Automatic work-stealing across all cores  |
| CPU-bound in async context | `spawn_blocking` + rayon | Keeps the Tokio runtime unblocked         |
| I/O-bound concurrent work  | `tokio` tasks           | Thousands of tasks on few threads         |
| Short parallel burst       | `std::thread::scope`   | Borrows stack data, no Arc needed         |
| Lock-free flag or counter  | Atomics                 | Single-word sync without mutex overhead   |
| Complex parallel pipelines | `crossbeam` channels    | Bounded/unbounded with select support     |

**Key insight:** async is for waiting efficiently;
threads are for doing work in parallel. Mixing them
wrong — CPU work on the async runtime, or I/O polling
on a thread pool — wastes resources.

---

## std::thread Basics

```rust
use std::thread;

let handle = thread::spawn(move || {
    expensive_computation()
});

let result = handle.join().expect("thread panicked");
```

### std::thread::scope (Rust 1.63+)

Scoped threads can borrow from the parent stack — no
`Arc` or `'static` bound required:

```rust
let data = vec![1, 2, 3, 4, 5];
let mut results = vec![0; 5];

thread::scope(|s| {
    for (i, chunk) in data.chunks(2).enumerate() {
        let results = &mut results;
        s.spawn(move || {
            results[i] = chunk.iter().sum();
        });
    }
});
// All threads joined here — results is ready
```

**Use when:** you need a short burst of parallelism
with stack-local data, and don't want the overhead of
`Arc` or a thread pool.

---

## Rayon: Data Parallelism

Rayon provides parallel iterators that distribute work
across a global thread pool using work-stealing.

```toml
[dependencies]
rayon = "1"
```

### Parallel iterators

```rust
use rayon::prelude::*;

let sum: i64 = data.par_iter().map(|x| x * x).sum();

let processed: Vec<_> = items
    .par_iter()
    .filter(|item| item.is_valid())
    .map(|item| transform(item))
    .collect();
```

### Parallel sort and dedup

```rust
data.par_sort_unstable();
data.par_sort_unstable_by_key(|item| item.score);
```

### Parallel collect with into_par_iter

```rust
let results: Vec<Output> = inputs
    .into_par_iter()
    .map(|input| process(input))
    .collect();
```

### Custom thread pool

The global pool uses all cores by default. For
constrained environments:

```rust
let pool = rayon::ThreadPoolBuilder::new()
    .num_threads(4)
    .build()
    .unwrap();

pool.install(|| {
    data.par_iter().for_each(|item| process(item));
});
```

---

## Crossbeam

### Scoped threads (pre-1.63 alternative)

Since Rust 1.63 added `std::thread::scope`, you rarely
need `crossbeam::scope`. Use it only if you need
crossbeam's additional utilities (e.g., epoch-based
memory reclamation).

### Crossbeam channels

More capable than `std::sync::mpsc` — bounded, zero-
capacity (rendezvous), and `select!`:

```rust
use crossbeam_channel::{bounded, select, unbounded};

let (tx, rx) = bounded::<Task>(100);

// Producer
tx.send(task)?;

// Consumer with select
select! {
    recv(rx) -> msg => handle(msg?),
    recv(shutdown_rx) -> _ => break,
    default(Duration::from_secs(1)) => {
        println!("idle timeout");
    }
}
```

**Use crossbeam channels when:**
- You need `select!` across multiple channels in
  synchronous code
- You want a zero-capacity (rendezvous) channel
- You're in a non-async context

---

## Atomics

Lock-free, single-word synchronization primitives.
Faster than mutexes for simple counters and flags.

### Common atomic types

```rust
use std::sync::atomic::{
    AtomicBool, AtomicUsize, Ordering,
};

static RUNNING: AtomicBool = AtomicBool::new(true);
static COUNTER: AtomicUsize = AtomicUsize::new(0);

// Flag: signal shutdown
RUNNING.store(false, Ordering::Release);
if !RUNNING.load(Ordering::Acquire) {
    return;
}

// Counter: count events
COUNTER.fetch_add(1, Ordering::Relaxed);
let count = COUNTER.load(Ordering::Relaxed);
```

### Memory Ordering

| Ordering   | Use When                               | Cost     |
| ---------- | -------------------------------------- | -------- |
| `Relaxed`  | Independent counter, no ordering needs | Cheapest |
| `Acquire`  | Reading a flag/value set by Release    | Low      |
| `Release`  | Publishing a flag/value for Acquire    | Low      |
| `AcqRel`   | Read-modify-write (fetch_add, CAS)     | Medium   |
| `SeqCst`   | Need total ordering across all threads | Highest  |

**Practical guidance:**

- **Counters** (stats, metrics): `Relaxed` is fine — you
  don't need ordering guarantees for a counter.
- **Shutdown flag**: `Release` when setting, `Acquire`
  when reading — ensures memory written before the flag
  is visible after reading the flag.
- **Compare-and-swap loops**: `AcqRel` — you're reading
  and writing in one operation.
- **When in doubt**: `SeqCst` is always correct but
  slowest. Optimize to weaker orderings only when you
  understand why it's safe.

### AtomicPtr

```rust
use std::sync::atomic::{AtomicPtr, Ordering};

let data = Box::new(Config::default());
let ptr = AtomicPtr::new(Box::into_raw(data));

// Swap in new config
let new = Box::into_raw(Box::new(Config::updated()));
let old = ptr.swap(new, Ordering::AcqRel);

// Safety: we know old came from Box::into_raw
unsafe { drop(Box::from_raw(old)); }
```

**Prefer `Arc` over `AtomicPtr`** unless you need
lock-free data structures. `AtomicPtr` requires unsafe
and manual memory management.

---

## Combining Rayon with Tokio

Never run rayon work directly on the Tokio runtime —
it blocks async worker threads. Always bridge through
`spawn_blocking`:

```rust
use rayon::prelude::*;

async fn parallel_process(
    items: Vec<Item>,
) -> Result<Vec<Output>, Error> {
    tokio::task::spawn_blocking(move || {
        items
            .par_iter()
            .map(|item| transform(item))
            .collect()
    })
    .await
    .map_err(|e| Error::TaskJoin(e))
}
```

### Dedicated rayon pool to avoid contention

```rust
use once_cell::sync::Lazy;

static COMPUTE_POOL: Lazy<rayon::ThreadPool> = Lazy::new(|| {
    rayon::ThreadPoolBuilder::new()
        .num_threads(num_cpus::get())
        .thread_name(|i| format!("compute-{i}"))
        .build()
        .unwrap()
});

async fn heavy_compute(data: Vec<f64>) -> Vec<f64> {
    tokio::task::spawn_blocking(move || {
        COMPUTE_POOL.install(|| {
            data.par_iter()
                .map(|x| complex_math(*x))
                .collect()
        })
    })
    .await
    .unwrap()
}
```

---

## Decision Framework

### "Should I use async or threads?"

1. **Is the work I/O-bound?** (network, disk, timers)
   → Use async (Tokio). Thousands of concurrent tasks
   on a handful of threads.

2. **Is the work CPU-bound with large data?**
   → Use `rayon::par_iter`. Automatic work-stealing,
   scales to all cores.

3. **Is the work CPU-bound inside an async context?**
   → `spawn_blocking` wrapping rayon. Never block the
   Tokio runtime.

4. **Do I need a short parallel burst with local data?**
   → `std::thread::scope`. No Arc, no thread pool setup.

5. **Do I need a lock-free flag, counter, or pointer?**
   → Atomics. Cheaper than a Mutex for single-word data.

6. **Do I need complex synchronous channel patterns?**
   → `crossbeam` channels with `select!`.

### Anti-patterns

- **CPU work on Tokio runtime**: blocks other tasks,
  causes latency spikes. Use `spawn_blocking`.
- **Spawning threads per request**: thread creation is
  expensive. Use a pool (rayon, or Tokio's blocking pool).
- **`Arc<Mutex<Vec<_>>>` for parallel mutation**: use
  rayon's parallel collect instead — no locks needed.
- **`SeqCst` everywhere**: correct but slow. Use the
  weakest ordering that's safe for your pattern.
- **Unbounded crossbeam channels in hot loops**: same
  backpressure problem as unbounded async channels.
