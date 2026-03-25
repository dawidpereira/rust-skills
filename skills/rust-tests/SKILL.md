---
name: rust-tests
description: >
  Rust unit and integration test strategy — when to write which kind of test,
  how to structure test modules, test helpers and builders, error path testing,
  parameterized tests, integration test organization, binary crate testing, and
  test isolation. Use when deciding between unit and integration tests, organizing
  a growing test suite, testing error paths, setting up shared test utilities,
  or structuring the tests/ directory for a workspace. Also use when tests are
  slow, brittle, or hard to navigate.
---

# Testing Strategy

## Core Question

**What is the right test boundary for this behavior, and how
should the test be structured for maintainability?**

Unit tests verify internal logic in isolation. Integration
tests verify that public API contracts hold when components
work together. Choosing wrong leads to either brittle tests
that break on refactoring (too-low boundary) or slow, opaque
tests that don't pinpoint failures (too-high boundary).

---

## Quick Decisions

| Situation                               | Reach For                           | Why                                                        |
| --------------------------------------- | ----------------------------------- | ---------------------------------------------------------- |
| Pure function with clear inputs/outputs | Unit test                           | Fast, precise, no setup needed                             |
| Private helper with tricky logic        | Unit test via `use super::*`        | Test the logic directly, not through layers of indirection |
| Public API contract                     | Integration test in `tests/`        | Proves the API works as consumers will use it              |
| Error path or edge case                 | Unit test per error variant         | Exhaustive, fast, catches regressions early                |
| Complex setup shared across tests       | Test builder or helper function     | DRY without sacrificing clarity                            |
| Multiple inputs, same logic             | Macro-generated parameterized tests | Covers variants without copy-paste                         |
| Binary crate behavior                   | Integration test with `Command`     | Tests the actual CLI interface end-to-end                  |
| Database or file system interaction     | Integration test with isolation     | Real dependencies need real cleanup strategies             |
| Documenting usage for library consumers | Doctest                             | Tested documentation stays accurate                        |
| Roundtrip or invariant properties       | Property test (see rust-quality)    | Finds edge cases humans miss                               |
| Mock-heavy test getting complex         | Reconsider the design               | Too many mocks signals a wrong abstraction boundary        |

---

## Key Principles

1. **Test behavior, not implementation.** Tests that mirror
   internal structure break on every refactor. Test what the
   function promises, not how it delivers.

2. **Unit tests are fast and precise.** If a unit test needs
   a database, a network, or five seconds to run, it is an
   integration test in disguise. Move it.

3. **Integration tests prove contracts.** They exercise the
   public API as a consumer would. Failures here mean you
   shipped a breaking change.

4. **Error paths need tests too.** Every `Result::Err` variant
   and every `None` return should have at least one test.
   Error paths are where bugs hide because they are where
   developers look last.

5. **Test names are documentation.** `parse_rejects_empty_input`
   tells you what broke. `test_parse_3` does not. Pattern:
   `function_condition_expected`.

6. **Shared setup belongs in helpers, not copy-pasted.** Extract
   builders and factory functions into the test module. Repeated
   arrange blocks drift apart and obscure the actual test logic.

7. **One assertion per concept.** A test can have multiple
   `assert!` calls if they verify one logical thing. Unrelated
   assertions belong in separate tests so failures are specific.

8. **Tests own their data.** Never depend on test execution
   order. Never share mutable state between tests without
   explicit isolation. Cargo runs tests in parallel by default.

---

## Usage Scenarios

**Scenario 1:** "I'm adding tests to existing code with no
coverage"
-> Start with integration tests for the public API to establish
a safety net. Then add unit tests for complex internal logic.
Integration tests catch the most breakage with the least
knowledge of internals. Unit tests follow once you understand
the code well enough to identify tricky logic worth testing
directly.

**Scenario 2:** "My test suite is slow and I don't know why"
-> Profile with `cargo test --lib` (unit only) vs
`cargo test --test '*'` (integration only). If unit tests are
slow, they are likely touching the filesystem or network — move
those to integration tests. If integration tests are slow,
look for missing isolation (shared databases, serial locks).

**Scenario 3:** "I have 50 test functions in one module and
it's hard to navigate"
-> Nest submodules inside `mod tests`: group by behavior
(`mod validation`, `mod parsing`, `mod error_cases`). Each
submodule gets `use super::*` from the parent test module.
Tests become `module::tests::validation::rejects_empty_input`.

**Scenario 4:** "I need to test a CLI binary that reads stdin
and writes files"
-> Use `std::process::Command` or `assert_cmd` in integration
tests. Create `TempDir` for file output. Assert on exit code,
stdout, and stderr. Never test CLI behavior through library
unit tests — test the actual binary.

---

## Reference Index

| Reference                                | Read When                                                                                                                                         |
| ---------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------- |
| [references/unit-tests.md][unit]         | Writing unit tests: module organization, private function testing, error paths, test builders, parameterized tests, feature flags, test filtering |
| [references/integration-tests.md][integ] | Writing integration tests: tests/ directory structure, shared utilities, binary testing, real dependencies, isolation, snapshots, workspaces      |

[unit]: references/unit-tests.md
[integ]: references/integration-tests.md

---

## Cross-References

| When                                        | Check                               |
| ------------------------------------------- | ----------------------------------- |
| Property-based testing with proptest        | rust-quality → Quick Decisions      |
| Mocking with mockall, trait extraction      | rust-quality → Quick Decisions      |
| Benchmarking with criterion                 | rust-quality → Quick Decisions      |
| Async test patterns (`#[tokio::test]`)      | rust-async → Quick Decisions        |
| Error type design affecting test assertions | rust-errors → Quick Decisions       |
| Project structure and test file placement   | rust-architecture → Quick Decisions |
