# Database Testing

## Strategy Overview

| Strategy | Isolation | Speed | Fidelity |
| --- | --- | --- | --- |
| Transaction rollback | Per-test | Fast | High |
| Per-test database | Per-test | Slower | Highest |
| Mocked repository | Per-test | Fastest | Low |

Use real databases for repository tests. Mock the
repository trait in application/service layer tests.

---

## Transaction Rollback Strategy

Wrap each test in a transaction that never commits.
The test sees its writes but they vanish on drop:

```rust
#[tokio::test]
async fn test_create_user() -> Result<(), sqlx::Error> {
    let pool = setup_test_pool().await;
    let mut tx = pool.begin().await?;

    let user_id = create_user(
        &mut *tx,
        &NewUser {
            email: "test@example.com".into(),
            name: "Test".into(),
        },
    )
    .await?;

    let user = find_user(&mut *tx, user_id).await?;
    assert_eq!(user.unwrap().email, "test@example.com");

    // tx drops here — automatic rollback, no cleanup
    Ok(())
}
```

**Advantages:** Fast, no cleanup needed, tests run
against real SQL.

**Limitations:** Cannot test commit behavior, triggers
that fire on commit, or operations that require
multiple connections.

---

## Per-Test Database Strategy

Create a fresh database for each test. Slower but
provides complete isolation:

```rust
async fn setup_test_db() -> (PgPool, String) {
    let db_name = format!("test_{}", Uuid::new_v4().simple());
    let admin_url = std::env::var("TEST_DATABASE_URL")
        .expect("TEST_DATABASE_URL must be set");

    let admin_pool = PgPool::connect(&admin_url).await.unwrap();
    sqlx::query(&format!("CREATE DATABASE \"{db_name}\""))
        .execute(&admin_pool)
        .await
        .unwrap();

    let test_url = format!(
        "{}/{}",
        admin_url.rsplit_once('/').unwrap().0,
        db_name
    );
    let pool = PgPool::connect(&test_url).await.unwrap();

    sqlx::migrate!("./migrations")
        .run(&pool)
        .await
        .unwrap();

    (pool, db_name)
}

async fn teardown_test_db(db_name: &str) {
    let admin_url = std::env::var("TEST_DATABASE_URL")
        .expect("TEST_DATABASE_URL must be set");
    let admin_pool = PgPool::connect(&admin_url).await.unwrap();

    sqlx::query(&format!(
        "DROP DATABASE IF EXISTS \"{db_name}\" WITH (FORCE)"
    ))
    .execute(&admin_pool)
    .await
    .unwrap();
}
```

---

## #[sqlx::test] Macro

sqlx provides a built-in test macro that handles
database creation and cleanup automatically:

```rust
#[sqlx::test]
async fn test_create_user(pool: PgPool) {
    let user_id = create_user(
        &pool,
        &NewUser {
            email: "test@example.com".into(),
            name: "Test".into(),
        },
    )
    .await
    .unwrap();

    let user = find_user(&pool, user_id)
        .await
        .unwrap()
        .expect("user should exist");

    assert_eq!(user.email, "test@example.com");
}
```

The macro:
- Creates a temporary database per test
- Runs all migrations automatically
- Drops the database after the test completes
- Requires `TEST_DATABASE_URL` or `DATABASE_URL`

### With fixtures

Load seed data before the test runs:

```rust
#[sqlx::test(fixtures("users", "products"))]
async fn test_place_order(pool: PgPool) {
    // fixtures/users.sql and fixtures/products.sql
    // have already been executed
    let order = place_order(&pool, &new_order()).await.unwrap();
    assert_eq!(order.items.len(), 2);
}
```

Fixture files live in `fixtures/` next to `migrations/`
and contain plain SQL INSERT statements.

---

## Test Fixtures and Seed Data

Create builder functions for test data instead of
relying on shared fixtures across tests:

```rust
fn new_user() -> NewUser {
    NewUser {
        email: format!("user-{}@test.com", Uuid::new_v4()),
        name: "Test User".into(),
    }
}

fn new_order_for(user_id: UserId) -> NewOrder {
    NewOrder {
        user_id,
        items: vec![
            OrderItem {
                product_id: ProductId(Uuid::new_v4()),
                quantity: 2,
            },
        ],
        total: Decimal::new(2999, 2),
    }
}
```

Unique emails and IDs prevent collision when tests
run in parallel against the same database.

---

## Testing Migrations

Verify that migrations apply cleanly and that rollback
works:

```rust
#[sqlx::test]
async fn migrations_apply_cleanly(pool: PgPool) {
    // #[sqlx::test] already ran all migrations.
    // If we got here, migrations succeeded.
    let result = sqlx::query("SELECT 1 as ok")
        .fetch_one(&pool)
        .await;
    assert!(result.is_ok());
}
```

For rollback testing, run the full up/down cycle in
CI:

```bash
sqlx migrate run
sqlx migrate revert --target-version 0
sqlx migrate run
```

---

## Mocking vs Real Database

### When to use a real database

- Repository and query tests
- Migration validation
- Testing constraints and triggers
- Testing transaction behavior

### When to mock

- Application/service layer tests where the repository
  is a trait dependency
- Tests that need to simulate database failures
- Tests that need deterministic behavior

```rust
#[cfg_attr(test, mockall::automock)]
#[async_trait]
pub trait UserRepository {
    async fn find(&self, id: UserId) -> Result<Option<User>>;
    async fn create(&self, user: &NewUser) -> Result<UserId>;
}

#[tokio::test]
async fn test_registration_sends_email() {
    let mut repo = MockUserRepository::new();
    repo.expect_create()
        .returning(|_| Ok(UserId(Uuid::new_v4())));

    let mut mailer = MockMailer::new();
    mailer.expect_send()
        .once()
        .returning(|_| Ok(()));

    let service = RegistrationService::new(
        Arc::new(repo),
        Arc::new(mailer),
    );

    let result = service
        .register(&NewUser {
            email: "new@example.com".into(),
            name: "New User".into(),
        })
        .await;

    assert!(result.is_ok());
}
```

---

## CI Setup

### GitHub Actions with Postgres

```yaml
name: Tests
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
          POSTGRES_DB: test
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    env:
      DATABASE_URL: postgres://test:test@localhost:5432/test

    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
      - run: cargo install sqlx-cli --no-default-features --features rustls,postgres
      - run: sqlx database setup
      - run: cargo test
```

### Docker Compose for local development

```yaml
services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_USER: dev
      POSTGRES_PASSWORD: dev
      POSTGRES_DB: app_dev
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data

  postgres-test:
    image: postgres:16
    environment:
      POSTGRES_USER: test
      POSTGRES_PASSWORD: test
      POSTGRES_DB: app_test
    ports:
      - "5433:5432"

volumes:
  pgdata:
```

Use separate instances for development and testing to
avoid data interference.

### Environment setup

```bash
# .env (development)
DATABASE_URL=postgres://dev:dev@localhost:5432/app_dev

# .env.test (testing)
DATABASE_URL=postgres://test:test@localhost:5433/app_test
```

Run tests with the test environment:

```bash
DATABASE_URL=postgres://test:test@localhost:5433/app_test \
    cargo test
```
