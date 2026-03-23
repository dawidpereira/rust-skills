# Anti-Patterns Reference

Quick reference for code review. Each entry shows the
problem, a fix, and when the "bad" pattern is actually
acceptable.

---

## Unwrap Abuse

`.unwrap()` panics with no context, crashing the program.

```rust
// Bad
let content = std::fs::read_to_string("config.toml").unwrap();

// Good
let content = std::fs::read_to_string("config.toml")
    .context("failed to read config")?;
```

**When acceptable:** Tests, `const`/`static` init with
known-valid values (`Regex::new(r"^\d+$").unwrap()`),
after an explicit check with proof comment.
Alternatives: `?`, `unwrap_or`, `unwrap_or_default`,
`if let`, `match`. See rust-errors.

---

## Expect Without Context

`.expect("failed")` still panics. Use it to document
invariants, not to handle recoverable errors.

```rust
// Bad: expect on recoverable error
let file = File::open(path).expect("failed to open");

// Good: expect documents why this can't fail
let guard = mutex.lock().expect("mutex poisoned — unrecoverable");
```

**When acceptable:** Mutex poisoning, thread spawn
failure, compile-time-guaranteed values. The message
should explain *why this can't fail*. See rust-errors.

---

## Excessive Cloning

`.clone()` allocates and copies. Ask whether a reference
would work before reaching for it.

```rust
// Bad: clone to read
fn process(data: &String) { let local = data.clone(); }

// Good: just borrow
fn process(data: &str) { println!("{data}"); }
```

**When acceptable:** Storing owned data in collections,
sending across threads (`'static` required), `Copy`
types (free). For repeated cloning, consider `Arc` or
`clone_from()`. See rust-ownership.

---

## Lock Across Await

Holding a `MutexGuard` across `.await` blocks other
tasks. With `std::sync::Mutex`, it deadlocks the runtime.

```rust
// Bad: guard held across await
let mut data = mutex.lock().await;
let result = fetch_remote(&data).await; // blocks everyone
*data = result;

// Good: extract, drop lock, then await
let snapshot = { mutex.lock().await.clone() };
let result = fetch_remote(&snapshot).await;
*mutex.lock().await = result;
```

**When acceptable:** Never for std locks in async. For
tokio::sync::Mutex only if trivially short with no
awaits inside. See rust-async.

---

## &String / &Vec<T> Instead of Slices

`&String` forces callers to own a `String`. `&str`
accepts any string-like type via deref coercion. Same
for `&Vec<T>` → `&[T]`, `&PathBuf` → `&Path`.

```rust
// Bad
fn greet(name: &String) { println!("Hello, {name}"); }
fn sum(nums: &Vec<i32>) -> i32 { nums.iter().sum() }

// Good
fn greet(name: &str) { println!("Hello, {name}"); }
fn sum(nums: &[i32]) -> i32 { nums.iter().sum() }
```

**When acceptable:** Only if you need type-specific
methods like `capacity()`. See rust-api.

---

## Indexing Over Iteration

`v[i]` requires bounds checks, prevents SIMD
auto-vectorization, and risks off-by-one errors.

```rust
// Bad: bounds checked every access
for i in 0..data.len() { sum += data[i] * data[i]; }

// Good: no bounds checks, vectorizes
data.iter().map(|&x| (x as i64) * (x as i64)).sum::<i64>();
```

**When acceptable:** `enumerate()` with mutation,
`step_by()`, multi-dimensional matrix access. See
rust-perf.

---

## Panic for Expected Errors

`panic!` crashes the program. Use it for bugs, not for
conditions that can happen in normal operation.

```rust
// Bad: panics on network error
let resp = client.get(url).send()
    .unwrap_or_else(|_| panic!("request failed"));

// Good: return Result
let resp = client.get(url).send().context("HTTP request failed")?;
```

**When acceptable:** Invariant violations that indicate
a bug, test code, truly unrecoverable state. See
rust-errors.

---

## Empty Error Catch

`let _ = fallible()` or `.ok()` silently discards
errors. Failures go unnoticed and bugs hide.

```rust
// Bad: error vanishes
let _ = write_to_file(data);

// Good: log if non-critical
if let Err(e) = write_to_file(data) {
    warn!("write failed: {e}");
}
```

For batch operations, collect errors instead of
discarding or failing on the first:

```rust
let mut errors = Vec::new();
for item in items {
    if let Err(e) = process(&item) {
        errors.push((item.id, e));
    }
}
```

**When acceptable:** Documented best-effort cleanup
(`let _ = stream.shutdown(); // TCP close not
actionable`). Always comment why. See rust-errors.

---

## Over-Abstraction

Trait hierarchies and generics before concrete needs.
Costs compile time, binary size, and readability.

```rust
// Bad: generic for single use case
fn add<T, U, R>(a: T, b: U) -> R
where T: Into<R>, U: Into<R>, R: std::ops::Add<Output = R>
{ a.into() + b.into() }

// Good: concrete until proven otherwise
fn add(a: i32, b: i32) -> i32 { a + b }
```

**Rule of Three:** Wait for three concrete
implementations before extracting a trait.

**Signs:** Single-impl traits, type parameter soup,
marker traits with no methods, `where` clauses longer
than the function body.

**When acceptable:** Library public APIs, 2+ concrete
types sharing behavior, performance requiring static
dispatch. See rust-types.

---

## Premature Optimization

Optimizing without profiling. Intuition about
performance is often wrong.

```rust
// Bad: unsafe to skip bounds checks without measurement
let val = unsafe { *data.get_unchecked(i) };

// Good: profile first, optimize what matters
let val = data[i]; // leave it if profiling shows it's not hot
```

**When acceptable:** Algorithm choice (O(n) vs O(n^2)),
known-expensive operations in tight loops (allocations,
format!). Everything else: measure first. See rust-perf.

---

## Type Erasure Overuse

`Box<dyn Trait>` adds heap allocation and dynamic
dispatch. `impl Trait` is zero-cost when there's a
single concrete type.

```rust
// Bad: boxing when one type is returned
fn get_iter() -> Box<dyn Iterator<Item = i32>> {
    Box::new((0..10).map(|x| x * 2))
}

// Good: zero overhead
fn get_iter() -> impl Iterator<Item = i32> {
    (0..10).map(|x| x * 2)
}
```

**When acceptable:** Heterogeneous collections (mixed
runtime types), config-determined types, recursive
types, plugin systems. If all variants are known at
compile time, consider an enum instead. See rust-types.

---

## format!() in Hot Paths

`format!()` allocates a new `String` every call. In
hot loops, reuse a buffer with `write!()`.

```rust
// Bad: allocates per iteration
for i in 0..1000 { let s = format!("item-{i}"); process(&s); }

// Good: reuse buffer
let mut buf = String::with_capacity(32);
for i in 0..1000 {
    buf.clear();
    write!(&mut buf, "item-{i}").unwrap();
    process(&buf);
}
```

**When acceptable:** Cold paths — startup, error
messages, one-time construction. See rust-perf.

---

## Intermediate collect()

Each `.collect()` forces evaluation and allocates.
Chain lazily, collect once.

```rust
// Bad: three allocations, three passes
let a: Vec<_> = data.iter().filter(|x| **x > 0).collect();
let b: Vec<_> = a.iter().map(|x| x * 2).collect();

// Good: one allocation, one pass
let result: Vec<_> = data.iter()
    .filter(|x| **x > 0)
    .map(|x| x * 2)
    .take(10)
    .collect();
```

**When acceptable:** Need to iterate the result multiple
times, sort it, or access by random index. See
rust-perf.

---

## Stringly Typed

Using `String` where an enum or newtype would enforce
valid states at compile time.

```rust
// Bad: any string accepted
fn set_status(status: &str) { /* "active"? "actve"? */ }

// Good: compiler enforces valid states
enum Status { Active, Inactive, Pending }
fn set_status(status: Status) { }
```

**When acceptable:** At system boundaries where data
arrives as text (HTTP, config, CLI). Parse into types
at the boundary, use types internally. See rust-types.
