# Unit Tests Reference

## Test Module Organization

Place unit tests in the same file as the code they test. The `#[cfg(test)]`
attribute excludes test code from release builds.

```rust
pub fn validate_email(input: &str) -> Result<Email, ValidationError> {
    // ...
}

pub fn parse_config(raw: &str) -> Result<Config, ParseError> {
    // ...
}

#[cfg(test)]
mod tests {
    use super::*;

    mod validation {
        use super::*;

        #[test]
        fn email_accepts_standard_format() {
            assert!(validate_email("user@example.com").is_ok());
        }

        #[test]
        fn email_rejects_missing_at_sign() {
            assert!(matches!(
                validate_email("invalid"),
                Err(ValidationError::MissingAtSign)
            ));
        }
    }

    mod parsing {
        use super::*;

        #[test]
        fn config_parses_valid_toml() {
            let raw = r#"port = 8080"#;
            assert_eq!(parse_config(raw).unwrap().port, 8080);
        }
    }
}
```

Nest submodules when a test file grows beyond ~20 tests. Group by behavior
(`mod validation`, `mod error_cases`, `mod edge_cases`), not by test type.
Each submodule inherits `use super::*` from the parent.

For very large modules, consider a separate test file alongside the source:

```
src/
├── parser.rs
└── parser_tests.rs   # #[cfg(test)] #[path = "parser_tests.rs"] mod tests;
```

Use `#[path]` in the source file to include it:

```rust
// src/parser.rs
#[cfg(test)]
#[path = "parser_tests.rs"]
mod tests;
```

---

## Testing Private vs Public Functions

Unit tests live inside the module, so they have access to private items via
`use super::*`. The question is whether you *should* test them directly.

| Situation | Approach | Why |
|-----------|----------|-----|
| Private function with complex logic | Test directly | Avoids convoluted public-API-only setups |
| Private function that is a thin wrapper | Test through the public API | Direct test adds no value and couples to internals |
| Private function you might refactor soon | Test through the public API | Direct test would break on refactor |
| Private function with many edge cases | Test directly, accept the coupling trade-off | Better to catch edge-case bugs than to avoid coupling |

The trade-off: direct tests on privates break when you refactor internals.
If you accept that trade-off for a complex helper, do it. If the function
is simple enough that its behavior is fully exercised through public callers,
skip the direct test.

---

## Test Helpers and Builders

When multiple tests need similar setup, extract the shared logic into helpers
within the test module. This keeps arrange blocks short and focused on what
makes each test unique.

```rust
#[cfg(test)]
mod tests {
    use super::*;

    fn test_order() -> Order {
        Order {
            id: OrderId::new(Uuid::nil()),
            items: vec![],
            status: OrderStatus::Draft,
            created_at: Utc::now(),
        }
    }

    fn test_item(name: &str, price: u64) -> OrderItem {
        OrderItem {
            name: name.to_string(),
            price: Money::from_cents(price),
            quantity: 1,
        }
    }

    #[test]
    fn order_total_sums_items() {
        let mut order = test_order();
        order.items.push(test_item("Widget", 1000));
        order.items.push(test_item("Gadget", 2000));

        assert_eq!(order.total(), Money::from_cents(3000));
    }
}
```

For more complex setup, use a builder:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    struct TestOrder {
        order: Order,
    }

    impl TestOrder {
        fn new() -> Self {
            Self { order: test_order() }
        }

        fn with_item(mut self, name: &str, price: u64) -> Self {
            self.order.items.push(test_item(name, price));
            self
        }

        fn with_status(mut self, status: OrderStatus) -> Self {
            self.order.status = status;
            self
        }

        fn build(self) -> Order {
            self.order
        }
    }

    #[test]
    fn paid_order_cannot_add_items() {
        let order = TestOrder::new()
            .with_item("Widget", 1000)
            .with_status(OrderStatus::Paid)
            .build();

        assert!(order.add_item(test_item("Extra", 500)).is_err());
    }
}
```

Use `Default` when the struct supports it and most fields don't matter:

```rust
#[cfg(test)]
impl Default for Config {
    fn default() -> Self {
        Self {
            port: 8080,
            host: "localhost".to_string(),
            max_connections: 100,
        }
    }
}
```

---

## Testing Error Paths Systematically

Every `Err` variant and every `None` return should have at least one dedicated
test. Pattern: one test per error condition.

```rust
#[derive(Debug, thiserror::Error)]
pub enum ParseError {
    #[error("input is empty")]
    EmptyInput,
    #[error("invalid format: {0}")]
    InvalidFormat(String),
    #[error("value out of range: {value} (max {max})")]
    OutOfRange { value: u64, max: u64 },
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn parse_rejects_empty_input() {
        assert!(matches!(
            parse(""),
            Err(ParseError::EmptyInput)
        ));
    }

    #[test]
    fn parse_rejects_invalid_format() {
        assert!(matches!(
            parse("not-a-number"),
            Err(ParseError::InvalidFormat(_))
        ));
    }

    #[test]
    fn parse_rejects_out_of_range() {
        assert!(matches!(
            parse("999999"),
            Err(ParseError::OutOfRange { .. })
        ));
    }

    #[test]
    fn parse_out_of_range_includes_value_in_message() {
        let err = parse("999999").unwrap_err();
        assert!(err.to_string().contains("999999"));
    }
}
```

Test error messages when they are part of the contract (user-facing errors,
API responses). For internal errors, testing the variant is sufficient.

---

## Assertion Patterns

### Basic assertions

```rust
assert_eq!(result, expected);          // Equality with debug output
assert_ne!(result, other);             // Inequality
assert!(condition);                     // Boolean
assert!(condition, "got {result}");     // Boolean with message
```

### Pattern matching

```rust
assert!(matches!(result, Ok(value) if value > 0));
assert!(matches!(result, Err(MyError::NotFound)));
```

### Custom assertion helpers

When the same assertion logic appears in multiple tests, extract it:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    fn assert_contains(haystack: &str, needle: &str) {
        assert!(
            haystack.contains(needle),
            "expected {haystack:?} to contain {needle:?}"
        );
    }

    fn assert_err_variant(result: Result<Order, OrderError>, expected: &str) {
        let err = result.expect_err("expected an error");
        assert_contains(&err.to_string(), expected);
    }

    #[test]
    fn cancel_paid_order_explains_why() {
        let order = TestOrder::new()
            .with_status(OrderStatus::Paid)
            .build();
        assert_err_variant(order.cancel(), "cannot cancel a paid order");
    }
}
```

### `#[should_panic]`

Reserve for invariant violations only — where a panic is the intended behavior:

```rust
#[test]
#[should_panic(expected = "index out of bounds")]
fn get_panics_on_invalid_index() {
    let list = NonEmptyList::from(vec![1, 2, 3]);
    let _ = list[5];
}
```

Prefer `Result`-based testing for everything else. `#[should_panic]` cannot
distinguish between the expected panic and an unrelated one.

---

## Parameterized-Style Testing

Rust has no built-in parameterized tests, but macros handle it cleanly.

### Inline macro for simple cases

```rust
#[cfg(test)]
mod tests {
    use super::*;

    macro_rules! parse_valid {
        ($name:ident, $input:expr, $expected:expr) => {
            #[test]
            fn $name() {
                assert_eq!(parse($input).unwrap(), $expected);
            }
        };
    }

    parse_valid!(parses_zero, "0", 0);
    parse_valid!(parses_positive, "42", 42);
    parse_valid!(parses_with_leading_zeros, "007", 7);
    parse_valid!(parses_max, "255", 255);

    macro_rules! parse_invalid {
        ($name:ident, $input:expr) => {
            #[test]
            fn $name() {
                assert!(parse($input).is_err());
            }
        };
    }

    parse_invalid!(rejects_empty, "");
    parse_invalid!(rejects_negative, "-1");
    parse_invalid!(rejects_overflow, "256");
    parse_invalid!(rejects_letters, "abc");
}
```

Each macro invocation generates a separate `#[test]` function, so failures
report the exact case that broke — unlike a loop inside one test.

### `test-case` crate

For larger parameter sets, the `test-case` crate provides attribute syntax:

```rust
use test_case::test_case;

#[test_case("0", 0 ; "zero")]
#[test_case("42", 42 ; "positive")]
#[test_case("007", 7 ; "leading zeros")]
fn parse_valid(input: &str, expected: u8) {
    assert_eq!(parse(input).unwrap(), expected);
}
```

---

## Test Fixtures Without External Crates

### RAII environment guard

```rust
#[cfg(test)]
struct EnvGuard {
    key: String,
    original: Option<String>,
}

#[cfg(test)]
impl EnvGuard {
    fn set(key: &str, value: &str) -> Self {
        let original = std::env::var(key).ok();
        std::env::set_var(key, value);
        Self { key: key.to_string(), original }
    }
}

#[cfg(test)]
impl Drop for EnvGuard {
    fn drop(&mut self) {
        match &self.original {
            Some(val) => std::env::set_var(&self.key, val),
            None => std::env::remove_var(&self.key),
        }
    }
}

#[test]
fn reads_custom_port_from_env() {
    let _guard = EnvGuard::set("APP_PORT", "9090");
    assert_eq!(Config::from_env().port, 9090);
}
```

The guard restores the original value even if the test panics, because
`Drop` runs during unwinding.

### Temp file fixture

```rust
#[cfg(test)]
fn temp_file_with(content: &str) -> (tempfile::NamedTempFile, std::path::PathBuf) {
    let file = tempfile::NamedTempFile::new().unwrap();
    std::fs::write(file.path(), content).unwrap();
    let path = file.path().to_path_buf();
    (file, path)
}

#[test]
fn process_reads_file_contents() {
    let (_file, path) = temp_file_with("hello");
    assert_eq!(process_file(&path).unwrap(), "HELLO");
}
```

Keep the `_file` binding alive — dropping it deletes the temp file.

---

## Feature Flag Testing

Gate test modules behind features when the code under test is feature-gated:

```rust
#[cfg(feature = "json")]
pub fn to_json<T: Serialize>(value: &T) -> String {
    serde_json::to_string(value).unwrap()
}

#[cfg(test)]
#[cfg(feature = "json")]
mod json_tests {
    use super::*;

    #[test]
    fn serializes_struct_to_json() {
        let user = User { name: "Alice".into(), age: 30 };
        let json = to_json(&user);
        assert!(json.contains("Alice"));
    }
}
```

Run feature-specific tests:

```bash
cargo test --features json         # With feature
cargo test --no-default-features   # Without defaults
cargo test --all-features          # Everything
```

---

## `#[ignore]` and Test Filtering

### Marking slow or external-dependency tests

```rust
#[test]
#[ignore = "requires running PostgreSQL"]
fn integration_with_real_database() {
    // ...
}
```

### Running strategies

```bash
cargo test                     # All non-ignored tests
cargo test -- --ignored        # Only ignored tests
cargo test -- --include-ignored # Everything including ignored

cargo test parse               # Tests with "parse" in the name
cargo test --lib               # Unit tests only
cargo test --test '*'          # Integration tests only
cargo test --test api_tests    # Specific integration test file
```

### CI strategy

Use two stages:

1. **Fast gate** (every push): `cargo test` — runs all non-ignored tests
2. **Nightly or pre-release**: `cargo test -- --include-ignored` — runs everything
   including slow tests that need external services

This keeps the fast feedback loop tight while still catching issues from
expensive tests on a regular cadence.
