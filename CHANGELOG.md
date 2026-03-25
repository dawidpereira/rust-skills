# Changelog

## 2026-03-25

### Added

- **rust-serde** — Serialization and deserialization with serde.
  Covers enum representation strategies (tagged, untagged), attribute
  quick reference, custom serializers for validated newtypes, zero-copy
  deserialization, feature-gating serde in libraries, and DTO mapping.
  References: attributes, enums, custom.

- **rust-database** — Database integration patterns with sqlx.
  Covers compile-time vs runtime queries, connection pooling, transaction
  scoping with auto-rollback, newtype-to-SQL type mapping, migration
  management, and database testing strategies.
  References: sqlx, transactions, testing.

- **rust-macros** — Declarative and procedural macro authoring.
  Covers macro_rules! patterns, fragment specifiers, repetition, hygiene,
  proc macro crate setup with syn/quote, derive/attribute/function-like
  macros, and the macro-vs-generic decision framework.
  References: declarative, procedural.

- **rust-ci-deploy** — CI/CD pipelines, Docker builds, and release
  automation. Covers GitHub Actions and GitLab CI templates, caching
  strategies, cross-compilation with musl, multi-stage Docker builds,
  cargo-dist for releases, and dependency auditing with cargo-deny.
  References: ci, docker, release.

### Expanded

- **rust-async** — Added streams (Stream trait, StreamExt, backpressure),
  Pin, async fn in traits, and non-async concurrency (std::thread, rayon,
  crossbeam, atomics with Ordering semantics).
  New references: streams, threads and parallelism.

- **rust-types** — Added closures (Fn/FnMut/FnOnce traits, capture
  semantics, returning closures) and custom iterators (Iterator,
  IntoIterator, FromIterator implementations).
  New reference: closures and iterators.

- **rust-architecture** — Added Axum middleware and extractor patterns
  (extractor ordering, custom extractors, tower middleware, from_fn,
  State vs Extension, auth patterns, CORS/compression/rate-limiting).
  New reference: web middleware.

- **rust-errors** — Added diagnostic error reporting (miette for rich
  diagnostics, color-eyre for colorized backtraces, user-facing vs
  developer-facing errors, CLI exit code conventions).

- **rust-tracing** — Structured observability with the `tracing` ecosystem.
  Covers subscriber setup, `#[instrument]` decoration, span lifecycle,
  `RUST_LOG` configuration, async-safe instrumentation, tower-http and
  sqlx middleware, sensitive data redaction, and OpenTelemetry export.
  References: setup, instrumentation, structured fields, production.

- **rust-tests** — Unit and integration test strategy for maintainable
  test suites. Covers test boundary selection, test builder helpers,
  error-path testing, parameterized tests, workspace test organization,
  and binary crate testing.
  References: unit tests, integration tests.

- **rust-ddd** — Domain-driven design modeled through Rust's type system.
  Aggregates own consistency via `&mut self`, value objects use newtypes,
  domain events are enums with exhaustive match, and repository traits
  live in the domain layer. Focuses on compile-time invariant enforcement
  rather than porting Java/C# patterns.
  References: building blocks, infrastructure, anti-patterns.

### Fixed

- Aligned examples and cross-references across all 12 skills to follow
  the cross-reference rule consistently.

## 2026-03-24

### Added

- **rust-architecture** — Project structure patterns for vertical slice
  architecture, component-based TUI layout with Ratatui, and cross-feature
  communication. Covers single-crate vs workspace decisions, Axum and
  Actix-web project setup, and when to split into modules vs crates.
  References: vertical slices, TUI components, cross-slice communication.

## 2026-03-23

### Added

- Expanded the anti-patterns reference in rust-quality with concrete
  examples and guidance for each anti-pattern.

### Fixed

- Corrected inaccuracies across skills and agent specifications.
- Resolved ambiguous reference link labels in rust-errors and rust-api
  that caused markdown rendering issues.

### Other

- Added MIT license and `.gitignore`.

## 2026-03-22

### Added

- **rust-ownership** — Borrowing, lifetimes, smart pointers, and move
  semantics. Maps compiler errors (E0382, E0597, E0506, E0507) to
  design questions about who should own data and for how long. Covers
  Arc, Rc, Box, Cell, RefCell, Mutex, and RwLock with clone-vs-redesign
  decision guidance.
  References: borrowing, interior mutability, smart pointers.

- **rust-errors** — Error handling with Result, Option, thiserror, and
  anyhow. Distinguishes library errors (typed, specific) from application
  errors (contextual, opaque). Covers the `?` operator, error chaining,
  and when panic is appropriate.
  References: decisions, patterns.

- **rust-types** — Type system design with generics, traits, and dispatch.
  Covers static vs dynamic dispatch trade-offs, sealed traits, the
  parse-don't-validate pattern, newtypes, typestate, and PhantomData for
  compile-time invariants.
  References: generics, trait design, type-driven development.

- **rust-unsafe** — Unsafe Rust and FFI with review checklists. Covers
  raw pointers, transmute, MaybeUninit, `#[repr]` layout control, and
  SAFETY comment documentation requirements. Every unsafe block must
  answer what invariants it upholds.
  References: checklist.

- **rust-api** — Public API design, naming conventions, and documentation
  standards. Covers builder patterns, `#[must_use]` and `#[non_exhaustive]`,
  the `as_`/`to_`/`into_` naming scheme, doc-tests, serde feature gating,
  and trait implementation expectations.
  References: documentation, naming, patterns.

- **rust-async** — Async concurrency with Tokio. Distinguishes I/O-bound
  (async) from CPU-bound (spawn_blocking) work. Covers channels (mpsc,
  broadcast, watch, oneshot), Send/Sync bounds, JoinSet, CancellationToken,
  `tokio::select!`, and graceful shutdown patterns.
  References: channels, safety, Tokio patterns.

- **rust-perf** — Performance optimization guided by measurement. Covers
  allocation strategies (SmallVec, `with_capacity`, arenas), release
  profile tuning (LTO, PGO, codegen-units), inline hints, criterion
  benchmarking, and flamegraph profiling.
  References: compiler, memory, runtime.

- **rust-quality** — Code quality, Clippy linting, and project structure.
  Covers lint groups and configuration, module organization, workspace
  layout, Cargo.toml defaults, and testing tools (proptest, mockall,
  criterion). Delegates test strategy to rust-tests.
  References: anti-patterns, linting, project setup, testing tools.

- **Research agents** — Four agents for fetching live Rust ecosystem data:
  **rust-changelog** fetches release notes by version from releases.rs,
  **crate-researcher** pulls crate metadata from lib.rs and crates.io,
  **docs-researcher** retrieves API documentation from docs.rs and
  doc.rust-lang.org, and **clippy-researcher** looks up lint details
  from the Clippy documentation site.

- **Project setup** — Initial repository with CLAUDE.md project
  instructions (skill index, cross-reference rule, compiler-error
  reasoning philosophy), README.md with installation guide, and
  `.markdownlint.json` configuration.
