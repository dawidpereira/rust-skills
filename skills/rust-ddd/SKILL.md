---
name: rust-ddd
description: >
  Rust domain-driven design — aggregates, entities, value objects, domain events,
  repository traits, and use case organization. Use when modeling a rich business
  domain, deciding where business rules live, choosing between service structs and
  aggregate methods, separating domain models from database types, or structuring
  features with DDD building blocks. Also use when working with bounded contexts,
  applying the repository pattern, or designing domain error types in Rust.
---

# Domain-Driven Design

## Core Question

**How should this business domain be modeled so that Rust's
type system and ownership enforce the rules at compile time?**

DDD in Rust is not a port of Java or C#. There is no DI
container, no mediator, no exception hierarchy. Rust's private
fields, `&mut self`, newtype pattern, and exhaustive `match`
give you compile-time enforcement that other languages achieve
only through runtime discipline.

---

## Quick Decisions

| Situation | Reach For | Why |
|-----------|-----------|-----|
| Wrapping a primitive with rules | Newtype value object | Validated at construction, zero-cost, type-safe |
| Identity matters, not just value equality | Entity with ID field | Compared by ID, not by all fields |
| Cluster of entities with invariants | Aggregate with private fields + command methods | `&mut self` = single writer, ownership = encapsulation |
| Something happened that other parts care about | Domain event enum | Decouples side effects, compiler forces exhaustive handling |
| Persisting/loading aggregates | Repository trait in feature, impl in infrastructure | Domain code stays framework-free |
| Orchestrating a use case | One file per use case (handler + DTOs) | Thin orchestration, no business logic in handler |
| Shared types across features (Money, IDs) | `shared/models.rs` | One source of truth, no feature coupling |
| Mapping domain errors to HTTP | `shared/errors.rs` with `From` impls | Domain stays framework-free |
| Feature-specific domain errors | `thiserror` enum per feature | Each feature owns its failure modes |
| Loading aggregate from DB | `reconstitute()` associated function | Bypasses validation for trusted data, no spurious events |
| Simple CRUD, no invariants | Standard VSA slice — skip DDD | DDD adds cost; use only when domain is rich |

---

## Key Principles

1. **Rust is not Java/C#.** No DI container (wire deps
   explicitly with `Arc<dyn Trait>`). No mediator (Axum
   extractors ARE DI, the Router IS the dispatcher). No
   exceptions (errors are `Result<T, E>` values).

2. **Ownership IS encapsulation.** Private fields on an
   aggregate mean outside code cannot break invariants.
   `&mut self` on command methods means one writer at a time.
   The compiler enforces what Java achieves with `synchronized`.

3. **Enums are sum types, not labels.** Rust enums are tagged
   unions — each variant can carry different data. They are the
   same construct as `Option` and `Result`: algebraic sum types
   with monadic combinators (`.map()`, `.and_then()`, `?`).
   `OrderStatus`, `OrderEvent`, `OrderError` — all enums. The
   compiler forces exhaustive handling via `match`. When state
   determines which operations are valid, consider making the
   aggregate itself an enum (typestate) so invalid transitions
   are compile errors, not runtime `Result`s. See
   references/building-blocks.md § "Enums Beyond Labels".

4. **Value objects are newtypes.** Never pass raw `String`,
   `i64`, or `Uuid` for domain concepts. `OrderId(Uuid)`,
   `Quantity(u32)`, `Money { amount, currency }` — validated
   at construction, zero-cost at runtime.

5. **Domain models are pure.** `models.rs`, `events.rs`,
   `error.rs`, `repository.rs` have zero imports from axum,
   sqlx, or any web/ORM framework. Infrastructure implements
   the traits.

6. **Aggregates collect events.** Command methods push events
   to an internal `Vec`. The use case handler calls
   `take_events()` after saving, then publishes them. The
   aggregate never knows about the event bus.

7. **`reconstitute()` loads from DB.** A `pub(crate)` associated
   function that builds an aggregate without validation or
   event generation. Only infrastructure code calls it.

8. **File-per-use-case replaces god services.** Each use case
   is a free async function in its own file with only the
   dependencies it needs. No 500-line service struct.

---

## Usage Scenarios

**Scenario 1:** "I'm modeling an order with line items, totals,
and status transitions"
→ Build an `Order` aggregate with private fields. Status
transitions are command methods (`place()`, `cancel()`) that
return `Result` and push domain events. Line items are entities
owned by the aggregate.

**Scenario 2:** "I have Money, Currency, Email used across
multiple features"
→ Put these in `shared/models.rs` as value objects with
validated constructors. Each feature imports them but owns its
own aggregates and business logic.

**Scenario 3:** "My service struct has 500 lines and 10
injected dependencies"
→ Split into file-per-use-case. Move business logic from
service methods to aggregate command methods. Each use case
handler takes only the 2-3 dependencies it needs.

**Scenario 4:** "I need to save an aggregate but domain code
shouldn't know about sqlx"
→ Define a repository trait in the feature (`repository.rs`).
Implement it in `infrastructure/persistence/` with separate
DB row structs. Map DB rows to domain objects via
`reconstitute()`.

**Scenario 5:** "My model.rs has 400 lines with aggregate,
entities, and value objects"
→ Promote to `models.rs` + `models/` folder. Split into
`models/order.rs`, `models/order_item.rs`,
`models/value_objects.rs`. Imports from outside the feature
don't change.

---

## Reference Index

| Reference | Read When |
|-----------|-----------|
| [references/building-blocks.md][bb] | Implementing aggregates, entities, value objects, domain events, domain errors — full code examples |
| [references/infrastructure.md][infra] | Repository traits and implementations, DB row mapping, use case files, error mapping, shared types, module wiring |
| [references/anti-patterns.md][anti] | Reviewing DDD code for common mistakes — anemic models, god services, leaking DB types |

[bb]: references/building-blocks.md
[infra]: references/infrastructure.md
[anti]: references/anti-patterns.md

---

## Cross-References

| When | Check |
|------|-------|
| Project folder structure, slice layout, module conventions | rust-architecture → Quick Decisions |
| Cross-feature communication strategies | rust-architecture → references/cross-slice.md |
| Newtype patterns, parse-don't-validate, typestate | rust-types → Quick Decisions |
| `thiserror` enum design, error propagation with `?` | rust-errors → Quick Decisions |
| Async repository traits, tokio patterns | rust-async → Quick Decisions |
| Public API design, naming conventions | rust-api → Quick Decisions |
