---
name: rust-architecture
description: >
  Rust project architecture patterns — vertical slice architecture for web APIs, component-based
  TUI structure for Ratatui apps, and cross-feature communication strategies. Use when organizing
  features into slices, wiring routes and services, choosing how features communicate, structuring
  a Ratatui terminal application, or deciding between single-crate and workspace layouts. Also use
  when setting up a new Axum/Actix-web project structure or adding a new feature to an existing
  sliced codebase.
---

# Architecture

## Core Question

**How should this project's modules, boundaries, and data flow
be organized so that adding a feature means changing one place?**

The right architecture makes features independently addable,
deletable, and testable. The wrong one scatters a single change
across the entire codebase.

---

## Quick Decisions

| Situation | Reach For | Why |
|-----------|-----------|-----|
| Web API with distinct features | Vertical Slice Architecture | Each feature is self-contained |
| Complex read vs write models | VSA + CQRS variant | Separates command and query paths within a slice |
| Large API, team-per-feature | VSA workspace layout | Compile-time boundaries, independent deps |
| Terminal UI with interactive views | TUI component-based (Ratatui) | Each component owns state, rendering, input |
| TUI app with business logic / data layers | TUI components + vertical slices | Feature slices own their components, services, and models |
| Feature needs data from another | Cross-slice communication | Pick the lightest strategy that works |
| Custom request validation | Axum extractor (`FromRequestParts`) | Compile-time enforced, reusable across handlers |
| Simple middleware (logging, auth) | `axum::middleware::from_fn` | Minimal boilerplate, async-friendly |
| Middleware ordering | outermost = first executed | CORS and tracing before auth, auth before handlers |
| Shared state (DB pool, config) | `State<AppState>` with `FromRef` | Compile-time checked, no runtime downcasting |
| Route-specific auth | `.route_layer(middleware::from_fn(require_auth))` | Protects only routes in that group |
| Request/response transformation | Tower `Layer` implementation | Full control over Service wrapping |
| Project has < 10 source files | Don't architect — flat structure | See rust-quality → Quick Decisions |

---

## Architecture Selection

Start with the application type:

- **Web API** → Vertical slices. Single crate for < 15 features,
  workspace for more. If reads and writes have very different shapes,
  add CQRS within the slice.
- **Terminal UI** → Component-based. Each screen or widget is a
  component with init/update/draw. If the app has data persistence
  or distinct feature domains, combine with vertical slices — each
  feature owns its components alongside service/model/repository.
- **Library** → Feature-based modules. See rust-quality for module
  organization patterns.
- **CLI tool** → Flat or feature-based depending on size. See
  rust-quality for thresholds.

Then consider domain complexity:

- **Simple CRUD** → Standard VSA slice (handler, service, model,
  repository, dto, routes, tests).
- **Complex domain logic** → VSA + CQRS commands/queries within
  each slice.
- **Rich domain model** → DDD building blocks within VSA slices.
  See rust-ddd skill for aggregates, value objects, events,
  repository separation.

---

## Key Rules

1. One feature = one directory. Never scatter a feature across layers.
2. Slices and components never import each other's internals.
3. Shared code lives in an explicit `shared/` module — infrastructure and config only.
4. New feature = new directory + register route or component. Nothing else changes.
5. Delete a feature = delete its directory. If other code breaks, the boundary leaked.
6. Handlers and components stay thin — delegate to services or state logic.
7. Each slice or component owns its tests.
8. Domain models are the source of truth; DTOs exist for the boundary.
9. Start with one crate. Split to workspace only when compile times or team size demand it.
10. When in doubt, duplicate between slices rather than coupling them with a shared abstraction.

---

## Usage Scenarios

**Scenario 1:** "I'm building a new web API with Axum"
→ Start with VSA single-crate layout. Create `features/` with one
directory per resource. Wire routes in `router.rs`, shared
infrastructure in `shared/`.

**Scenario 2:** "Feature A needs to read data from Feature B"
→ Check cross-slice communication strategies. For display: use a
read model (SQL JOIN). For business logic: use a public API trait.
For async reactions: domain events.

**Scenario 3:** "I'm building a terminal UI with Ratatui"
→ Use the component-based structure. Each component implements
the Component trait with init/handle_events/update/draw. If the
app has data layers or multiple feature domains, combine with
vertical slices — each feature owns its components + service + model.

**Scenario 4:** "My web API has 20+ features and builds are slow"
→ Graduate to VSA workspace layout. One crate per feature, shared
crate for infrastructure. Inherit dependencies and lints from
workspace root.

---

## Reference Index

| Reference | Read When |
|-----------|-----------|
| [references/vertical-slices.md](references/vertical-slices.md) | Setting up a web API: slice structure, file responsibilities, root wiring, CQRS variant, workspace layout, shared infrastructure, common dependencies |
| [references/cross-slice.md](references/cross-slice.md) | One feature needs data or behavior from another: 4 strategies (API traits, read models, shared types, domain events) with decision guide |
| [references/tui-components.md](references/tui-components.md) | Building a Ratatui TUI: component trait, file structure, app loop, state management |
| [references/web-middleware.md](references/web-middleware.md) | Axum extractors (ordering, custom, rejection), tower middleware (from_fn, Layer, ordering), State vs Extension, auth patterns, CORS/compression/rate-limiting |
| rust-ddd skill | Rich domain models within slices: aggregates, value objects, domain events, repository separation. Use when domain complexity warrants DDD building blocks |

---

## Cross-References

| When | Check |
|------|-------|
| Cargo.toml defaults, workspace setup, module organization basics | rust-quality → Quick Decisions |
| Async handler patterns, tokio runtime in web APIs | rust-async → Quick Decisions |
| Error type design for shared error module | rust-errors → Quick Decisions |
| Trait design for service boundaries and API traits | rust-types → Quick Decisions |
| Public API design within slices, naming conventions | rust-api → Quick Decisions |
| Rich domain model, aggregates, value objects, events | rust-ddd → Quick Decisions |
| Request tracing middleware, correlation IDs | rust-tracing → Quick Decisions |
| Rejection/error responses from extractors | rust-errors → Quick Decisions |
