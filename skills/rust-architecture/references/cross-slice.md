# Cross-Slice Communication

When a feature needs data or behavior owned by another feature.
The goal: minimal coupling while keeping slices independently
deletable. Start with the lightest strategy that works.

---

## Strategy 1: Public API Trait

Each slice exposes a small `api.rs` with a trait. Other slices
depend on the trait, not on internals. This is the primary
strategy for cross-slice business logic.

```rust
// features/users/api.rs — the ONLY thing other slices see

#[async_trait]
pub trait UserApi: Send + Sync {
    async fn get_user_summary(&self, id: Uuid) -> Result<UserSummary>;
}

#[derive(Clone, Serialize)]
pub struct UserSummary {
    pub id: Uuid,
    pub name: String,
    pub email: String,
}

impl UserApi for UserService {
    async fn get_user_summary(&self, id: Uuid) -> Result<UserSummary> {
        let user = self.repo.find_by_id(id).await?;
        Ok(UserSummary { id: user.id, name: user.name, email: user.email })
    }
}
```

The consuming slice depends on the trait, injected via constructor:

```rust
// features/orders/service.rs

pub struct OrderService {
    repo: Arc<dyn OrderRepository>,
    user_api: Arc<dyn UserApi>,
    product_api: Arc<dyn ProductApi>,
}

impl OrderService {
    pub async fn get_order_details(&self, id: Uuid) -> Result<OrderDetails> {
        let order = self.repo.find_by_id(id).await?;
        let user = self.user_api.get_user_summary(order.user_id).await?;
        let products = self.product_api
            .get_products_by_ids(&order.product_ids).await?;
        Ok(OrderDetails { order, user, products })
    }
}
```

**When to use:** One slice needs to call behavior in another —
validate, transform, or check live state. Trait-based DI makes
mocking straightforward in tests.

---

## Strategy 2: Read Models (SQL JOINs)

The consuming slice queries across tables directly and maps to
its own flattened struct. Zero cross-slice Rust imports.

```rust
// features/orders/read_model.rs

#[derive(Debug, Serialize, sqlx::FromRow)]
pub struct OrderWithDetails {
    pub order_id: Uuid,
    pub status: String,
    pub created_at: DateTime<Utc>,
    pub user_name: String,
    pub user_email: String,
    pub product_name: String,
    pub product_price: Decimal,
    pub quantity: i32,
}
```

```rust
// features/orders/repository.rs

async fn get_order_with_details(&self, id: Uuid) -> Result<OrderWithDetails> {
    sqlx::query_as!(OrderWithDetails, r#"
        SELECT
            o.id as order_id, o.status, o.created_at,
            u.name as user_name, u.email as user_email,
            p.name as product_name, p.price as product_price,
            oi.quantity
        FROM orders o
        JOIN users u ON o.user_id = u.id
        JOIN order_items oi ON oi.order_id = o.id
        JOIN products p ON oi.product_id = p.id
        WHERE o.id = $1
    "#, id)
    .fetch_one(&self.pool)
    .await
    .map_err(Into::into)
}
```

**When to use:** One slice needs to display data from another
(query endpoints). Most performant — single query, no N+1. Couples
at the database schema level, not at the Rust module level.

---

## Strategy 3: Shared Domain Types

Put type-safe IDs and value objects in a shared `domain/` module.
Both slices reference the same types without depending on each
other's internals.

```rust
// domain/ids.rs

#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash, Serialize, Deserialize, sqlx::Type)]
#[sqlx(transparent)]
pub struct UserId(pub Uuid);

#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash, Serialize, Deserialize, sqlx::Type)]
#[sqlx(transparent)]
pub struct ProductId(pub Uuid);

#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash, Serialize, Deserialize, sqlx::Type)]
#[sqlx(transparent)]
pub struct OrderId(pub Uuid);
```

```rust
// features/orders/model.rs — uses shared IDs

use crate::domain::ids::{OrderId, UserId, ProductId};

pub struct Order {
    pub id: OrderId,
    pub user_id: UserId,
    pub items: Vec<OrderItem>,
    pub status: OrderStatus,
}
```

**Rule:** Only IDs and value objects go in `domain/`. Never full
entities or business logic. If two slices need slightly different
versions of a type, duplicate it rather than forcing a shared
abstraction.

**When to use:** Almost always. Type-safe IDs prevent mixing up
a `UserId` with a `ProductId` at compile time.

---

## Strategy 4: Domain Events

Slices publish events instead of calling each other. Best for
side effects that should not block the main operation.

```rust
// shared/events.rs

#[derive(Debug, Clone)]
pub enum DomainEvent {
    OrderCreated {
        order_id: OrderId,
        user_id: UserId,
        product_ids: Vec<ProductId>,
    },
    OrderCancelled { order_id: OrderId },
    UserRegistered { user_id: UserId },
}

#[derive(Clone)]
pub struct EventBus {
    sender: broadcast::Sender<DomainEvent>,
}

impl EventBus {
    pub fn new() -> Self {
        let (sender, _) = broadcast::channel(256);
        Self { sender }
    }

    pub fn publish(&self, event: DomainEvent) {
        let _ = self.sender.send(event);
    }

    pub fn subscribe(&self) -> broadcast::Receiver<DomainEvent> {
        self.sender.subscribe()
    }
}
```

Publisher:

```rust
// features/orders/service.rs

pub async fn create_order(&self, cmd: CreateOrder) -> Result<Order> {
    let order = self.repo.insert(&cmd).await?;
    self.events.publish(DomainEvent::OrderCreated {
        order_id: order.id,
        user_id: cmd.user_id,
        product_ids: cmd.product_ids.clone(),
    });
    Ok(order)
}
```

Subscriber:

```rust
// features/products/listener.rs

pub async fn handle_order_events(
    mut rx: broadcast::Receiver<DomainEvent>,
    product_svc: Arc<ProductService>,
) {
    while let Ok(event) = rx.recv().await {
        if let DomainEvent::OrderCreated { product_ids, .. } = event {
            product_svc.decrease_stock(&product_ids).await
                .unwrap_or_else(|e| tracing::error!("{e}"));
        }
    }
}
```

**When to use:** Async side effects — send email, update
inventory, emit analytics. Eventual consistency is acceptable.

---

## Decision Guide

| Need | Strategy | Coupling |
|------|----------|----------|
| Call behavior in another slice | Public API Trait | Low (trait boundary) |
| Display data from another slice | Read Model / JOIN | Low (DB schema only) |
| Share a type definition (IDs) | Shared Domain Types | Medium (shared crate) |
| React to something happening | Domain Events | Lowest (fire and forget) |

Most projects combine strategies 1 + 2 + 3: shared IDs everywhere,
read models for query endpoints, and public API traits for cross-slice
business logic.

**Escalation path:** Start with read models for data display. If you
need business logic across slices, add a public API trait. If you need
async reactions, add domain events. Don't reach for events when a
simple trait call would work.

---

## Anti-Patterns

**Direct internal imports.** Importing another slice's `service.rs`
or `repository.rs` directly. Only `api.rs` exports should cross slice
boundaries.

**Circular dependencies.** If slice A depends on slice B and B depends
on A, extract the shared concern into `shared/` or introduce an event.

**God shared module.** Putting business logic in `shared/` because
two slices need it. Shared should contain only infrastructure (db,
errors, config) and domain types (IDs, value objects). Business logic
belongs in a slice.

**Over-engineering with events.** Using domain events for synchronous
data reads. If orders needs to check whether a user exists before
creating an order, that is a synchronous API trait call, not an event.
