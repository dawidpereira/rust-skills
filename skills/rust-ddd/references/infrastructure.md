# Infrastructure & Use Cases

Domain code defines traits. Infrastructure implements them.
Use case files orchestrate the flow. This reference covers
everything outside the domain model itself.

---

## Repository Traits

Repository traits live in the feature directory. They define
data access using only domain types — no sqlx, no diesel.

```rust
// features/orders/repository.rs

use async_trait::async_trait;
use super::models::{Order, OrderId};

#[async_trait]
pub trait OrderRepository: Send + Sync {
    async fn find_by_id(&self, id: OrderId)
        -> Result<Option<Order>, RepositoryError>;
    async fn save(&self, order: &Order)
        -> Result<(), RepositoryError>;
    async fn delete(&self, id: OrderId)
        -> Result<(), RepositoryError>;
    async fn find_by_customer(&self, customer_id: crate::shared::models::CustomerId)
        -> Result<Vec<Order>, RepositoryError>;
}

#[derive(Debug, thiserror::Error)]
pub enum RepositoryError {
    #[error("Not found: {0:?}")]
    NotFound(OrderId),
    #[error("Persistence error: {0}")]
    Technical(#[from] anyhow::Error),
}
```

The trait returns domain types (`Order`, `OrderId`), not DB
rows. Infrastructure code handles the mapping.

---

## DB Row Structs

Separate from domain models. These live in the infrastructure
implementation file, never in feature domain code.

```rust
// infrastructure/persistence/pg_order_repository.rs

#[derive(sqlx::FromRow)]
struct OrderRow {
    id: Uuid,
    customer_id: Uuid,
    status: String,
    total_cents: i64,
    currency: String,
    created_at: DateTime<Utc>,
}

#[derive(sqlx::FromRow)]
struct OrderItemRow {
    id: Uuid,
    order_id: Uuid,
    product_id: Uuid,
    product_name: String,
    quantity: i32,
    unit_price_cents: i64,
}
```

DB rows are flat structs matching column types. They never
appear in feature code.

---

## Repository Implementation

Maps between DB rows and domain aggregates via `reconstitute`.

```rust
// infrastructure/persistence/pg_order_repository.rs

use crate::features::orders::models::*;
use crate::features::orders::repository::{OrderRepository, RepositoryError};
use crate::shared::models::{CustomerId, Money, Currency, ProductId};

pub struct PgOrderRepository {
    pool: PgPool,
}

impl PgOrderRepository {
    pub fn new(pool: PgPool) -> Self { Self { pool } }
}

#[async_trait]
impl OrderRepository for PgOrderRepository {
    async fn find_by_id(&self, id: OrderId)
        -> Result<Option<Order>, RepositoryError>
    {
        let row = sqlx::query_as::<_, OrderRow>(
            "SELECT * FROM orders WHERE id = $1",
        )
        .bind(id.as_uuid())
        .fetch_optional(&self.pool)
        .await
        .map_err(|e| RepositoryError::Technical(e.into()))?;

        let Some(row) = row else { return Ok(None) };

        let item_rows = sqlx::query_as::<_, OrderItemRow>(
            "SELECT * FROM order_items WHERE order_id = $1",
        )
        .bind(id.as_uuid())
        .fetch_all(&self.pool)
        .await
        .map_err(|e| RepositoryError::Technical(e.into()))?;

        let currency = parse_currency(&row.currency)?;

        let items = item_rows
            .into_iter()
            .map(|r| {
                let qty = u32::try_from(r.quantity)
                    .map_err(|e| RepositoryError::Technical(e.into()))?;
                Ok(OrderItem::reconstitute(
                    r.id,
                    ProductId::from_uuid(r.product_id),
                    r.product_name,
                    Quantity::new(qty)
                        .map_err(|e| RepositoryError::Technical(e.into()))?,
                    Money::new(r.unit_price_cents, currency)
                        .map_err(|e| RepositoryError::Technical(e.into()))?,
                ))
            })
            .collect::<Result<Vec<_>, RepositoryError>>()?;

        let status = match row.status.as_str() {
            "draft" => OrderStatus::Draft,
            "placed" => OrderStatus::Placed,
            "paid" => OrderStatus::Paid,
            "shipped" => OrderStatus::Shipped,
            "delivered" => OrderStatus::Delivered,
            "cancelled" => OrderStatus::Cancelled,
            _ => {
                return Err(RepositoryError::Technical(
                    anyhow::anyhow!("Unknown status: {}", row.status),
                ))
            }
        };

        let total = Money::new(row.total_cents, currency)
            .map_err(|e| RepositoryError::Technical(e.into()))?;

        Ok(Some(Order::reconstitute(
            OrderId::from_uuid(row.id),
            CustomerId::from_uuid(row.customer_id),
            items,
            status,
            total,
            row.created_at,
        )))
    }

    async fn save(&self, order: &Order) -> Result<(), RepositoryError> {
        let mut tx = self.pool
            .begin()
            .await
            .map_err(|e| RepositoryError::Technical(e.into()))?;

        // UPSERT order + order_items in transaction
        // ...

        tx.commit()
            .await
            .map_err(|e| RepositoryError::Technical(e.into()))?;
        Ok(())
    }

    // ... delete, find_by_customer follow the same pattern
}

fn parse_currency(s: &str) -> Result<Currency, RepositoryError> {
    match s {
        "USD" => Ok(Currency::USD),
        "EUR" => Ok(Currency::EUR),
        "GBP" => Ok(Currency::GBP),
        _ => Err(RepositoryError::Technical(
            anyhow::anyhow!("Unknown currency: {s}"),
        )),
    }
}
```

The key pattern: query DB rows → map to domain value objects →
call `reconstitute()` on the aggregate.

---

## Use Case Files

One file per use case. Each contains a request DTO, a response
DTO, and a handler function. The handler orchestrates — it
does not contain business logic.

```rust
// features/orders/create_order.rs

use axum::{extract::State, http::StatusCode, Json, response::IntoResponse};
use serde::{Deserialize, Serialize};
use std::sync::Arc;
use uuid::Uuid;
use validator::Validate;

use super::models::{Order, OrderItem, Quantity};
use super::repository::OrderRepository;
use crate::shared::models::{CustomerId, Money, Currency, ProductId};
use crate::shared::errors::AppError;

#[derive(Debug, Deserialize, Validate)]
pub struct CreateOrderRequest {
    pub customer_id: Uuid,
    #[validate(length(min = 1))]
    pub items: Vec<OrderItemInput>,
}

#[derive(Debug, Deserialize)]
pub struct OrderItemInput {
    pub product_id: Uuid,
    pub product_name: String,
    pub quantity: u32,
    pub unit_price_cents: i64,
}

#[derive(Debug, Serialize)]
pub struct CreateOrderResponse {
    pub id: Uuid,
    pub status: String,
    pub total_cents: i64,
}

pub async fn handle(
    State(repo): State<Arc<dyn OrderRepository>>,
    Json(body): Json<CreateOrderRequest>,
) -> Result<impl IntoResponse, AppError> {
    body.validate()?;

    let items = body
        .items
        .into_iter()
        .map(|i| {
            let qty = Quantity::new(i.quantity)?;
            let price = Money::new(i.unit_price_cents, Currency::USD)?;
            Ok(OrderItem::new(ProductId::from_uuid(i.product_id), i.product_name, qty, price))
        })
        .collect::<Result<Vec<_>, _>>()?;

    let mut order = Order::create(CustomerId::from_uuid(body.customer_id), items, Currency::USD)?;
    repo.save(&order).await?;

    let _events = order.take_events();
    // event_publisher.publish_all(events).await?;

    Ok((
        StatusCode::CREATED,
        Json(CreateOrderResponse {
            id: *order.id().as_uuid(),
            status: format!("{:?}", order.status()),
            total_cents: order.total().amount_cents(),
        }),
    ))
}
```

A simpler use case with fewer dependencies:

```rust
// features/orders/cancel_order.rs

use axum::{extract::{State, Path}, http::StatusCode, response::IntoResponse};
use std::sync::Arc;
use uuid::Uuid;

use super::models::OrderId;
use super::repository::OrderRepository;
use crate::shared::errors::AppError;

pub async fn handle(
    State(repo): State<Arc<dyn OrderRepository>>,
    Path(order_id): Path<Uuid>,
) -> Result<impl IntoResponse, AppError> {
    let mut order = repo
        .find_by_id(OrderId::from_uuid(order_id))
        .await?
        .ok_or(AppError::NotFound("Order not found".into()))?;

    order.cancel()?;
    repo.save(&order).await?;

    let _events = order.take_events();

    Ok(StatusCode::NO_CONTENT)
}
```

Each handler takes only the dependencies it needs. No god
service with 10 injected traits.

### When to Extract a Command Handler Struct

Only when the same business logic must be triggered from
multiple entry points (HTTP + message queue + CLI):

```rust
pub struct CreateOrderHandler {
    repo: Arc<dyn OrderRepository>,
}

impl CreateOrderHandler {
    pub async fn handle(
        &self,
        cmd: CreateOrderCommand,
    ) -> Result<OrderId, OrderError> {
        // business logic
    }
}

// HTTP handler calls it:
pub async fn handle(
    State(h): State<Arc<CreateOrderHandler>>,
    Json(body): Json<CreateOrderRequest>,
) -> Result<impl IntoResponse, AppError> {
    let cmd = CreateOrderCommand::from(body);
    let id = h.handle(cmd).await?;
    Ok((StatusCode::CREATED, Json(json!({ "id": id }))))
}

// Queue consumer calls it:
pub async fn on_message(h: &CreateOrderHandler, msg: QueueMessage) {
    let cmd = CreateOrderCommand::from(msg);
    h.handle(cmd).await.unwrap();
}
```

For single-entry-point use cases (most web APIs), the free
function approach is simpler and sufficient.

---

## Context-Aware Use Cases

When a use case needs caller identity (multi-tenancy, audit
logging, permission checks), add `RequestContext` as the first
extractor argument and pass it to repository methods.

`RequestContext` is a plain struct in `shared/context.rs` with
no framework imports — just `uuid` and std. See
rust-architecture → references/request-context.md for the full
definition and extractor implementation.

```rust
// features/orders/repository.rs (context-aware variant)

use crate::shared::context::RequestContext;

#[async_trait]
pub trait OrderRepository: Send + Sync {
    async fn find_by_id(
        &self,
        ctx: &RequestContext,
        id: OrderId,
    ) -> Result<Option<Order>, RepositoryError>;

    async fn save(
        &self,
        ctx: &RequestContext,
        order: &Order,
    ) -> Result<(), RepositoryError>;
}
```

The handler extracts context and threads it through:

```rust
// features/orders/create_order.rs (context-aware variant)

pub async fn handle(
    ctx: RequestContext,
    State(repo): State<Arc<dyn OrderRepository>>,
    Json(body): Json<CreateOrderRequest>,
) -> Result<impl IntoResponse, AppError> {
    body.validate()?;
    let items = map_to_domain(body.items)?;
    let mut order = Order::create(
        CustomerId::from_uuid(ctx.user_id), items, Currency::USD,
    )?;

    repo.save(&ctx, &order).await?;

    let _events = order.take_events();
    Ok((StatusCode::CREATED, Json(response)))
}
```

```rust
// features/orders/cancel_order.rs (context-aware variant)

pub async fn handle(
    ctx: RequestContext,
    State(repo): State<Arc<dyn OrderRepository>>,
    Path(order_id): Path<Uuid>,
) -> Result<impl IntoResponse, AppError> {
    let mut order = repo
        .find_by_id(&ctx, OrderId::from_uuid(order_id))
        .await?
        .ok_or(AppError::NotFound("Order not found".into()))?;

    order.cancel()?;
    repo.save(&ctx, &order).await?;

    let _events = order.take_events();
    Ok(StatusCode::NO_CONTENT)
}
```

The repository implementation uses `ctx.tenant_id` for scoped
queries and `ctx.user_id` / `ctx.request_id` for audit logging.
See rust-architecture → references/request-context.md for the
full implementation with tenant scoping and audit trail.

---

## Error Mapping

Domain errors stay framework-free. The mapping to HTTP status
codes lives in `shared/errors.rs`.

```rust
// shared/errors.rs

use axum::{http::StatusCode, response::{IntoResponse, Response}, Json};
use serde_json::json;
use crate::features::orders::error::OrderError;
use crate::features::orders::repository::RepositoryError;

#[derive(Debug, thiserror::Error)]
pub enum AppError {
    #[error(transparent)]
    Order(#[from] OrderError),
    #[error(transparent)]
    Repository(#[from] RepositoryError),
    #[error("Not found: {0}")]
    NotFound(String),
    #[error("Validation: {0}")]
    Validation(String),
    #[error("Unauthorized")]
    Unauthorized,
    #[error(transparent)]
    Internal(#[from] anyhow::Error),
}

impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        let (status, message) = match &self {
            AppError::Order(OrderError::EmptyOrder)
            | AppError::Order(OrderError::ZeroQuantity) => {
                (StatusCode::BAD_REQUEST, self.to_string())
            }
            AppError::Order(OrderError::InvalidTransition(..))
            | AppError::Order(OrderError::OrderNotDraft) => {
                (StatusCode::CONFLICT, self.to_string())
            }
            AppError::Repository(RepositoryError::NotFound(_))
            | AppError::NotFound(_) => {
                (StatusCode::NOT_FOUND, self.to_string())
            }
            AppError::Validation(msg) => {
                (StatusCode::BAD_REQUEST, msg.clone())
            }
            AppError::Unauthorized => {
                (StatusCode::UNAUTHORIZED, "Unauthorized".into())
            }
            _ => {
                tracing::error!("Internal error: {self:?}");
                (
                    StatusCode::INTERNAL_SERVER_ERROR,
                    "Internal error".into(),
                )
            }
        };
        (status, Json(json!({ "error": message }))).into_response()
    }
}
```

---

## Shared Types

`shared/models.rs` contains only types used by 2+ features.
Never entities or aggregates — only value objects and IDs.

```rust
// shared/models.rs

use serde::{Serialize, Deserialize};
use uuid::Uuid;

// Type-safe IDs shared across features
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash, Serialize, Deserialize)]
pub struct OrderId(Uuid);
// ... new(), from_uuid(), as_uuid()

#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash, Serialize, Deserialize)]
pub struct CustomerId(Uuid);
// ...

#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash, Serialize, Deserialize)]
pub struct ProductId(Uuid);
// ...

// Value objects
pub struct Money { /* ... */ }
pub enum Currency { USD, EUR, GBP }
pub struct Email(String);

#[derive(Debug, thiserror::Error)]
pub enum DomainError {
    #[error("Negative amount not allowed")]
    NegativeAmount,
    #[error("Currency mismatch")]
    CurrencyMismatch,
    #[error("Invalid email: {0}")]
    InvalidEmail(String),
}
```

Rule: if only one feature uses an ID type, keep it in that
feature's `models.rs`. Move to `shared/models.rs` only when
a second feature needs it.

---

## Module Wiring

How features, infrastructure, and shared modules connect.

```rust
// lib.rs
pub mod features;
pub mod infrastructure;
pub mod shared;
```

```rust
// features.rs — sits next to features/
pub mod orders;
pub mod customers;
pub mod products;
```

```rust
// features/orders.rs — sits next to features/orders/
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

The feature's module root file (`orders.rs`) declares all
submodules and defines the router. Use `name.rs` + `name/`
convention — never `mod.rs`. See rust-architecture for the
full project layout.

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
sqlx = { version = "0.8", features = ["runtime-tokio", "postgres", "uuid", "chrono", "rust_decimal"] }
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
tokio-test = "0.4"
```
