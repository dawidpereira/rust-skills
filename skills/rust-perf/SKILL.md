---
name: rust-perf
description: >
  Rust performance optimization — memory, compiler hints, and profiling. Use when optimizing
  allocations (SmallVec, with_capacity, arena), configuring release profiles (LTO, PGO,
  codegen-units), adding inline hints, benchmarking with criterion, or profiling with
  flamegraph. Also use when reviewing code for unnecessary allocations, premature optimization,
  or format! in hot paths.
---

# Performance Optimization

## Core Question

**Have you measured first, or are you guessing?**

Intuition about performance is often wrong. The code you
think is slow frequently isn't. Profile first, then optimize
the actual bottleneck with data-driven decisions.

---

## Quick Decisions

| Situation | Reach For | Why |
|-----------|-----------|-----|
| Collection size known upfront | `Vec::with_capacity(n)` | Avoids ~10 reallocations for 1000 elements |
| Usually-small collection (2-8 items) | `SmallVec<[T; N]>` | Stack-allocated for common case, heap fallback |
| Hard upper bound, no heap allowed | `ArrayVec<T, N>` | Guaranteed stack-only, panics on overflow |
| Fixed-size heap data, never grows | `Box<[T]>` / `Box<str>` | Saves 8 bytes per instance vs Vec/String |
| Often-empty vectors in many instances | `ThinVec<T>` | 8 bytes empty vs 24 bytes for Vec |
| One enum variant much larger than others | `Box` the large variant | Clippy `large_enum_variant` catches this |
| Millions of short strings (< 24 chars) | `CompactString` | Inline storage, zero heap allocation |
| Repeatedly cloning into same variable | `target.clone_from(&source)` | Reuses existing allocation |
| Temporary collection in a loop | `.clear()` + reuse | Keeps capacity across iterations |
| `format!()` in a hot loop | `write!(&mut buf, ...)` | Zero allocation with reused buffer |
| Many small allocations (AST, parsing) | `bumpalo::Bump` arena | Bump-pointer allocation, bulk free |
| Parsing without ownership needs | `&str` / `&[u8]` slices | Zero-copy, no allocation |
| Map insert-or-update | `.entry().or_insert()` | Single lookup instead of two |
| Iterating with manual indexing | `.iter()` / `.zip()` | Eliminates bounds checks, enables SIMD |
| Intermediate `.collect()` calls | Chain iterators lazily | One allocation, one pass |
| Release build performance | `lto = "fat"`, `codegen-units = 1` | 10-20% improvement typical |
| Proven hot inner loop function | `#[inline]` or `#[inline(always)]` | Cross-crate inlining hint |
| Error construction path | `#[cold]` + `#[inline(never)]` | Keeps cold code out of hot path |
| Need to benchmark correctly | `black_box()` inputs and outputs | Prevents dead code elimination |

---

## Optimization Priority

Optimize in this order — each level has roughly 10x less impact than the one above:

1. **Algorithm & data structure** — O(n) vs O(n^2) dwarfs everything else
2. **Data layout** — SoA vs AoS, cache-friendly access, avoid pointer chasing
3. **Allocations** — with_capacity, reuse buffers, arena allocators
4. **Compiler hints** — LTO, PGO, codegen-units, inline hints
5. **SIMD & low-level** — portable SIMD, bounds check elimination, target-cpu

---

## Profile BEFORE Optimizing

### Tools

| Tool | What It Shows | When to Use |
|------|---------------|-------------|
| `cargo flamegraph` | CPU time by call stack | First step — find where time goes |
| `cargo instruments -t time` (macOS) | CPU time profiling | macOS alternative to perf |
| `criterion` | Micro-benchmark with statistics | Compare before/after for specific functions |
| `DHAT` / `heaptrack` | Heap allocation sites and counts | When allocation pressure is suspected |
| `perf stat -e cache-misses` | Cache efficiency | When data layout matters |
| `cargo bloat` | Binary size by function/crate | When binary size is a concern |

### Workflow

```text
1. Write correct code first
2. Write benchmarks for suspected hot paths
3. Profile under realistic load
4. Identify actual bottlenecks (top 10% of time)
5. Optimize ONE thing
6. Measure improvement with same benchmark
7. Repeat if needed
```

### Criterion Quick Setup

```rust
use criterion::{black_box, criterion_group, criterion_main, Criterion};

fn bench_hot_function(c: &mut Criterion) {
    let data = generate_test_data(1000);
    c.bench_function("hot_function", |b| {
        b.iter(|| hot_function(black_box(&data)))
    });
}

criterion_group!(benches, bench_hot_function);
criterion_main!(benches);
```

---

## Anti-Patterns to Watch For

| Anti-Pattern | Fix |
|--------------|-----|
| `format!()` in hot loop | `write!(&mut buffer, ...)` with reused buffer |
| Intermediate `.collect()` between iterator steps | Chain lazily, collect once at end |
| Optimizing without profiling data | Run `cargo flamegraph` first |
| `#[inline(always)]` everywhere | Let compiler decide; use `#[inline]` for cross-crate |
| `unsafe` to skip bounds checks | Use iterators — they eliminate bounds checks safely |
| `contains_key()` then `insert()` | Use `.entry()` API for single lookup |

---

## Usage Scenarios

**Scenario 1:** "My web handler is slow and I'm not
sure why"
-> Run `cargo flamegraph` on a representative workload.
Look for wide bars (time hogs), `malloc`/`free`
(allocation heavy), `memcpy` (unnecessary copies).
Optimize only what the flamegraph shows as hot.

**Scenario 2:** "I have a Vec that I fill in a loop and
it's showing up in profiling"
-> Check if size is known: use `with_capacity()`. If the
Vec is reused across iterations: `.clear()` instead of
creating new. If always small (< 8 items): consider
`SmallVec`. If fixed after creation: convert to
`Box<[T]>`.

**Scenario 3:** "I need maximum throughput for a release
binary"
-> Set release profile: `lto = "fat"`,
`codegen-units = 1`, `panic = "abort"`,
`strip = true`. For deployment on known hardware:
`RUSTFLAGS="-C target-cpu=native"`. For maximum gains:
use PGO with representative workloads.

---

## Release Profile

```toml
[profile.release]
opt-level = 3
lto = "fat"
codegen-units = 1
panic = "abort"
strip = true

[profile.bench]
inherits = "release"
debug = true
strip = false

[profile.dev.package."*"]
opt-level = 3
```

---

## Reference Files

| File | Read When |
|------|-----------|
| [references/memory.md](references/memory.md) | Allocation strategies: with_capacity, SmallVec, arena, zero-copy, compact strings, type size assertions |
| [references/compiler.md](references/compiler.md) | Compiler hints: inline, #[cold], LTO, PGO, codegen-units, target-cpu, SIMD, cache layout |
| [references/runtime.md](references/runtime.md) | Runtime patterns: iterators vs indexing, lazy chains, entry API, drain/extend, collect patterns, benchmarking |

---

## Cross-References

| When | Check |
|------|-------|
| Clone vs borrow decision for performance | rust-ownership -> Quick Decisions |
| Error construction on cold paths | rust-errors -> Quick Decisions |
| Async runtime and spawn_blocking for CPU work | rust-async -> Quick Decisions |
| API design that avoids unnecessary allocations | rust-api -> Quick Decisions |
| Clippy perf lints and lint configuration | rust-quality -> Quick Decisions |
