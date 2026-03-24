# Vertical Slice Architecture

Organize Rust web APIs **by feature**, not by technical layer. Each
feature owns its entire stack — from HTTP route to database query — in
one self-contained module. Adding a feature means adding one folder.
Deleting a feature means removing one folder.

---

## Single-Crate Project Structure

The standard starting point for an Axum API:

```
my-app/
├── Cargo.toml
├── .env
├── migrations/
├── src/
│   ├── main.rs                    # Entry point, server bootstrap
│   ├── lib.rs                     # Re-exports, AppState
│   ├── router.rs                  # Merges all feature routers
│   ├── features/                  # ★ Vertical slices
│   │   ├── mod.rs
│   │   ├── users/
│   │   │   ├── mod.rs
│   │   │   ├── handler.rs
│   │   │   ├── service.rs
│   │   │   ├── model.rs
│   │   │   ├── repository.rs
│   │   │   ├── dto.rs
│   │   │   ├── routes.rs
│   │   │   └── tests.rs
│   │   ├── orders/                # Same shape, different feature
│   │   │   └── ...
│   │   └── products/
│   │       └── ...
│   ├── shared/                    # Cross-cutting concerns
│   │   ├── mod.rs
│   │   ├── db.rs
│   │   ├── errors.rs
│   │   ├── auth.rs
│   │   ├── middleware.rs
│   │   └── config.rs
│   └── domain/                    # Optional: shared value objects
│       ├── mod.rs
│       └── value_objects.rs
└── tests/
    └── api_tests.rs
```

---

## Anatomy of a Slice

Every slice has the same internal shape. Each file has one job.

### mod.rs — Visibility gate

```rust
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

The source of truth for this feature's data. Derives `sqlx::FromRow`
for database mapping but never appears in HTTP responses directly.

```rust
#[derive(Debug, Clone, Serialize, Deserialize, sqlx::FromRow)]
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
        .route("/users/:id", get(get_user).delete(delete_user))
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

## CQRS Variant

For slices with complex write logic, split into commands and queries:

```
features/orders/
├── mod.rs
├── commands/
│   ├── mod.rs
│   ├── create_order/
│   │   ├── command.rs             # CreateOrderCommand struct
│   │   ├── handler.rs             # Executes the command
│   │   └── validator.rs
│   └── cancel_order/
│       └── ...
├── queries/
│   ├── mod.rs
│   ├── get_order/
│   │   ├── query.rs
│   │   ├── handler.rs
│   │   └── response.rs
│   └── list_orders/
│       └── ...
├── model.rs
├── repository.rs
└── routes.rs
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
