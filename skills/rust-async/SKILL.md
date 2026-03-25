---
name: rust-async
description: >
  Async Rust and concurrency with Tokio. Use when writing async code, choosing channel types
  (mpsc/broadcast/watch/oneshot), dealing with Send/Sync bounds, spawn_blocking, JoinSet,
  CancellationToken, or fixing issues with locks held across .await points. Also use for
  tokio::select!, graceful shutdown, and structured concurrency patterns.
---

# Async & Concurrency

## Core Question

**Is this I/O-bound (async) or CPU-bound (threads/spawn_blocking)?**

Async is for waiting on external things (network, disk,
timers). CPU-heavy work blocks the runtime — move it to
`spawn_blocking` or a dedicated thread pool.

---

## Error → Design Question

| Symptom              | Don't Just Say           | Ask Instead                                                 |
| -------------------- | ------------------------ | ----------------------------------------------------------- |
| Future is not `Send` | "Add Send bound"         | Why does data cross a thread boundary? Can you restructure? |
| Deadlock with Mutex  | "Use tokio::sync::Mutex" | Should you hold a lock across await at all?                 |
| Task hangs forever   | "Add timeout"            | Is there a cancellation path?                               |
| Channel fills up     | "Make it unbounded"      | What's the backpressure strategy?                           |

---

## Quick Decisions

| Situation                               | Reach For           | Why                                   |
| --------------------------------------- | ------------------- | ------------------------------------- |
| Run independent futures concurrently    | `tokio::join!`      | Runs all, returns all results         |
| Run fallible futures, fail fast         | `tokio::try_join!`  | Returns first error, drops rest       |
| Race futures, handle first completion   | `tokio::select!`    | Cancel losers automatically           |
| Dynamic number of spawned tasks         | `JoinSet`           | Add/remove tasks, collect results     |
| CPU-intensive work in async context     | `spawn_blocking`    | Moves to blocking thread pool         |
| File I/O in async code                  | `tokio::fs`         | Non-blocking file operations          |
| Graceful shutdown                       | `CancellationToken` | Hierarchical cancellation             |
| One producer, one consumer, one message | `oneshot`           | Request-response pattern              |
| Work queue (multiple producers)         | `mpsc` (bounded)    | Backpressure built in                 |
| All subscribers get all messages        | `broadcast`         | Pub/sub pattern                       |
| Share latest value, skip intermediate   | `watch`             | Config updates, state sharing         |
| Shared read-only data across tasks      | `Arc<T>`            | Clone Arc, not the data               |
| Shared mutable state across tasks       | `Arc<Mutex<T>>`     | Or `Arc<RwLock<T>>` if reads dominate |

---

## Channel Selection

| Channel     | Pattern               | Capacity               | Receivers          |
| ----------- | --------------------- | ---------------------- | ------------------ |
| `oneshot`   | Request → Response    | 1 message              | 1                  |
| `mpsc`      | Work queue            | Bounded (set capacity) | 1                  |
| `broadcast` | Pub/sub (all get all) | Bounded (ring buffer)  | N (all messages)   |
| `watch`     | Latest value          | 1 (latest only)        | N (skip to newest) |

**Always use bounded channels** unless you have a specific
reason not to. Unbounded channels grow without limit when
producer outpaces consumer.

Buffer sizing: start with `num_producers * 2` or expected
burst size. Monitor with `capacity()` and `len()`.

---

## The Lock-Across-Await Problem

Never hold a `std::sync::Mutex` guard across an `.await`:

```rust
// Bad: guard held across await — can deadlock
let mut guard = data.lock().unwrap();
*guard = fetch().await;

// Good: extract, await, then lock again
let current = data.lock().unwrap().clone();
let new_data = process(current).await;
*data.lock().unwrap() = new_data;
```

`tokio::sync::Mutex` is await-safe but has higher overhead.
Prefer restructuring to avoid holding locks across await
entirely.

---

## Usage Scenarios

**Scenario 1:** "I need to make 5 HTTP requests and combine the results"
→ Use `tokio::try_join!` for a fixed number, or
`JoinSet` for a dynamic number.
Don't await them sequentially — that's 5x slower.

**Scenario 2:** "My async task needs to do JSON parsing on large payloads"
→ JSON parsing is CPU-bound. Use
`spawn_blocking(move || serde_json::from_str(&data))`
to avoid blocking the runtime.

**Scenario 3:** "I need to shut down gracefully when Ctrl+C is pressed"
→ Create a `CancellationToken`, pass child tokens to tasks,
and use `tokio::select!` to race work against
`token.cancelled()`. On signal, cancel the root token.

---

## Reference Files

| File                                                         | Read When                                                                       |
| ------------------------------------------------------------ | ------------------------------------------------------------------------------- |
| [references/tokio-patterns.md](references/tokio-patterns.md) | Runtime setup, spawn_blocking, join/select patterns, JoinSet, cancellation      |
| [references/channels.md](references/channels.md)             | Choosing and using mpsc/broadcast/watch/oneshot, backpressure, message patterns |
| [references/safety.md](references/safety.md)                 | Lock safety across await, Send/Sync issues, clone-before-await patterns         |

---

## Cross-References

| When                                         | Check                            |
| -------------------------------------------- | -------------------------------- |
| Smart pointers for shared state (Arc, Mutex) | rust-ownership → Quick Decisions |
| Error handling in async (try_join, ?)        | rust-errors → Quick Decisions    |
| Async trait design and Send bounds           | rust-types → Quick Decisions     |
| Tokio runtime profile settings               | rust-perf → Quick Decisions      |
| Tracing spans in async code, .instrument()   | rust-tracing → Quick Decisions   |
