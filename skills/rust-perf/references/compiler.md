# Compiler Optimization Reference

## Inline Hints

### `#[inline]` — Small Hot Functions

Suggests inlining. Primarily helps cross-crate calls (same-crate functions are already candidates). Use for small public functions in libraries.

```rust
#[inline]
pub fn len(&self) -> usize {
    self.inner.len()
}

#[inline]
pub fn is_ascii_digit(b: u8) -> bool {
    b >= b'0' && b <= b'9'
}
```

Generic functions across crate boundaries often need `#[inline]` to enable monomorphization at call sites.

### `#[inline(always)]` — Use Sparingly

Forces inlining regardless of compiler heuristics. Overuse bloats binary and hurts instruction cache. Reserve for proven hot paths with benchmark data.

```rust
// Only after profiling shows measurable benefit
impl Hasher for MyHasher {
    #[inline(always)]
    fn write(&mut self, bytes: &[u8]) {
        self.state = self.state.wrapping_add(bytes.len() as u64);
    }
}
```

Good candidates: tiny functions in hot inner loops, hash functions, SIMD helpers, iterator adapters. Verify with `cargo asm` or `cargo bloat`.

### `#[inline(never)]` — Cold Paths

Prevents inlining. Keeps cold code (error handling, logging, panics) out of the hot instruction stream.

```rust
fn process_data(data: &[u8]) -> Result<Output, Error> {
    if data.is_empty() {
        return Err(empty_data_error());
    }
    do_processing(data)
}

#[cold]
#[inline(never)]
fn empty_data_error() -> Error {
    Error::Empty {
        context: format!("Expected data, got empty slice"),
    }
}
```

---

## #[cold] — Unlikely Code Paths

Tells the compiler a function is rarely called. Effects:
- Code placed in separate section, away from hot code
- Branch prediction hints favor the non-cold path
- Cold functions not inlined into hot paths
- Compiler spends less optimization budget on cold code

```rust
#[cold]
fn handle_error<E: std::fmt::Display>(e: E) -> ! {
    eprintln!("Fatal error: {}", e);
    std::process::exit(1);
}

#[cold]
fn log_rare_event(event: &Event) {
    log::warn!("Rare event occurred: {:?}", event);
}
```

Almost always combine `#[cold]` with `#[inline(never)]` for maximum effect. Use for: error handlers, panic paths, fallback code, rare-event logging.

---

## Branch Hints

On stable Rust, use code structure to hint at likely branches:

```rust
// Early return = unlikely hint to compiler
fn process(data: Option<&Data>) -> Result<Output, Error> {
    let data = match data {
        None => return cold_none_error(),
        Some(d) => d,
    };
    fast_process(data)
}
```

Put the likely case in the `if` branch, unlikely in `else`. Put common match arms first. Extract unlikely paths into `#[cold]` functions.

On nightly: `std::intrinsics::likely()` / `unlikely()`. On stable with crate: `likely_stable::likely()`.

---

## LTO — Link-Time Optimization

Enables cross-crate inlining, dead code elimination, and devirtualization. Typical 5-20% performance improvement.

```toml
[profile.release]
lto = "fat"    # Maximum optimization, slowest compile
# lto = "thin" # Good balance for CI builds
```

| Setting | Compile Time | Performance |
|---------|--------------|-------------|
| `false` (default) | Fast | Baseline |
| `"thin"` | Medium | +5-15% |
| `"fat"` | Slow | +10-20% |

Use `"fat"` for production releases, `"thin"` for CI, `false` for development. Libraries should not set LTO — let downstream users choose.

---

## codegen-units = 1

Default is 16 parallel compilation units. Setting to 1 lets LLVM optimize across the entire crate: better inlining, dead code elimination, and constant propagation.

```toml
[profile.release]
codegen-units = 1  # 5-20% runtime improvement, 2-5x slower compile
```

Always combine with LTO for maximum effect. Use 16 (default) for dev builds, 1 for production releases.

---

## PGO — Profile-Guided Optimization

Uses real runtime behavior to guide optimization. 10-30% improvement beyond standard optimizations.

```bash
# 1. Build instrumented binary
RUSTFLAGS="-Cprofile-generate=/tmp/pgo-data" cargo build --release

# 2. Run representative workloads
./target/release/my_app < typical_workload.txt

# 3. Merge profile data
llvm-profdata merge -o /tmp/pgo-data/merged.profdata /tmp/pgo-data

# 4. Build optimized binary
RUSTFLAGS="-Cprofile-use=/tmp/pgo-data/merged.profdata" cargo build --release
```

Representative workloads are critical — profile with actual data patterns, not microbenchmarks. Combine with BOLT for additional 5-15% post-link optimization.

---

## target-cpu — Architecture-Specific Optimization

Default compiles for generic x86-64 baseline (SSE2 only). Modern CPUs have AVX2, AVX-512, and other instructions that go unused.

```toml
# .cargo/config.toml
[build]
rustflags = ["-C", "target-cpu=native"]
```

```bash
# Or via environment
RUSTFLAGS="-C target-cpu=native" cargo build --release

# Common targets
target-cpu=x86-64-v3  # AVX2, BMI2 (most modern x86)
target-cpu=apple-m1   # Apple Silicon
target-cpu=neoverse-n1 # AWS Graviton2
```

Use `native` for local/same-hardware deployments. Use specific targets for cross-compilation. For portable binaries, use runtime feature detection with `is_x86_feature_detected!`.

---

## Bounds Check Elimination

Iterators eliminate bounds checks that manual indexing requires. This enables vectorization and removes branch mispredictions.

```rust
// Bad: two bounds checks per iteration
fn sum_products(a: &[f64], b: &[f64]) -> f64 {
    let mut sum = 0.0;
    for i in 0..a.len() {
        sum += a[i] * b[i];
    }
    sum
}

// Good: no bounds checks, SIMD-friendly
fn sum_products(a: &[f64], b: &[f64]) -> f64 {
    a.iter().zip(b.iter()).map(|(x, y)| x * y).sum()
}
```

Other bounds-check-free patterns: `.windows(n)`, `.chunks_exact(n)`, `.split_at(mid)`, slice patterns (`let [a, b, c, rest @ ..] = data`).

Use `get_unchecked()` only when bounds are proven by prior assertions and profiling shows the check matters.

---

## Portable SIMD

Process multiple values per instruction (4x-8x speedup for suitable algorithms).

**Autovectorization (stable):** Use iterators, avoid early exits in loops, use `chunks_exact` for aligned access. LLVM often vectorizes automatically.

**Portable SIMD (nightly):**
```rust
#![feature(portable_simd)]
use std::simd::*;

fn sum_simd(data: &[f32]) -> f32 {
    let (prefix, middle, suffix) = data.as_simd::<8>();
    let mut simd_sum = f32x8::splat(0.0);
    for chunk in middle {
        simd_sum += *chunk;
    }
    prefix.iter().sum::<f32>() + simd_sum.reduce_sum() + suffix.iter().sum::<f32>()
}
```

**`wide` crate (stable):** Cross-platform SIMD types with medium control.

| Approach | Stability | Portability | Control |
|----------|-----------|-------------|---------|
| Autovectorization | Stable | Excellent | Low |
| `wide` crate | Stable | Good | Medium |
| Portable SIMD | Nightly | Excellent | High |
| Platform intrinsics | Stable | None | Maximum |

---

## Cache-Friendly Data Layouts

Cache misses cost ~100 cycles (L3 miss) vs ~4 cycles (L1 hit). Data layout determines cache efficiency.

### Struct of Arrays (SoA) over Array of Structs (AoS)

```rust
// Bad: AoS — loads 40 bytes per particle, uses only 24
struct Particle { position: [f32; 3], velocity: [f32; 3], mass: f32, id: u64 }
let particles: Vec<Particle> = ...;

// Good: SoA — contiguous access per field
struct Particles {
    positions_x: Vec<f32>,
    positions_y: Vec<f32>,
    velocities_x: Vec<f32>,
    velocities_y: Vec<f32>,
}
```

### Hot/Cold Splitting

Separate frequently accessed fields from rarely accessed ones into different structs/vecs.

### Avoid Pointer Chasing

Prefer contiguous arrays with index-based references over linked lists and Box chains. Sequential access keeps the prefetcher happy.

### Cache-Line Alignment

```rust
#[repr(C, align(64))]
struct PaddedCounter {
    value: AtomicU64,
    _pad: [u8; 56],
}
```

Use `repr(C, align(64))` to prevent false sharing in concurrent code. Measure with `perf stat -e cache-misses`.

---

## Complete Release Profile

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

# Custom profiles
[profile.release-fast-compile]
inherits = "release"
lto = false
codegen-units = 16

[profile.min-size]
inherits = "release"
opt-level = "z"
```
