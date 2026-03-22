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
| rust-errors | Result, Option, error types, propagation, panic vs return |
| rust-types | Generics, traits, dispatch, newtypes, typestate, PhantomData |
| rust-async | Tokio, channels, Send/Sync, spawn, concurrency patterns |
| rust-unsafe | Unsafe blocks, FFI, raw pointers, safety documentation |
| rust-api | API design, builder patterns, naming conventions, documentation |
| rust-perf | Memory optimization, compiler hints, profiling, benchmarking |
| rust-quality | Testing, clippy lints, project structure, Cargo.toml defaults |

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
