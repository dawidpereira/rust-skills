# Linting Reference

## Lint Level Overview

| Level | Group | Purpose |
|-------|-------|---------|
| `deny` | `correctness` | Code that is outright wrong |
| `warn` | `suspicious` | Patterns that are almost always bugs |
| `warn` | `style` | Non-idiomatic code |
| `warn` | `complexity` | Unnecessarily complex code |
| `warn` | `perf` | Performance anti-patterns |
| selective | `pedantic` | Opinionated style checks |

## Deny: correctness

Hard errors for logic bugs, undefined behavior, and code that does not do what it appears
to do.

```toml
[lints.clippy]
correctness = "deny"
```

What it catches:

```rust
// Infinite iterator without take
for x in std::iter::repeat(1) { println!("{x}"); }

// NaN comparison (always false)
if x == f64::NAN { }

// Impossible condition
if x >= 0 && x < 0 { }

// Approximate constant instead of std::f64::consts::PI
let pi = 3.14;

// Invalid regex
let re = Regex::new("[");
```

## Warn: suspicious

Syntactically valid patterns that are almost always wrong.

```toml
[lints.clippy]
suspicious = "warn"
```

What it catches:

```rust
// Operator precedence trap
let bits = 1 << 4 + 1;

// Side effect in map (use for_each)
vec.iter().map(|x| { println!("{x}"); x }).collect::<Vec<_>>();

// Suspicious cast
let inverted = !x as i32;

// Almost-swapped comparison
if 5 < x && x < 3 { }
```

## Warn: style

Enforces idiomatic Rust patterns.

```toml
[lints.clippy]
style = "warn"
```

What it catches:

```rust
// Bad: len() == 0
if data.len() == 0 { }
// Good: is_empty()
if data.is_empty() { }

// Bad: redundant closure
iter.map(|x| foo(x))
// Good: direct reference
iter.map(foo)

// Bad: match for single pattern
match option {
    Some(x) => do_something(x),
    None => {},
}
// Good: if let
if let Some(x) = option { do_something(x) }

// Bad: explicit return at end
return Some(*first);
// Good: implicit return
Some(*first)
```

## Warn: complexity

Identifies code that can be simplified.

```toml
[lints.clippy]
complexity = "warn"
```

What it catches:

```rust
// Bad                                   // Good
match opt { Some(x) => Some(x+1),       opt.map(|x| x + 1)
            None => None }
if !(x == 0) { }                        if x != 0 { }
let y = x.clone(); // x is Copy         let y = x;
opt.and_then(|x| Some(x + 1))           opt.map(|x| x + 1)
```

## Warn: perf

Catches performance anti-patterns.

```toml
[lints.clippy]
perf = "warn"
```

What it catches:

```rust
// Bad                                       // Good
s.contains("x");                             s.contains('x');
for x in vec![1, 2, 3] { }                  for x in [1, 2, 3] { }
slice.iter().cloned().collect::<Vec<_>>();    slice.to_vec();
vec.extend(std::iter::once(item));           vec.push(item);
```

Notable perf lints:

| Lint | Fix |
|------|-----|
| `box_collection` | `Vec<T>` not `Box<Vec<T>>` |
| `large_enum_variant` | Box the large variant |
| `single_char_pattern` | Use `char` not `&str` |
| `unnecessary_to_owned` | Remove redundant `.to_owned()` |

## Selective: pedantic

Enable as a baseline and disable noisy lints:

```toml
[lints.clippy]
pedantic = "warn"
missing_errors_doc = "allow"
missing_panics_doc = "allow"
module_name_repetitions = "allow"
must_use_candidate = "allow"
too_many_lines = "allow"
similar_names = "allow"
struct_excessive_bools = "allow"
```

Useful pedantic lints to keep enabled:

| Lint | Why |
|------|-----|
| `doc_markdown` | Catch unmarked code in docs |
| `semicolon_if_nothing_returned` | Consistent semicolons |
| `unused_self` | Methods that should be functions |
| `wildcard_imports` | Avoid glob imports |

Alternative approach -- explicit opt-in without the group:

```toml
[lints.clippy]
doc_markdown = "warn"
semicolon_if_nothing_returned = "warn"
unused_self = "warn"
wildcard_imports = "warn"
```

## missing_docs

Warn on undocumented public items. For libraries, documentation is the user interface.

```toml
[lints.rust]
missing_docs = "warn"
```

Only applies to `pub` items. Suppress for internal modules:

```rust
#[allow(missing_docs)]
pub mod internal { /* ... */ }
```

## undocumented_unsafe_blocks

Require `// SAFETY:` comments on every unsafe block.

```toml
[lints.clippy]
undocumented_unsafe_blocks = "warn"
```

```rust
// Bad
unsafe { std::slice::from_raw_parts(ptr, len) }

// Good
// SAFETY: Caller guarantees ptr is valid for reads of len bytes,
// properly aligned, initialized, and no mutable references exist.
unsafe { std::slice::from_raw_parts(ptr, len) }
```

The SAFETY comment must explain what invariants are upheld, why they hold, and what
could go wrong if violated.

## cargo Lint Group

For published crates, enable `clippy::cargo` to check Cargo.toml metadata:

```toml
[lints.clippy]
cargo = "warn"
```

Catches missing `description`, `license`, `repository`, wildcard dependencies, and
multiple crate versions in the dependency tree. Disable for internal/unpublished crates.

## Workspace Lints

Define lints once at the workspace root and inherit in members:

```toml
# workspace Cargo.toml
[workspace.lints.clippy]
correctness = "deny"
suspicious = "warn"
style = "warn"
complexity = "warn"
perf = "warn"

[workspace.lints.rust]
missing_docs = "warn"

# member Cargo.toml
[lints]
workspace = true
```

## cargo fmt in CI

Run `cargo fmt --check` in CI to enforce consistent formatting.

```yaml
# GitHub Actions
- run: cargo fmt --all --check
```

Configure with `rustfmt.toml`:

```toml
edition = "2024"
max_width = 100
use_small_heuristics = "Max"
imports_granularity = "Module"
group_imports = "StdExternalCrate"
```

Use `#[rustfmt::skip]` for manually aligned code (matrices, lookup tables).

## Running Clippy

```bash
cargo clippy                          # Basic check
cargo clippy -- -D warnings           # Fail on any warning
cargo clippy --all-targets --all-features -- -D warnings  # Full check
```
