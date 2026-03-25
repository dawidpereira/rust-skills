# Testing Reference

For comprehensive unit and integration test strategy — module organization, test builders,
error path testing, parameterized tests, test isolation, binary crate testing, and snapshot
testing — see the **rust-tests** skill.

This reference covers advanced testing tools: proptest, mockall, criterion, tokio::test,
RAII fixtures, doctests, and `#[should_panic]`.

## Quick Reference: Test Commands

```bash
cargo test                     # All non-ignored tests
cargo test --lib               # Unit tests only
cargo test --test '*'          # Integration tests only
cargo test --test api_tests    # Specific integration test file
cargo test -- --ignored        # Only ignored tests
cargo test parse               # Tests with "parse" in the name
```

## Proptest: Property-Based Testing

Generate random inputs to verify properties hold across all values.

```toml
[dev-dependencies]
proptest = "1.0"
```

```rust
use proptest::prelude::*;

proptest! {
    #[test]
    fn reverse_reverse_is_identity(s in ".*") {
        let reversed: String = s.chars().rev().collect();
        let double_reversed: String = reversed.chars().rev().collect();
        assert_eq!(s, double_reversed);
    }

    #[test]
    fn sort_is_idempotent(mut v in prop::collection::vec(any::<i32>(), 0..100)) {
        v.sort();
        let sorted = v.clone();
        v.sort();
        assert_eq!(v, sorted);
    }
}
```

| Property | Example |
|----------|---------|
| Roundtrip | `decode(encode(x)) == x` |
| Idempotence | `f(f(x)) == f(x)` |
| Invariants | `len(push(v, x)) == len(v) + 1` |

Custom strategies:

```rust
fn user_strategy() -> impl Strategy<Value = User> {
    ("[a-zA-Z]{1,20}", 0..120u8)
        .prop_map(|(name, age)| User { name, age })
}
```

## Mockall: Trait Mocking

Extract dependencies into traits and use mockall for test doubles.

```toml
[dev-dependencies]
mockall = "0.12"
```

```rust
use mockall::automock;

#[automock]
trait Database {
    fn get_user(&self, id: u64) -> Option<User>;
}

struct UserService<D: Database> { db: D }

impl<D: Database> UserService<D> {
    fn find_user(&self, id: u64) -> Result<User, Error> {
        self.db.get_user(id).ok_or(Error::NotFound)
    }
}

#[cfg(test)]
mod tests {
    use super::*;
    use mockall::predicate::*;

    #[test]
    fn find_user_returns_user_when_found() {
        let mut mock = MockDatabase::new();
        mock.expect_get_user()
            .with(eq(42))
            .returning(|_| Some(User { id: 42, name: "Alice".into() }));

        let service = UserService { db: mock };
        assert_eq!(service.find_user(42).unwrap().name, "Alice");
    }

    #[test]
    fn find_user_returns_error_when_not_found() {
        let mut mock = MockDatabase::new();
        mock.expect_get_user().returning(|_| None);

        let service = UserService { db: mock };
        assert!(matches!(service.find_user(999), Err(Error::NotFound)));
    }
}
```

For async traits, combine with `async-trait`:

```rust
#[automock]
#[async_trait]
trait AsyncDatabase: Send + Sync {
    async fn fetch(&self, id: u64) -> Option<Data>;
}
```

## Criterion: Benchmarking

Use criterion for statistically rigorous benchmarks. Always use `black_box` to prevent
compiler optimization.

```toml
[dev-dependencies]
criterion = "0.5"

[[bench]]
name = "my_benchmark"
harness = false
```

```rust
use criterion::{black_box, criterion_group, criterion_main, Criterion};

fn bench_fibonacci(c: &mut Criterion) {
    c.bench_function("fib 20", |b| {
        b.iter(|| fibonacci(black_box(20)))
    });
}

criterion_group!(benches, bench_fibonacci);
criterion_main!(benches);
```

Use `benchmark_group` to compare implementations side by side.

## Tokio Async Tests

Use `#[tokio::test]` instead of manually creating a runtime.

```rust
#[tokio::test]
async fn fetch_user_returns_data() {
    let client = TestClient::new();
    let result = client.fetch_user(42).await;
    assert!(result.is_ok());
}

#[tokio::test(flavor = "multi_thread", worker_threads = 2)]
async fn concurrent_operations_succeed() { /* ... */ }
```

## RAII Test Fixtures

Use Drop-based guards for automatic cleanup, even on panic.

```rust
use tempfile::NamedTempFile;

#[test]
fn process_file_succeeds() {
    let file = NamedTempFile::new().unwrap();
    std::fs::write(file.path(), "test data").unwrap();
    assert!(process_file(file.path()).is_ok());
}
```

Write custom RAII guards (implementing `Drop`) for environment variables, test servers,
or database transactions. The guard restores original state when dropped.

## `#[should_panic]`

Verify that code panics with the expected message:

```rust
#[test]
#[should_panic(expected = "NonEmpty cannot be empty")]
fn non_empty_rejects_empty_vec() {
    NonEmpty::new(Vec::<i32>::new());
}
```

Prefer `Result` over panics. Reserve `#[should_panic]` for invariant violations only.

## Doctests

Keep documentation examples as executable tests:

```rust
/// Parses a number from a string.
///
/// # Examples
///
/// ```
/// use my_crate::parse;
/// assert_eq!(parse("42"), 42);
/// ```
pub fn parse(s: &str) -> i32 {
    s.parse().unwrap()
}
```

Hide setup code with `# ` prefix. Use `no_run` for blocking examples, `compile_fail`
to verify invalid code does not compile.
