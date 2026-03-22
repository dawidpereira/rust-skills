# Runtime Patterns Reference

## Iterators Over Indexing

Iterators enable bounds check elimination, SIMD auto-vectorization, and cleaner code. Manual indexing (`for i in 0..len`) prevents these optimizations.

```rust
// Bad: bounds checked every access
fn sum_squares(data: &[i32]) -> i64 {
    let mut sum = 0i64;
    for i in 0..data.len() {
        sum += (data[i] as i64) * (data[i] as i64);
    }
    sum
}

// Good: no bounds checks, vectorizes
fn sum_squares(data: &[i32]) -> i64 {
    data.iter().map(|&x| (x as i64) * (x as i64)).sum()
}
```

Use `.zip()` for parallel iteration, `.enumerate()` when index is needed, `.iter_mut()` for mutation.

---

## Keep Iterators Lazy

Rust iterators compute on demand. Each `.collect()` forces evaluation and allocates. Chain operations lazily and collect once at the end.

```rust
// Bad: three allocations, three passes
fn process(data: Vec<i32>) -> Vec<i32> {
    let step1: Vec<_> = data.into_iter().filter(|x| *x > 0).collect();
    let step2: Vec<_> = step1.into_iter().map(|x| x * 2).collect();
    step2.into_iter().take(10).collect()
}

// Good: one allocation, one pass
fn process(data: Vec<i32>) -> Vec<i32> {
    data.into_iter()
        .filter(|x| *x > 0)
        .map(|x| x * 2)
        .take(10)
        .collect()
}
```

Lazy methods (return iterators): `.filter()`, `.map()`, `.take()`, `.skip()`, `.zip()`, `.chain()`, `.flat_map()`, `.enumerate()`.

Consuming methods (evaluate immediately): `.collect()`, `.for_each()`, `.count()`, `.sum()`, `.fold()`, `.any()`, `.all()`, `.find()`.

---

## Don't Collect Intermediate Iterators

Each `.collect()` allocates a new collection. Avoid collecting just to iterate again.

```rust
// Bad: collect to check existence
fn has_valid(items: &[Item]) -> bool {
    let valid: Vec<_> = items.iter().filter(|i| i.is_valid()).collect();
    !valid.is_empty()
}

// Good: short-circuits on first match
fn has_valid(items: &[Item]) -> bool {
    items.iter().any(|i| i.is_valid())
}
```

| Instead of Collecting to... | Use |
|-----------------------------|-----|
| Check if any match | `.any(predicate)` |
| Count elements | `.count()` |
| Sum values | `.sum()` |
| Find first match | `.find(predicate)` |
| Check if all match | `.all(predicate)` |

Collect is needed when: iterating multiple times, sorting, random access, or transferring ownership.

Return `impl Iterator<Item = T>` from functions to let callers decide whether to collect.

---

## Entry API for Maps

Single lookup for insert-or-update. Without it, `contains_key()` + `insert()` hashes twice.

```rust
// Bad: two lookups, two hash computations
if map.contains_key(&key) {
    *map.get_mut(&key).unwrap() += 1;
} else {
    map.insert(key, 1);
}

// Good: one lookup
*map.entry(key).or_insert(0) += 1;
```

### Entry API Methods

| Method | Behavior |
|--------|----------|
| `.or_insert(val)` | Insert val if empty |
| `.or_insert_with(f)` | Insert f() if empty (lazy) |
| `.or_default()` | Insert Default::default() if empty |
| `.and_modify(f)` | Apply f if occupied |

### Common Patterns

```rust
// Word count
for word in text.split_whitespace() {
    *counts.entry(word).or_insert(0) += 1;
}

// Group by
for item in items {
    groups.entry(item.category.clone()).or_default().push(item);
}

// Update or insert with different logic
map.entry(key)
    .and_modify(|v| v.count += 1)
    .or_insert(Value { count: 1 });
```

---

## drain() — Remove and Reuse Allocation

`drain()` removes elements while keeping the collection's allocated capacity.

```rust
// Transfer elements between collections
fn transfer_all(src: &mut Vec<Item>, dst: &mut Vec<Item>) {
    dst.extend(src.drain(..));
}

// Batch processing with reused buffers
let mut batch = Vec::with_capacity(100);
while !data.is_empty() {
    batch.extend(data.drain(..100.min(data.len())));
    process_batch(&batch);
    batch.clear();
}
```

| Operation | Elements | Capacity | Returns |
|-----------|----------|----------|---------|
| `.clear()` | Removed | Kept | Nothing |
| `.drain(..)` | Removed | Kept | Iterator of owned elements |
| `std::mem::take()` | Moved out | Reset to 0 | Owned collection |

---

## extend() — Batch Insertions

`extend()` pre-allocates capacity from size hints and inserts in bulk. Individual `push()` calls may trigger multiple reallocations.

```rust
// Bad: may reallocate many times
for item in other_vec {
    results.push(item);
}

// Good: single capacity check, bulk insert
results.extend(other_vec);

// Combine collections
a.extend(b);           // Move from Vec
a.extend_from_slice(&b); // Copy from slice (Copy types)
a.append(&mut b);      // Move all, b becomes empty
```

For strings: `parts.concat()` or `parts.join("")` instead of loop with `push_str`.

---

## Avoid chain() in Hot Loops

`chain()` adds a branch check on every `.next()` call (which iterator is active?). In tight hot loops, use separate loops instead.

```rust
// Bad: branch per element in hot path
for x in a.iter().chain(b.iter()) {
    sum += *x as i64;
}

// Good: branch-free inner loops
for x in a { sum += *x as i64; }
for x in b { sum += *x as i64; }
```

`chain()` is fine for one-time iteration, lazy evaluation with short-circuit, or small collections outside hot paths.

---

## collect_into() — Reuse Containers

`collect_into()` (stable since Rust 1.83) collects into an existing collection, reusing its allocation.

```rust
let mut buffer = Vec::new();
for batch in data {
    buffer.clear();
    batch.iter()
        .filter(|&&x| x > 0)
        .copied()
        .collect_into(&mut buffer);
    process(&buffer);
}
```

Pre-1.83 alternative: `buffer.extend(iter)` after `buffer.clear()`. Works with any collection implementing `Extend`.

---

## black_box() in Benchmarks

The compiler eliminates computations whose results are unused. `black_box()` prevents this in benchmarks.

```rust
use criterion::{black_box, criterion_group, criterion_main, Criterion};

fn bench(c: &mut Criterion) {
    c.bench_function("compute", |b| {
        b.iter(|| {
            // black_box input: prevents constant folding
            // black_box output: prevents dead code elimination
            black_box(expensive_computation(black_box(42)))
        })
    });
}
```

`std::hint::black_box` is stable since Rust 1.66. Always black_box both inputs (prevent constant folding) and outputs (prevent elimination).

---

## Release Profile Settings

Default release profile leaves performance on the table. Tune for production.

```toml
[profile.release]
opt-level = 3          # Maximum optimization
lto = "fat"            # Cross-crate optimization (+10-20%)
codegen-units = 1      # Single compilation unit (+5-20%)
panic = "abort"        # No unwind tables, smaller binary
strip = true           # Remove symbols

[profile.bench]
inherits = "release"
debug = true           # Keep symbols for flamegraphs
strip = false
```

| Option | Default | Optimized | Effect |
|--------|---------|-----------|--------|
| `opt-level` | 3 | 3 | Max optimization |
| `lto` | false | "fat" | Cross-crate optimization |
| `codegen-units` | 16 | 1 | Better whole-crate optimization |
| `panic` | "unwind" | "abort" | Smaller binary |

For size optimization: `opt-level = "z"`, `lto = "fat"`, `codegen-units = 1`, `strip = true`.

---

## Profile Before Optimizing

Intuition about performance is wrong more often than right. The optimization workflow:

1. Write correct, idiomatic code
2. Benchmark suspected hot paths with criterion
3. Profile under realistic load (`cargo flamegraph`)
4. Identify actual bottlenecks (top 10% of time)
5. Optimize ONE thing
6. Measure improvement
7. Repeat if needed

### What Flamegraphs Reveal

- **Wide bars** = time hogs (optimize these)
- **malloc/free** = allocation-heavy code (reuse buffers, use with_capacity)
- **memcpy** = unnecessary copying (use references, Cow, zero-copy)
- **Bounds checks** = indexing in tight loops (switch to iterators)

### Common Premature Optimizations to Avoid

| Premature | Reality |
|-----------|---------|
| `#[inline(always)]` everywhere | Compiler usually knows better |
| `unsafe` for bounds checks | Iterators do this safely |
| Custom allocator | Default allocator is fast enough |
| Object pooling for small objects | Allocator handles this well |
| Hand-rolled SIMD | Auto-vectorization often works |

---

## Don't Use format!() in Hot Paths

`format!()` allocates a new String every call. In loops, this creates allocation churn.

```rust
use std::fmt::Write;

// Bad: allocates per iteration
for event in events {
    let msg = format!("[{}] {}", event.level, event.message);
    logger.log(&msg);
}

// Good: reuses buffer
let mut buf = String::with_capacity(256);
for event in events {
    buf.clear();
    write!(buf, "[{}] {}", event.level, event.message).unwrap();
    logger.log(&buf);
}
```

`format!()` is fine for: startup messages, error paths, one-time operations, debug output.
