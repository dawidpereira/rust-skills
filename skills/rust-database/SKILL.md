---
name: rust-database
description: >
  Rust database access with sqlx — compile-time checked
  queries, migrations, connection pooling, transactions,
  type mapping for domain newtypes, and database testing.
  Use when choosing between sqlx, diesel, and sea-orm,
  writing queries with compile-time safety, setting up
  migrations, configuring connection pools, mapping Rust
  newtypes to SQL types, scoping transactions, testing
  repository layers against real databases, or diagnosing
  pool timeouts and migration failures.
---

# Database Access

## Core Question

**How does this data move between Rust's type system and
the database, preserving correctness at both boundaries?**

Compile-time query checking catches schema drift before
it reaches production. Newtype wrappers carry domain
meaning through the SQL boundary. Transactions scope
the unit of change. If your types lie about what the
database holds, no amount of testing will save you.

---

## Error → Design Question

| Symptom | Ask Instead |
| --- | --- |
| "no rows returned" | Should this be `Option` or an error? |
| "type mismatch" | Are your Rust types aligned with the schema? |
| "pool timed out" | Is the pool sized for this workload? |
| "migration failed" | Is the migration reversible? |
| "column not found" | Are compile-time queries in sync with the schema? |
| Transaction deadlock | Is the transaction scope too broad? |

---

## Quick Decisions

| Situation | Reach For | Why |
| --- | --- | --- |
| New project choosing DB library | `sqlx` | Compile-time safety, no ORM overhead, async-native |
| Need ORM-like features | `sea-orm` | Built on sqlx, ActiveRecord-style, code-gen migrations |
| Legacy schema, complex joins | `diesel` | Strong type-safe query builder, sync-first |
| Compile-time checked queries | `sqlx::query!()` with `DATABASE_URL` | Schema mismatches fail at compile time |
| Runtime dynamic queries | `sqlx::query()` with `.bind()` | Flexible WHERE clauses, user-driven filters |
| Mapping rows to structs | `query_as!()` or `FromRow` derive | Type-safe row mapping without manual extraction |
| Newtype in DB column | Implement `Type` + `Encode` + `Decode` | Domain types pass through SQL boundary cleanly |
| Schema migrations | `sqlx-cli` with reversible migrations | Version-controlled, repeatable, rollback support |
| Connection pool sizing | CPU cores × 2 + disk spindles (start 5–10) | Avoid starving the pool or overwhelming the DB |
| Transactions | `pool.begin()`, commit on success | Auto-rollback on drop prevents partial writes |
| Nested transactions | Savepoints via `Transaction::begin()` | Partial rollback without aborting outer transaction |
| Testing with real DB | Per-test database or transaction rollback | Isolation without mocks, real query execution |
| N+1 query problem | Batch with `WHERE IN` or `JOIN` | Single round-trip instead of N |
| Read replicas | Separate pool, route reads explicitly | Keep write pool available, scale reads independently |

---

## sqlx Query Styles

Four styles serve different needs:

- **`query!()`** — compile-time checked against a live
  database or cached `sqlx-data.json`. Use for all
  static queries. Catches typos and type mismatches
  before runtime.
- **`query_as!()`** — same compile-time checking but
  maps results directly to a named struct. Cleaner
  than manual field extraction.
- **`query()`** — runtime-only, no compile-time
  checking. Use when the query shape depends on user
  input or conditional logic.
- **`QueryBuilder`** — builds dynamic queries
  programmatically. Use for bulk inserts, dynamic
  WHERE clauses, or conditional JOINs.

See `references/sqlx.md` for examples of each style.

---

## The Newtype-to-SQL Bridge

Domain newtypes (`UserId(Uuid)`, `Email(String)`) must
cross the SQL boundary. Implement `sqlx::Type`,
`sqlx::Encode`, and `sqlx::Decode` — or derive them
with `#[derive(sqlx::Type)]` for simple wrappers.

The `#[sqlx(transparent)]` attribute delegates to the
inner type, keeping the domain wrapper invisible to SQL:

```rust
#[derive(sqlx::Type)]
#[sqlx(transparent)]
pub struct UserId(Uuid);
```

See `references/sqlx.md` for manual trait
implementations when you need custom mapping.

---

## Transaction Scoping

Transactions auto-rollback when dropped without an
explicit `commit()`. Pass `&mut Transaction` through
functions to keep the transaction scope visible:

```rust
let mut tx = pool.begin().await?;
create_order(&mut tx, &order).await?;
charge_payment(&mut tx, &payment).await?;
tx.commit().await?;
```

If `charge_payment` fails, the `?` propagates the
error, `tx` is dropped, and everything rolls back.

See `references/transactions.md` for savepoints,
deadlock prevention, and read-only transactions.

---

## Usage Scenarios

**Scenario 1:** "I'm building a new web service
with Postgres"
→ Add `sqlx` with `runtime-tokio` and `postgres`
features. Use `sqlx-cli` for migrations. Configure
`PgPoolOptions` with 5–10 connections. Use `query!()`
for all static queries.
See `references/sqlx.md` for Cargo.toml and pool setup.

**Scenario 2:** "I need to evolve my database schema"
→ Create reversible migrations with `sqlx migrate add`.
Run with `sqlx migrate run`. Keep `sqlx-data.json` in
version control for offline compile-time checking.
See `references/sqlx.md` for migration commands.

**Scenario 3:** "I'm testing my repository layer"
→ Use `#[sqlx::test]` for automatic test database
management, or wrap each test in a transaction that
never commits. Seed data with fixture functions.
See `references/testing.md` for both strategies.

**Scenario 4:** "My domain types need to map to
database columns"
→ Derive `sqlx::Type` with `#[sqlx(transparent)]` for
simple newtypes. Implement `Type`/`Encode`/`Decode`
manually for enums or complex mappings.
See `references/sqlx.md` for full examples.

---

## Reference Files

| File | Read When |
| --- | --- |
| [references/sqlx.md](references/sqlx.md) | Cargo.toml setup, query macros, FromRow, QueryBuilder, pooling, migrations, newtype mapping |
| [references/transactions.md](references/transactions.md) | Transaction lifecycle, passing through functions, savepoints, deadlock prevention |
| [references/testing.md](references/testing.md) | Per-test databases, transaction rollback, fixtures, sqlx::test macro, CI setup |

---

## Cross-References

| When | Check |
| --- | --- |
| Repository pattern, module layout | rust-architecture → Quick Decisions |
| Repository traits, aggregate boundaries | rust-ddd → Quick Decisions |
| Pool async behavior, spawn and .await | rust-async → Quick Decisions |
| Test isolation, test builder pattern | rust-tests → Quick Decisions |
| Newtype design, derive strategies | rust-types → Quick Decisions |
| DTO mapping, serde for API responses | rust-serde → Quick Decisions |
