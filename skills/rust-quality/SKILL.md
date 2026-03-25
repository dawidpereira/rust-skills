---
name: rust-quality
description: >
  Rust code quality — testing, linting, project structure, and anti-patterns. Use when setting
  up tests (proptest, mockall, criterion), configuring clippy lints, organizing modules and
  workspaces, or reviewing code for common Rust anti-patterns like excessive cloning or unwrap
  abuse. Also use when setting up a new Rust project's Cargo.toml with recommended lint and
  profile settings.
---

# Rust Quality

## Core Question

> Is this tested, linted, and organized for the next developer?

## Quick Decisions

| Situation | Action |
|-----------|--------|
| New project | Apply default Cargo.toml settings below |
| Adding a feature | Write tests first (unit + integration) |
| Reviewing code | Check anti-patterns index |
| Setting up CI | `cargo fmt --check && cargo clippy -- -D warnings && cargo test` |
| Organizing modules | Feature-based, flat for small projects |
| Benchmarking | Use criterion, never `Instant::now()` |
| Mocking dependencies | Extract traits, use mockall |
| Property testing | Use proptest for roundtrip/invariant checks |
| Workspace setup | Inherit lints and deps from workspace root |

## Default Cargo.toml Settings

Apply these settings to every new Rust project:

```toml
[package]
edition = "2024"
rust-version = "1.85"

[lints.clippy]
correctness = "deny"
suspicious = "warn"
style = "warn"
complexity = "warn"
perf = "warn"

[profile.release]
opt-level = 3
lto = "fat"
codegen-units = 1
panic = "abort"
strip = true

[profile.dev.package."*"]
opt-level = 3
```

For workspaces, define lints at the root and inherit:

```toml
# workspace Cargo.toml
[workspace.lints.clippy]
correctness = "deny"
suspicious = "warn"
style = "warn"
complexity = "warn"
perf = "warn"

# member Cargo.toml
[lints]
workspace = true
```

## Clippy Lint Levels

### Deny: correctness

Hard errors for code that is outright wrong. Catches infinite iterators, NaN comparisons,
impossible conditions, and invalid regex. Never allow these.

```toml
[lints.clippy]
correctness = "deny"
```

### Warn: suspicious, style, complexity, perf

Soft warnings for likely bugs, non-idiomatic patterns, unnecessary complexity, and
performance anti-patterns.

```toml
[lints.clippy]
suspicious = "warn"
style = "warn"
complexity = "warn"
perf = "warn"
```

### Selective: pedantic

Enable pedantic as a baseline, then disable noisy lints:

```toml
[lints.clippy]
pedantic = "warn"
missing_errors_doc = "allow"
missing_panics_doc = "allow"
module_name_repetitions = "allow"
must_use_candidate = "allow"
too_many_lines = "allow"
```

### Additional recommended lints

```toml
[lints.clippy]
undocumented_unsafe_blocks = "warn"

[lints.rust]
missing_docs = "warn"
```

For published crates, add `cargo = "warn"` to catch missing metadata and wildcard
dependencies.

## Project Structure Quick Reference

### Small projects (< 10 files): flat

```
src/
├── main.rs
├── lib.rs
├── config.rs
├── database.rs
└── error.rs
```

### Medium projects (10-20 files): feature-based modules

```
src/
├── main.rs          # Thin entry point
├── lib.rs           # Re-exports, module declarations
├── user/
│   ├── mod.rs
│   ├── model.rs
│   ├── repository.rs
│   └── service.rs
├── order/
│   ├── mod.rs
│   ├── model.rs
│   └── service.rs
└── shared/
    ├── mod.rs
    ├── error.rs
    └── database.rs
```

### Large projects: workspace

```
my-project/
├── Cargo.toml            # [workspace] with shared lints/deps
├── crates/
│   ├── core/
│   ├── api/
│   └── cli/
└── tests/
```

### Visibility rules

| Scope | Keyword | Use for |
|-------|---------|---------|
| Public API | `pub` | Types and functions users need |
| Crate-internal | `pub(crate)` | Shared implementation details |
| Parent-only | `pub(super)` | Sibling submodule helpers |
| Private | (default) | Everything else |

### Key patterns

- Keep `main.rs` thin, logic in `lib.rs` for testability.
- Organize by feature (user/, order/), not by type (models/, services/).
- Use `pub use` re-exports in `mod.rs` to create clean public APIs.
- Create a `prelude` module for commonly used types in libraries.
- Put multiple binaries in `src/bin/`.
- Use `mod.rs` for complex modules, adjacent files for simple ones.

## Usage Scenarios

### Scenario 1: Setting up a new Rust project

1. Apply default Cargo.toml settings (edition, lints, profiles).
2. Create `src/lib.rs` with module declarations and `src/main.rs` as thin entry point.
3. Add `rustfmt.toml` with `edition = "2024"` and `max_width = 100`.
4. Set up CI: `cargo fmt --check && cargo clippy -- -D warnings && cargo test`.
5. Structure tests: `#[cfg(test)] mod tests` in each file, `tests/` for integration.

### Scenario 2: Reviewing Rust code for quality

1. Check the anti-patterns index for common issues.
2. Verify all public items have documentation.
3. Confirm tests follow arrange/act/assert with descriptive names.
4. Look for `.unwrap()` in non-test code (use `?` or `.expect()` with context).
5. Ensure dependencies are behind traits for testability.
6. Run `cargo clippy -- -D warnings` and fix all warnings.

### Scenario 3: Adding tests to existing code

1. Unit tests: `#[cfg(test)] mod tests { use super::*; }` in the module file.
2. Integration tests: `tests/` directory, testing only the public API.
3. Property tests: `proptest!` for roundtrip, idempotence, and invariant properties.
4. Mocking: extract dependencies into traits, use `mockall` for mock generation.
5. Benchmarks: `criterion` in `benches/`, with `black_box` to prevent optimization.
6. Async tests: `#[tokio::test]` for async functions.

## Reference Index

| Reference | Covers |
|-----------|--------|
| [references/testing.md](references/testing.md) | Unit tests, integration tests, proptest, mockall, criterion, tokio::test, RAII fixtures, doctests |
| [references/linting.md](references/linting.md) | Clippy lint levels, pedantic config, workspace lints, missing_docs, unsafe docs, cargo fmt in CI |
| [references/project.md](references/project.md) | lib/main split, feature modules, visibility, re-exports, prelude, workspaces, dependency inheritance |
| [references/anti-patterns.md](references/anti-patterns.md) | 15 common anti-patterns with bad/good examples and "when acceptable" guidance |

## Cross-References

- **Error handling** patterns: see the `rust-errors` skill for `thiserror`, `anyhow`,
  `Result<T, E>`, and error context chains.
- **API design** and naming conventions: see the `rust-api` skill for builder pattern, newtype,
  `From`/`Into`, sealed traits, `#[non_exhaustive]`, and naming (references/naming.md).
- **Performance** optimization: see the `rust-perf` skill for profiling, memory layout,
  SIMD, and release profile tuning.
- **Async patterns**: see the `rust-async` skill for tokio runtime, channels, cancellation,
  and structured concurrency.
