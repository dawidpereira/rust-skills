# Rust Skills

## Before Fixing a Compiler Error

Don't jump to the obvious fix. The compiler tells you
*what* broke — your job is to ask *why*.

- **Moved value?** Who should own this data? Is clone
  the answer, or is the ownership design wrong?
- **Dangling reference?** Is the scope boundary in the
  right place, or should the data live longer?
- **Trait bound not satisfied?** Is the trait design
  right, or is the abstraction fighting the problem?
- **Type mismatch?** Are the types modeling the domain
  correctly?

If you've tried the same fix twice and it keeps
cascading, stop patching and reconsider the design.

## Skill Index

| Skill | Use When |
|-------|----------|
| rust-ownership | Borrowing, lifetimes, smart pointers, move semantics, clone decisions |
| rust-errors | Result, Option, error types, propagation, panic vs return, diagnostic crates (miette, color-eyre) |
| rust-types | Generics, traits, dispatch, newtypes, typestate, PhantomData, closures, Fn traits, custom iterators |
| rust-async | Tokio, channels, Send/Sync, spawn, streams, Pin, threads vs async, rayon, atomics |
| rust-unsafe | Unsafe blocks, FFI, raw pointers, safety documentation |
| rust-serde | Serde derive, custom serializers, enum representations, serde attributes, zero-copy deserialization, DTO patterns |
| rust-macros | macro_rules!, proc macros, derive/attribute macros, fragment specifiers, repetition, hygiene, cargo expand |
| rust-api | API design, builder patterns, naming conventions, documentation |
| rust-perf | Memory optimization, compiler hints, profiling, benchmarking |
| rust-database | sqlx, compile-time queries, migrations, connection pooling, transactions, newtype mapping, database testing |
| rust-quality | Clippy lints, project structure, Cargo.toml defaults, proptest, mockall, criterion |
| rust-tests | Unit test strategy, integration test organization, test builders, error path testing, test isolation |
| rust-architecture | Project structure: vertical slices, TUI components, cross-feature communication, Axum middleware/extractors |
| rust-ddd | Domain-driven design: aggregates, entities, value objects, domain events, repository separation |
| rust-tracing | Tracing setup, structured logging, spans, #[instrument], RUST_LOG, middleware, OpenTelemetry |
| rust-ci-deploy | CI pipelines, GitHub Actions, caching, cross-compilation, Docker builds, release automation, cargo-dist |

## Cross-Reference Rule

When a skill cross-references another, read only the
referenced skill's **Quick Decisions** table. Do not
load its full reference files unless the user's question
specifically requires that depth.

## Agents

These agents fetch live data and require WebFetch or
MCP web access. Without these tools, answer from skill
knowledge and note when live data would improve the
answer.

| Need | Agent |
|------|-------|
| Rust release info | rust-changelog |
| Crate version/info | crate-researcher |
| API documentation | docs-researcher |
| Clippy lint details | clippy-researcher |
