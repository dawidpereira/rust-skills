# Vertical Slice Architecture

Organize Rust web APIs **by feature**, not by technical layer. Each
feature owns its entire stack — from HTTP route to database query — in
one self-contained module. Adding a feature means adding one folder.
Deleting a feature means removing one folder.

---

## Single-Crate Project Structure

The standard starting point for an Axum API. Uses `name.rs` + `name/`
folder convention — never `mod.rs`.

```
my-app/
├── Cargo.toml
├── .env
├── migrations/
├── src/
│   ├── main.rs                    # Entry point, server bootstrap
│   ├── lib.rs                     # Re-exports, AppState
│   ├── router.rs                  # Merges all feature routers
│   ├── features.rs                # pub mod users; pub mod orders; ...
│   ├── features/
│   │   ├── users.rs               # pub mod handler; ... + pub fn router()
│   │   ├── users/
│   │   │   ├── handler.rs
│   │   │   ├── service.rs
│   │   │   ├── model.rs
│   │   │   ├── repository.rs
│   │   │   ├── dto.rs
│   │   │   ├── routes.rs
│   │   │   └── tests.rs
│   │   ├── orders.rs              # Same shape, different feature
│   │   ├── orders/
│   │   │   └── ...
│   │   └── products.rs
│   │       ...
│   ├── shared.rs                  # pub mod db; pub mod errors; ...
│   └── shared/
│       ├── db.rs
│       ├── errors.rs
│       ├── models.rs              # Shared value objects and IDs
│       ├── auth.rs
│       ├── middleware.rs
│       └── config.rs
└── tests/
    └── api_tests.rs
```

Module root files (`features.rs`, `users.rs`, `shared.rs`) sit
NEXT TO their corresponding folder and contain `pub mod`
declarations.

---

## Anatomy of a Slice

Every slice has the same internal shape. Each file has one job.

### users.rs — Module root

```rust
// features/users.rs — sits next to features/users/
pub mod handler;
pub mod service;
pub mod model;
pub mod repository;
pub mod dto;
pub mod routes;

#[cfg(test)]
mod tests;
```

### handler.rs — HTTP boundary

Axum handlers extract request data and delegate to the service.
Handlers never contain business logic.

```rust
pub async fn create_user(
    State(svc): State<Arc<UserService>>,
    Json(body): Json<CreateUserRequest>,
) -> Result<impl IntoResponse, AppError> {
    let user = svc.create(body).await?;
    Ok((StatusCode::CREATED, Json(UserResponse::from(user))))
}

pub async fn get_user(
    State(svc): State<Arc<UserService>>,
    Path(id): Path<Uuid>,
) -> Result<impl IntoResponse, AppError> {
    let user = svc.find_by_id(id).await?;
    Ok(Json(UserResponse::from(user)))
}
```

### service.rs — Business logic

Orchestrates repository calls, enforces invariants, applies rules.
Takes owned DTOs in, returns domain models out.

```rust
pub struct UserService {
    repo: Arc<dyn UserRepository>,
}

impl UserService {
    pub fn new(repo: Arc<dyn UserRepository>) -> Self {
        Self { repo }
    }

    pub async fn create(&self, cmd: CreateUserRequest) -> Result<User> {
        let user = User::new(cmd.email, cmd.name);
        self.repo.insert(&user).await
    }

    pub async fn find_by_id(&self, id: Uuid) -> Result<User> {
        self.repo
            .find_by_id(id)
            .await?
            .ok_or_else(|| AppError::NotFound("User not found".into()))
    }
}
```

### model.rs — Domain entity

The source of truth for this feature's data. For simple CRUD
features, domain models can use standard types and public fields
directly. When a feature grows invariants or business rules,
graduate to newtypes and private fields — see DDD Slice Variant
below and the rust-ddd skill.

Domain models do not derive `sqlx::FromRow`. For simple CRUD
where the domain model matches the DB schema closely, `query_as!`
maps columns directly. When the domain model diverges from the DB
schema (newtypes, computed fields, status enums), use separate DB
row structs — see DDD Slice Variant below and the rust-ddd skill.

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct User {
    pub id: Uuid,
    pub email: String,
    pub name: String,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

impl User {
    pub fn new(email: String, name: String) -> Self {
        let now = Utc::now();
        Self {
            id: Uuid::new_v4(),
            email,
            name,
            created_at: now,
            updated_at: now,
        }
    }
}
```

### repository.rs — Database access

A trait for testability plus a concrete implementation. The trait
returns domain types, never raw database rows.

```rust
#[async_trait]
pub trait UserRepository: Send + Sync {
    async fn insert(&self, user: &User) -> Result<User>;
    async fn find_by_id(&self, id: Uuid) -> Result<Option<User>>;
    async fn list(&self, page: u32, per_page: u32) -> Result<Vec<User>>;
    async fn delete(&self, id: Uuid) -> Result<()>;
}

pub struct PgUserRepository {
    pool: PgPool,
}

#[async_trait]
impl UserRepository for PgUserRepository {
    async fn insert(&self, user: &User) -> Result<User> {
        sqlx::query_as!(User,
            r#"INSERT INTO users (id, email, name, created_at, updated_at)
               VALUES ($1, $2, $3, $4, $5)
               RETURNING *"#,
            user.id, user.email, user.name, user.created_at, user.updated_at
        )
        .fetch_one(&self.pool)
        .await
        .map_err(Into::into)
    }

    async fn find_by_id(&self, id: Uuid) -> Result<Option<User>> {
        sqlx::query_as!(User, "SELECT * FROM users WHERE id = $1", id)
            .fetch_optional(&self.pool)
            .await
            .map_err(Into::into)
    }
}
```

### dto.rs — Request/Response types

Separate from domain models. Validates input at the HTTP boundary.
Never derives `sqlx::FromRow`.

```rust
#[derive(Debug, Deserialize, Validate)]
pub struct CreateUserRequest {
    #[validate(email)]
    pub email: String,
    #[validate(length(min = 1, max = 100))]
    pub name: String,
}

#[derive(Debug, Serialize)]
pub struct UserResponse {
    pub id: Uuid,
    pub email: String,
    pub name: String,
}

impl From<User> for UserResponse {
    fn from(u: User) -> Self {
        Self { id: u.id, email: u.email, name: u.name }
    }
}
```

### routes.rs — Feature router

Returns a `Router` that the root `router.rs` merges. Each slice owns
its route definitions.

```rust
pub fn router(svc: Arc<UserService>) -> Router {
    Router::new()
        .route("/users", post(create_user).get(list_users))
        .route("/users/{id}", get(get_user).delete(delete_user))
        .with_state(svc)
}
```

### tests.rs — Feature tests

Unit tests live here. Mock the repository trait for isolation.

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[tokio::test]
    async fn create_user_returns_user() {
        // Arrange: mock repo, build service
        // Act: call service.create(...)
        // Assert: returned user has correct fields
    }

    #[tokio::test]
    async fn get_nonexistent_returns_not_found() {
        // Arrange: mock repo returns None
        // Act: call service.find_by_id(...)
        // Assert: returns AppError::NotFound
    }
}
```

---

## Root Wiring

### main.rs

```rust
#[tokio::main]
async fn main() -> anyhow::Result<()> {
    tracing_subscriber::fmt::init();

    let config = shared::config::AppConfig::from_env()?;
    let pool = shared::db::create_pool(&config.database_url).await?;
    sqlx::migrate!("./migrations").run(&pool).await?;

    let app = router::create_router(pool);
    let listener = TcpListener::bind(&config.server_addr).await?;
    tracing::info!("Listening on {}", config.server_addr);
    axum::serve(listener, app).await?;
    Ok(())
}
```

### router.rs — Dependency injection happens here

Build services, inject repositories, merge all feature routers:

```rust
pub fn create_router(pool: PgPool) -> Router {
    let user_repo = Arc::new(PgUserRepository::new(pool.clone()));
    let user_svc = Arc::new(UserService::new(user_repo));

    let order_repo = Arc::new(PgOrderRepository::new(pool.clone()));
    let order_svc = Arc::new(OrderService::new(order_repo, user_svc.clone()));

    Router::new()
        .merge(users::routes::router(user_svc))
        .merge(orders::routes::router(order_svc))
        .layer(shared::middleware::logging_layer())
}
```

---

## DDD Slice Variant

For features with rich domain logic — aggregates with invariants,
status transitions, domain events — add DDD building blocks within
the slice. The feature owns its aggregate, events, and domain errors
alongside handlers and repositories.

```
features/
├── orders.rs                    # Module root + router function
├── orders/
│   ├── models.rs                # Aggregate + entities + value objects
│   ├── events.rs                # Domain event enum
│   ├── error.rs                 # Domain error enum
│   ├── repository.rs            # Repository trait (no sqlx imports)
│   ├── create_order.rs          # Use case: handler + request/response DTO
│   ├── cancel_order.rs          # Use case
│   ├── ship_order.rs            # Use case
│   ├── get_order.rs             # Query use case
│   └── list_orders.rs           # Query use case
```

Key differences from the standard slice:

- **models.rs** replaces model.rs — contains aggregate with private
  fields, entities, and value objects. Business logic lives on the
  aggregate (`order.place()`, `order.cancel()`), not in a service.
- **events.rs** — Domain events collected by the aggregate, published
  after saving.
- **error.rs** — Feature-specific domain errors as a `thiserror` enum.
- **File-per-use-case** replaces handler.rs + service.rs — each use
  case file has its own handler function and DTOs.
- **repository.rs** defines a trait with zero sqlx imports. The
  implementation lives in `infrastructure/persistence/`.

```
infrastructure/
├── persistence.rs
├── persistence/
│   ├── pg_order_repository.rs   # impl OrderRepository + DB row structs
│   ├── pg_customer_repository.rs
│   └── db.rs                    # Pool setup
```

The module root wires submodules and defines the router:

```rust
// features/orders.rs
pub mod models;
pub mod events;
pub mod repository;
pub mod error;

pub mod create_order;
pub mod cancel_order;
pub mod ship_order;
pub mod get_order;
pub mod list_orders;

use axum::{routing::{get, post, put}, Router};

pub fn router() -> Router<crate::shared::AppState> {
    Router::new()
        .route("/orders", post(create_order::handle).get(list_orders::handle))
        .route("/orders/{id}", get(get_order::handle))
        .route("/orders/{id}/cancel", put(cancel_order::handle))
        .route("/orders/{id}/ship", put(ship_order::handle))
}
```

Use DDD slices when the domain has real invariants to enforce.
For simple CRUD, the standard slice (handler + service + model)
is sufficient. See the rust-ddd skill for complete code examples
of aggregates, value objects, events, and repository separation.

### Growth Rule

Start flat within a feature — one `models.rs` file is fine until
~300 lines. Then promote:

```
Stage 1: models.rs (~200 lines)
  Single file with aggregate + entities + value objects

Stage 2: models.rs → models/ (~300+ lines)
  models.rs        becomes: pub mod order; pub mod order_item; pub mod value_objects;
  models/
    order.rs       aggregate only
    order_item.rs  entity only
    value_objects.rs

Imports from outside the feature never change.
```

---

## CQRS Variant

For slices with complex write logic, split into commands and queries:

```
features/
├── orders.rs
├── orders/
│   ├── commands.rs
│   ├── commands/
│   │   ├── create_order.rs
│   │   │   # CreateOrderCommand + handler
│   │   └── cancel_order.rs
│   │       # ...
│   ├── queries.rs
│   ├── queries/
│   │   ├── get_order.rs
│   │   │   # query + handler + response
│   │   └── list_orders.rs
│   │       # ...
│   ├── models.rs
│   ├── repository.rs
│   └── routes.rs
```

Use CQRS when reads and writes have significantly different shapes
or performance requirements. For simple CRUD, the standard slice
structure is sufficient — don't add this complexity preemptively.

---

## Workspace Layout

When the project grows beyond ~15 features or multiple teams own
different features, graduate to a workspace with one crate per feature:

```
my-platform/
├── Cargo.toml                     # [workspace]
├── crates/
│   ├── api/                       # Binary: HTTP server
│   │   └── src/
│   │       ├── main.rs
│   │       └── router.rs
│   ├── feature-users/             # Library: one slice
│   │   └── src/
│   │       ├── lib.rs
│   │       ├── handler.rs
│   │       ├── service.rs
│   │       ├── model.rs
│   │       ├── repository.rs
│   │       └── routes.rs
│   ├── feature-orders/
│   │   └── ...
│   └── shared/
│       └── src/
│           ├── lib.rs
│           ├── db.rs
│           ├── errors.rs
│           ├── models.rs
│           └── config.rs
└── migrations/
```

Workspace Cargo.toml with dependency and lint inheritance:

```toml
[workspace]
members = ["crates/*"]
resolver = "2"

[workspace.dependencies]
axum = "0.8"
tokio = { version = "1", features = ["full"] }
sqlx = { version = "0.8", features = ["runtime-tokio", "postgres"] }
serde = { version = "1", features = ["derive"] }
uuid = { version = "1", features = ["v4", "serde"] }
anyhow = "1"
tracing = "0.1"

[workspace.lints.clippy]
correctness = "deny"
suspicious = "warn"
style = "warn"
complexity = "warn"
perf = "warn"
```

Feature crate inherits from workspace:

```toml
[package]
name = "feature-users"
version = "0.1.0"
edition = "2024"

[dependencies]
shared = { path = "../shared" }
axum.workspace = true
tokio.workspace = true
sqlx.workspace = true

[lints]
workspace = true
```

---

## Shared Infrastructure

### errors.rs

```rust
#[derive(Debug, thiserror::Error)]
pub enum AppError {
    #[error("Not found: {0}")]
    NotFound(String),
    #[error("Validation error: {0}")]
    Validation(String),
    #[error("Unauthorized")]
    Unauthorized,
    #[error(transparent)]
    Internal(#[from] anyhow::Error),
}

impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        let (status, message) = match &self {
            AppError::NotFound(msg) => (StatusCode::NOT_FOUND, msg.clone()),
            AppError::Validation(msg) => (StatusCode::BAD_REQUEST, msg.clone()),
            AppError::Unauthorized => (StatusCode::UNAUTHORIZED, "Unauthorized".into()),
            AppError::Internal(e) => {
                tracing::error!("Internal error: {e:?}");
                (StatusCode::INTERNAL_SERVER_ERROR, "Internal server error".into())
            }
        };
        (status, Json(json!({ "error": message }))).into_response()
    }
}
```

For DDD projects, `AppError` includes `From` impls for each
feature's domain errors. See rust-ddd → references/infrastructure.md.

### db.rs

```rust
pub async fn create_pool(database_url: &str) -> anyhow::Result<PgPool> {
    PgPoolOptions::new()
        .max_connections(10)
        .connect(database_url)
        .await
        .map_err(Into::into)
}
```

### config.rs

```rust
pub struct AppConfig {
    pub database_url: String,
    pub server_addr: String,
    pub jwt_secret: String,
}

impl AppConfig {
    pub fn from_env() -> anyhow::Result<Self> {
        dotenvy::dotenv().ok();
        Ok(Self {
            database_url: std::env::var("DATABASE_URL")?,
            server_addr: std::env::var("SERVER_ADDR")
                .unwrap_or_else(|_| "0.0.0.0:3000".into()),
            jwt_secret: std::env::var("JWT_SECRET")?,
        })
    }
}
```

---

## Common Dependencies

```toml
[dependencies]
axum = "0.8"
tokio = { version = "1", features = ["full"] }
tower = "0.5"
tower-http = { version = "0.6", features = ["cors", "trace"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
sqlx = { version = "0.8", features = ["runtime-tokio", "postgres", "uuid", "chrono"] }
validator = { version = "0.18", features = ["derive"] }
thiserror = "2"
anyhow = "1"
uuid = { version = "1", features = ["v4", "serde"] }
chrono = { version = "0.4", features = ["serde"] }
tracing = "0.1"
tracing-subscriber = "0.3"
dotenvy = "0.15"
async-trait = "0.1"

[dev-dependencies]
mockall = "0.13"
reqwest = { version = "0.12", features = ["json"] }
```
