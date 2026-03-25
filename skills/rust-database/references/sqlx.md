# sqlx Setup & Usage

## Cargo.toml

Choose a runtime and TLS backend. Enable the database
driver you need:

```toml
[dependencies]
sqlx = { version = "0.8", features = [
    "runtime-tokio",
    "tls-rustls",
    "postgres",
    "uuid",
    "chrono",
    "migrate",
] }
```

### Feature matrix

| Feature | Purpose |
| --- | --- |
| `runtime-tokio` | Tokio async runtime integration |
| `runtime-async-std` | async-std alternative |
| `tls-rustls` | Pure-Rust TLS (no OpenSSL) |
| `tls-native-tls` | System TLS (OpenSSL/Schannel) |
| `postgres` | PostgreSQL driver |
| `sqlite` | SQLite driver |
| `mysql` | MySQL/MariaDB driver |
| `uuid` | `Uuid` type support |
| `chrono` | `NaiveDateTime`, `DateTime<Utc>` |
| `time` | `time` crate types instead of chrono |
| `migrate` | Embedded migrations at compile time |

---

## Connection Pooling

```rust
use sqlx::postgres::PgPoolOptions;

let pool = PgPoolOptions::new()
    .max_connections(10)
    .min_connections(2)
    .acquire_timeout(std::time::Duration::from_secs(5))
    .idle_timeout(std::time::Duration::from_secs(600))
    .connect(&database_url)
    .await?;
```

### Sizing guidelines

- **Start with 5–10** for a typical web service.
- **Formula:** CPU cores × 2 + disk spindles. For
  SSDs, treat spindles as 1.
- **Monitor** `pool.size()` and acquire wait times.
  If acquires frequently timeout, increase the pool
  or optimize slow queries.
- **Don't over-provision.** Each connection holds
  server memory. 50+ connections usually means
  queries are too slow or the pool is leaking.

---

## query!() — Compile-Time Checked

Requires `DATABASE_URL` at compile time (env var or
`.env` file):

```rust
let user = sqlx::query!(
    "SELECT id, email, name FROM users WHERE id = $1",
    user_id
)
.fetch_one(&pool)
.await?;

println!("{}: {}", user.id, user.email);
```

The macro verifies column names, types, and parameter
count against the live database at compile time. A
schema mismatch is a compile error, not a runtime
surprise.

### Offline mode

For CI without a database, cache query metadata:

```bash
cargo sqlx prepare
```

This generates `.sqlx/` directory with query metadata.
Commit it to version control. sqlx checks these files
when `DATABASE_URL` is not set and `SQLX_OFFLINE=true`.

---

## query_as!() — Struct Mapping

Map rows directly to a named struct:

```rust
struct User {
    id: Uuid,
    email: String,
    name: Option<String>,
}

let users = sqlx::query_as!(
    User,
    "SELECT id, email, name FROM users WHERE active = $1",
    true
)
.fetch_all(&pool)
.await?;
```

### FromRow derive

For runtime queries, derive `FromRow` instead:

```rust
#[derive(sqlx::FromRow)]
struct User {
    id: Uuid,
    email: String,
    #[sqlx(rename = "display_name")]
    name: Option<String>,
}

let user: User = sqlx::query_as("SELECT * FROM users WHERE id = $1")
    .bind(user_id)
    .fetch_one(&pool)
    .await?;
```

`FromRow` attributes:
- `#[sqlx(rename = "col")]` — map to different column
- `#[sqlx(default)]` — use `Default::default()` if NULL
- `#[sqlx(flatten)]` — flatten nested struct fields
- `#[sqlx(skip)]` — skip field, use Default

---

## query() — Runtime Queries

No compile-time checking. Use when the query is
dynamic or `DATABASE_URL` is unavailable:

```rust
let row = sqlx::query("SELECT id, email FROM users WHERE id = $1")
    .bind(user_id)
    .fetch_one(&pool)
    .await?;

let email: String = row.get("email");
```

---

## QueryBuilder — Dynamic Queries

Build queries programmatically for dynamic filters
or bulk operations:

```rust
use sqlx::QueryBuilder;

let mut qb = QueryBuilder::new(
    "SELECT id, email FROM users WHERE 1=1"
);

if let Some(name) = &filter.name {
    qb.push(" AND name ILIKE ");
    qb.push_bind(format!("%{name}%"));
}

if let Some(active) = filter.active {
    qb.push(" AND active = ");
    qb.push_bind(active);
}

qb.push(" ORDER BY created_at DESC LIMIT ");
qb.push_bind(filter.limit.unwrap_or(50));

let users = qb
    .build_query_as::<User>()
    .fetch_all(&pool)
    .await?;
```

### Bulk inserts

```rust
let mut qb = QueryBuilder::new(
    "INSERT INTO events (name, payload, created_at) "
);

qb.push_values(events.iter(), |mut b, event| {
    b.push_bind(&event.name)
     .push_bind(&event.payload)
     .push_bind(event.created_at);
});

qb.build().execute(&pool).await?;
```

---

## Fetch Methods

| Method | Returns | Use When |
| --- | --- | --- |
| `fetch_one()` | Single row or error | Exactly one row expected |
| `fetch_optional()` | `Option<Row>` | Zero or one row |
| `fetch_all()` | `Vec<Row>` | All rows into memory |
| `fetch()` | `Stream<Row>` | Large result sets, process incrementally |

Prefer `fetch_optional()` over `fetch_one()` when the
row might not exist — it returns `None` instead of an
error.

---

## Migrations

### Setup

```bash
cargo install sqlx-cli --no-default-features \
    --features rustls,postgres

sqlx database create
sqlx migrate add create_users
```

This creates a reversible migration pair:
- `migrations/YYYYMMDDHHMMSS_create_users.up.sql`
- `migrations/YYYYMMDDHHMMSS_create_users.down.sql`

### Writing migrations

```sql
-- migrations/20240101000000_create_users.up.sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email TEXT NOT NULL UNIQUE,
    name TEXT,
    active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_users_email ON users (email);
```

```sql
-- migrations/20240101000000_create_users.down.sql
DROP TABLE IF EXISTS users;
```

### Running migrations

```bash
sqlx migrate run          # apply pending
sqlx migrate revert       # rollback last
sqlx migrate info         # show status
```

### Running migrations in code

```rust
sqlx::migrate!("./migrations")
    .run(&pool)
    .await?;
```

The `migrate!()` macro embeds migration files into the
binary at compile time.

---

## Newtype Mapping

### Transparent wrapper (derive)

For newtypes that wrap a single SQL-compatible type:

```rust
#[derive(Debug, Clone, sqlx::Type)]
#[sqlx(transparent)]
pub struct UserId(Uuid);

#[derive(Debug, Clone, sqlx::Type)]
#[sqlx(transparent)]
pub struct Email(String);
```

These work directly in `query!()` and `query_as!()`:

```rust
let user = sqlx::query_as!(
    UserRow,
    "SELECT id, email FROM users WHERE id = $1",
    user_id as UserId
)
.fetch_one(&pool)
.await?;
```

### Enum mapping

Map Rust enums to Postgres enums or text columns:

```rust
#[derive(Debug, Clone, sqlx::Type)]
#[sqlx(type_name = "user_role", rename_all = "lowercase")]
pub enum UserRole {
    Admin,
    Member,
    Guest,
}
```

Requires a matching Postgres enum:

```sql
CREATE TYPE user_role AS ENUM ('admin', 'member', 'guest');
```

### Manual implementation

When derive doesn't fit — for example, mapping a
newtype with validation:

```rust
use sqlx::{Database, Type, Encode, Decode};
use sqlx::encode::IsNull;
use sqlx::error::BoxDynError;

impl Type<sqlx::Postgres> for Email {
    fn type_info() -> sqlx::postgres::PgTypeInfo {
        <String as Type<sqlx::Postgres>>::type_info()
    }
}

impl<'q> Encode<'q, sqlx::Postgres> for Email {
    fn encode_by_ref(
        &self,
        buf: &mut <sqlx::Postgres as Database>::ArgumentBuffer<'q>,
    ) -> Result<IsNull, BoxDynError> {
        self.0.encode_by_ref(buf)
    }
}

impl<'r> Decode<'r, sqlx::Postgres> for Email {
    fn decode(
        value: <sqlx::Postgres as Database>::ValueRef<'r>,
    ) -> Result<Self, BoxDynError> {
        let s = <String as Decode<sqlx::Postgres>>::decode(value)?;
        Email::try_from(s)
            .map_err(|e| Box::new(e) as BoxDynError)
    }
}
```

This approach lets `Decode` run validation, rejecting
invalid data at the database boundary.
