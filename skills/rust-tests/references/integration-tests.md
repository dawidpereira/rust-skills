# Integration Tests Reference

## tests/ Directory Organization

Integration tests live in the `tests/` directory at the crate root. Each `.rs`
file compiles as a separate crate and can only access the public API.

### Flat layout (< 5 test files)

```
my_project/
├── src/lib.rs
└── tests/
    ├── api_tests.rs
    ├── cli_tests.rs
    └── persistence_tests.rs
```

### Nested layout (5+ test files or shared utilities)

```
my_project/
├── src/lib.rs
└── tests/
    ├── common/
    │   └── mod.rs            # Shared test utilities
    ├── api/
    │   ├── mod.rs            # mod declarations
    │   ├── users.rs
    │   └── orders.rs
    ├── api_tests.rs          # uses mod api; mod common;
    └── cli_tests.rs
```

Cargo treats every `.rs` file directly inside `tests/` as a test crate.
Files inside subdirectories (like `common/mod.rs`) are NOT compiled as
separate test crates — they must be imported with `mod common;` from a
top-level test file.

### Naming convention

Name test files by what they exercise, not by their test type:

```
tests/user_registration.rs   # Good: describes the behavior
tests/test_users.rs           # Avoid: "test_" prefix is redundant
```

---

## Shared Test Utilities (common/mod.rs)

The `tests/common/mod.rs` pattern provides shared helpers across integration
test files. Using `mod.rs` inside a directory prevents Cargo from treating
it as a standalone test crate.

```rust
// tests/common/mod.rs
use my_crate::Config;

pub fn test_config() -> Config {
    Config {
        database_url: "postgres://localhost/test".to_string(),
        port: 0,
        log_level: "error".to_string(),
    }
}

pub fn setup_test_client() -> my_crate::Client {
    my_crate::Client::new(test_config())
}
```

```rust
// tests/api_tests.rs
mod common;

#[test]
fn create_user_returns_201() {
    let client = common::setup_test_client();
    let response = client.create_user("Alice", "alice@example.com");
    assert_eq!(response.status(), 201);
}
```

### What belongs in common

- Configuration builders for test environments
- Client setup and teardown helpers
- Assertion helpers specific to integration tests
- Test data factories

### What does NOT belong in common

- Actual test functions (Cargo won't discover them)
- Test-specific setup that only one file uses (keep it local)
- Mocks (those belong in unit tests, not integration tests)

---

## Testing Binary Crates

Binary crates (`src/main.rs`) cannot be imported as a library. Test them
by running the compiled binary as a subprocess.

### Using `std::process::Command`

```rust
use std::process::Command;

#[test]
fn cli_prints_version() {
    let output = Command::new(env!("CARGO_BIN_EXE_my_tool"))
        .arg("--version")
        .output()
        .expect("failed to execute binary");

    assert!(output.status.success());
    let stdout = String::from_utf8_lossy(&output.stdout);
    assert!(stdout.contains("my_tool 0.1"));
}

#[test]
fn cli_fails_on_missing_input() {
    let output = Command::new(env!("CARGO_BIN_EXE_my_tool"))
        .arg("--input")
        .arg("/nonexistent/file.txt")
        .output()
        .expect("failed to execute binary");

    assert!(!output.status.success());
    let stderr = String::from_utf8_lossy(&output.stderr);
    assert!(stderr.contains("file not found"));
}
```

`env!("CARGO_BIN_EXE_my_tool")` resolves to the path of the built binary.
Replace `my_tool` with the binary name from `Cargo.toml`.

### Using `assert_cmd` for ergonomic CLI testing

```toml
[dev-dependencies]
assert_cmd = "2"
predicates = "3"
```

```rust
use assert_cmd::Command;
use predicates::prelude::*;

#[test]
fn cli_processes_input_file() {
    let mut cmd = Command::cargo_bin("my_tool").unwrap();
    cmd.arg("--input")
        .arg("tests/fixtures/sample.txt")
        .assert()
        .success()
        .stdout(predicate::str::contains("processed 10 lines"));
}
```

### Testing stdin

```rust
#[test]
fn cli_reads_from_stdin() {
    let mut cmd = Command::cargo_bin("my_tool").unwrap();
    cmd.write_stdin("line1\nline2\n")
        .assert()
        .success()
        .stdout(predicate::str::contains("2 lines"));
}
```

---

## Testing with Real Dependencies

### Temp directories for file operations

```rust
use tempfile::TempDir;

#[test]
fn export_writes_csv_file() {
    let dir = TempDir::new().unwrap();
    let output_path = dir.path().join("export.csv");

    my_crate::export_to_csv(&data, &output_path).unwrap();

    let content = std::fs::read_to_string(&output_path).unwrap();
    assert!(content.contains("Alice,30"));
}
// TempDir dropped here — directory is deleted automatically
```

### Unique ports for network tests

Bind to port 0 and let the OS assign a free port:

```rust
#[test]
fn server_responds_to_health_check() {
    let listener = std::net::TcpListener::bind("127.0.0.1:0").unwrap();
    let port = listener.local_addr().unwrap().port();

    let server = my_crate::Server::new(listener);
    let handle = std::thread::spawn(move || server.run());

    let response = reqwest::blocking::get(
        format!("http://127.0.0.1:{port}/health")
    ).unwrap();
    assert_eq!(response.status(), 200);

    // Shut down server...
}
```

### Database test isolation with transactions

Wrap each test in a transaction that rolls back:

```rust
use sqlx::PgPool;

async fn test_pool() -> PgPool {
    PgPool::connect("postgres://localhost/test_db")
        .await
        .unwrap()
}

#[tokio::test]
async fn create_user_persists_to_db() {
    let pool = test_pool().await;
    let mut tx = pool.begin().await.unwrap();

    let user_id = my_crate::create_user(&mut *tx, "Alice").await.unwrap();
    let user = my_crate::get_user(&mut *tx, user_id).await.unwrap();

    assert_eq!(user.name, "Alice");
    // tx dropped without commit — changes are rolled back
}
```

---

## Test Isolation Strategies

Cargo runs tests in parallel by default. Tests that share mutable global
state (environment variables, files, databases) will interfere with each
other unless isolated.

### Strategies by resource type

| Resource | Isolation Strategy |
|----------|--------------------|
| Files | `TempDir` per test — unique paths, automatic cleanup |
| Ports | Bind to port 0 — OS assigns unique port |
| Database | Transaction per test — rollback on drop |
| Environment variables | RAII guard (see unit-tests.md) + `serial_test` |
| Global/static state | `serial_test` crate — forces serial execution |

### `serial_test` for shared mutable state

```toml
[dev-dependencies]
serial_test = "3"
```

```rust
use serial_test::serial;

#[test]
#[serial]
fn test_that_modifies_env() {
    std::env::set_var("APP_MODE", "test");
    let config = Config::from_env();
    assert_eq!(config.mode, "test");
    std::env::remove_var("APP_MODE");
}

#[test]
#[serial]
fn test_that_also_reads_env() {
    let config = Config::from_env();
    assert_eq!(config.mode, "production"); // default when APP_MODE is unset
}
```

Tests marked `#[serial]` run one at a time. Non-serial tests still run in
parallel. Use this sparingly — serial tests slow down the suite.

### Detecting order-dependent tests

If tests pass individually but fail together, run them single-threaded to
confirm:

```bash
cargo test -- --test-threads=1
```

If they pass single-threaded but fail in parallel, you have shared state.
Find the shared resource and isolate it.

---

## Testing Public API Contracts

Integration tests import only the crate's public API. This is intentional —
they verify the contract that external consumers depend on.

```rust
// tests/api_contract.rs
use my_crate::{Client, Config, Error};

#[test]
fn client_returns_not_found_for_missing_user() {
    let client = Client::new(Config::default());
    let result = client.get_user(99999);
    assert!(matches!(result, Err(Error::NotFound)));
}

#[test]
fn client_creates_and_retrieves_user() {
    let client = Client::new(Config::default());
    let id = client.create_user("Alice").unwrap();
    let user = client.get_user(id).unwrap();
    assert_eq!(user.name, "Alice");
}
```

These tests catch accidental breaking changes — if you rename a public type,
change a return type, or remove a method, integration tests fail before your
users do.

### What makes a good API contract test

- Uses only public types and functions
- Tests realistic workflows (create then retrieve, not just create)
- Asserts on behavior, not on internal representation
- Fails when the public API changes in a breaking way

---

## Snapshot Testing with insta

Snapshot tests capture output and compare it against a stored reference.
Useful for formatters, serializers, error messages, and any output-heavy code.

```toml
[dev-dependencies]
insta = { version = "1", features = ["yaml"] }
```

### Basic snapshot

```rust
use insta::assert_snapshot;

#[test]
fn error_message_format() {
    let err = validate("").unwrap_err();
    assert_snapshot!(err.to_string());
}
```

First run creates `snapshots/module__error_message_format.snap`. Review and
accept with:

```bash
cargo insta test           # Run tests, create pending snapshots
cargo insta review         # Interactive review of pending snapshots
```

### Structured data snapshots

```rust
use insta::assert_yaml_snapshot;

#[test]
fn user_serialization() {
    let user = User { name: "Alice".into(), age: 30 };
    assert_yaml_snapshot!(user);
}
```

### When snapshots help vs hurt

| Good fit | Poor fit |
|----------|----------|
| Formatter output (many details to check) | Simple equality (use `assert_eq!`) |
| Error message regression | Highly volatile output (changes every run) |
| Serialization format stability | Test logic that needs explanation (snapshots are opaque) |
| CLI output verification | Performance-sensitive tests (snapshot I/O adds overhead) |

Snapshots become a maintenance burden when the output changes frequently for
legitimate reasons. If you find yourself updating snapshots on every PR,
the output is too volatile for snapshot testing.

---

## Workspace Integration Testing

### Top-level tests/ for cross-crate integration

```
my_workspace/
├── Cargo.toml              # [workspace]
├── crates/
│   ├── core/
│   ├── api/
│   └── cli/
└── tests/
    ├── common/mod.rs
    └── end_to_end.rs
```

The top-level `tests/` directory can import any workspace member:

```rust
// tests/end_to_end.rs
use core_crate::Domain;
use api_crate::Router;

#[test]
fn api_creates_domain_object() {
    let router = Router::new(Domain::default());
    let response = router.handle_request("/create", "{}");
    assert_eq!(response.status, 201);
}
```

### Targeted test runs

```bash
cargo test -p core-crate           # Tests for one crate
cargo test -p api-crate --lib      # Unit tests only for one crate
cargo test --workspace             # All tests in all crates
cargo test --workspace --test '*'  # Integration tests across workspace
```

### Shared test utilities across workspace

Create a `test-utils` crate in the workspace:

```toml
# crates/test-utils/Cargo.toml
[package]
name = "test-utils"
publish = false

[dependencies]
core-crate = { path = "../core" }
```

```toml
# crates/api/Cargo.toml
[dev-dependencies]
test-utils = { path = "../test-utils" }
```

This avoids duplicating test helpers across crates.

---

## Test Ordering and Independence

### Why test order must not matter

Cargo runs tests in parallel by default and does not guarantee execution
order. Tests that depend on running after another test will eventually fail
in CI, often intermittently and frustratingly.

### Common causes of order-dependent tests

| Cause | Fix |
|-------|-----|
| Shared mutable global (`lazy_static`, `OnceLock`) | Use `serial_test` or redesign to avoid global state |
| Tests writing to the same file path | Use `TempDir` for unique paths per test |
| Database rows left by a previous test | Use transaction-per-test with rollback |
| Environment variables set by another test | Use RAII guard + `serial_test` |

### Detecting the problem

```bash
cargo test                       # Fails intermittently
cargo test -- --test-threads=1   # Passes — confirms ordering issue
```

If single-threaded passes but parallel fails, you have shared mutable state.
Find the shared resource and isolate each test's access to it.

### Anti-pattern: test setup that relies on other tests

```rust
// BAD: test_create must run before test_read
#[test]
fn test_create() {
    create_user("Alice");  // writes to shared DB
}

#[test]
fn test_read() {
    let user = get_user("Alice");  // assumes test_create ran first
    assert_eq!(user.name, "Alice");
}
```

```rust
// GOOD: each test creates its own data
#[test]
fn create_user_persists() {
    let db = fresh_test_db();
    create_user(&db, "Alice");
    let user = get_user(&db, "Alice");
    assert_eq!(user.name, "Alice");
}
```

Each test must set up everything it needs. The only shared setup should be
in helper functions that create isolated instances.
