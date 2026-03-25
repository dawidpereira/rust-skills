# DDD Building Blocks

All examples use a single domain: an e-commerce Order with
line items, status transitions, and domain events. This keeps
examples consistent across the skill.

---

## Value Objects

Immutable types defined by their value, not identity. Two
`Money` instances with the same amount and currency are equal.
Wrap primitives to make illegal states unrepresentable.

### Type-Safe IDs

```rust
use serde::{Serialize, Deserialize};
use uuid::Uuid;

#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash, Serialize, Deserialize)]
pub struct OrderId(Uuid);

impl OrderId {
    pub fn new() -> Self { Self(Uuid::new_v4()) }
    pub fn from_uuid(id: Uuid) -> Self { Self(id) }
    pub fn as_uuid(&self) -> &Uuid { &self.0 }
}
```

If `OrderId` is referenced by multiple features, it lives in
`shared/models.rs`. If only the orders feature uses it, it
stays in `features/orders/models.rs`.

### Validated Value Objects

```rust
#[derive(Debug, Clone, Copy, PartialEq)]
pub struct Quantity(u32);

impl Quantity {
    pub fn new(val: u32) -> Result<Self, OrderError> {
        if val == 0 {
            return Err(OrderError::ZeroQuantity);
        }
        Ok(Self(val))
    }

    pub fn value(&self) -> u32 { self.0 }
}
```

Once constructed, `Quantity` is always valid. No need to
re-check elsewhere.

### Compound Value Objects

```rust
#[derive(Debug, Clone, PartialEq)]
pub struct Money {
    amount: i64,
    currency: Currency,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum Currency { USD, EUR, GBP }

impl Money {
    pub fn new(amount: i64, currency: Currency) -> Result<Self, DomainError> {
        if amount < 0 {
            return Err(DomainError::NegativeAmount);
        }
        Ok(Self { amount, currency })
    }

    pub fn zero(currency: Currency) -> Self {
        Self { amount: 0, currency }
    }

    pub fn amount_cents(&self) -> i64 { self.amount }
    pub fn currency(&self) -> Currency { self.currency }

    pub fn add(&self, other: &Money) -> Result<Self, DomainError> {
        if self.currency != other.currency {
            return Err(DomainError::CurrencyMismatch);
        }
        Money::new(self.amount + other.amount, self.currency)
    }

    pub fn multiply(&self, factor: u32) -> Self {
        Self {
            amount: self.amount * factor as i64,
            currency: self.currency,
        }
    }
}
```

`Money` and `Currency` typically live in `shared/models.rs`
since multiple features use them.

---

## Entities

Objects with identity. Two `OrderItem` instances with the same
fields but different IDs are different entities.

```rust
use crate::shared::models::ProductId;

#[derive(Debug, Clone)]
pub struct OrderItem {
    id: Uuid,
    product_id: ProductId,
    product_name: String,
    quantity: Quantity,
    unit_price: Money,
}

impl OrderItem {
    pub fn new(
        product_id: ProductId,
        name: String,
        qty: Quantity,
        price: Money,
    ) -> Self {
        Self {
            id: Uuid::new_v4(),
            product_id,
            product_name: name,
            quantity: qty,
            unit_price: price,
        }
    }

    pub fn id(&self) -> Uuid { self.id }
    pub fn product_id(&self) -> ProductId { self.product_id }
    pub fn product_name(&self) -> &str { &self.product_name }
    pub fn quantity(&self) -> &Quantity { &self.quantity }
    pub fn unit_price(&self) -> &Money { &self.unit_price }

    pub fn line_total(&self) -> Money {
        self.unit_price.multiply(self.quantity.value())
    }

    pub fn update_quantity(&mut self, qty: Quantity) {
        self.quantity = qty;
    }

    /// Reconstitute from DB — no validation, data already trusted
    pub(crate) fn reconstitute(
        id: Uuid,
        product_id: ProductId,
        product_name: String,
        quantity: Quantity,
        unit_price: Money,
    ) -> Self {
        Self { id, product_id, product_name, quantity, unit_price }
    }
}

// Entity equality is by ID, not by all fields
impl PartialEq for OrderItem {
    fn eq(&self, other: &Self) -> bool { self.id == other.id }
}
impl Eq for OrderItem {}
```

---

## Aggregates

The consistency boundary. An aggregate owns its entities and
value objects through private fields. Only command methods can
mutate state, and they enforce all invariants.

```rust
use chrono::{DateTime, Utc};
use crate::shared::models::CustomerId;

#[derive(Debug)]
pub struct Order {
    id: OrderId,
    customer_id: CustomerId,
    items: Vec<OrderItem>,
    status: OrderStatus,
    total: Money,
    events: Vec<super::events::OrderEvent>,
    created_at: DateTime<Utc>,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum OrderStatus {
    Draft,
    Placed,
    Paid,
    Shipped,
    Delivered,
    Cancelled,
}
```

### Factory Method — The Only Way to Create

```rust
impl Order {
    pub fn create(
        customer_id: CustomerId,
        items: Vec<OrderItem>,
        currency: Currency,
    ) -> Result<Self, OrderError> {
        if items.is_empty() {
            return Err(OrderError::EmptyOrder);
        }

        let total = items
            .iter()
            .try_fold(Money::zero(currency), |acc, item| {
                acc.add(&item.line_total())
            })
            .map_err(OrderError::Domain)?;

        let mut order = Self {
            id: OrderId::new(),
            customer_id,
            items,
            status: OrderStatus::Draft,
            total,
            events: Vec::new(),
            created_at: Utc::now(),
        };

        order.events.push(super::events::OrderEvent::Created {
            order_id: order.id,
            customer_id,
            total: order.total.clone(),
        });

        Ok(order)
    }
}
```

### Command Methods — Enforce Invariants

```rust
impl Order {
    pub fn place(&mut self) -> Result<(), OrderError> {
        match self.status {
            OrderStatus::Draft => {
                self.status = OrderStatus::Placed;
                self.events.push(
                    super::events::OrderEvent::Placed { order_id: self.id },
                );
                Ok(())
            }
            _ => Err(OrderError::InvalidTransition(
                self.status,
                OrderStatus::Placed,
            )),
        }
    }

    pub fn cancel(&mut self) -> Result<(), OrderError> {
        match self.status {
            OrderStatus::Draft | OrderStatus::Placed => {
                self.status = OrderStatus::Cancelled;
                self.events.push(
                    super::events::OrderEvent::Cancelled { order_id: self.id },
                );
                Ok(())
            }
            _ => Err(OrderError::CannotCancel(self.status)),
        }
    }

    pub fn add_item(&mut self, item: OrderItem) -> Result<(), OrderError> {
        if self.status != OrderStatus::Draft {
            return Err(OrderError::OrderNotDraft);
        }
        self.total = self.total
            .add(&item.line_total())
            .map_err(OrderError::Domain)?;
        self.events.push(super::events::OrderEvent::ItemAdded {
            order_id: self.id,
            product_id: item.product_id(),
            quantity: item.quantity().value(),
        });
        self.items.push(item);
        Ok(())
    }
}
```

### Queries — Immutable Access

```rust
impl Order {
    pub fn id(&self) -> OrderId { self.id }
    pub fn customer_id(&self) -> CustomerId { self.customer_id }
    pub fn status(&self) -> &OrderStatus { &self.status }
    pub fn total(&self) -> &Money { &self.total }
    pub fn items(&self) -> &[OrderItem] { &self.items }
    pub fn created_at(&self) -> DateTime<Utc> { self.created_at }
}
```

### Event Collection

```rust
impl Order {
    pub fn take_events(&mut self) -> Vec<super::events::OrderEvent> {
        std::mem::take(&mut self.events)
    }
}
```

The use case handler calls `take_events()` after saving and
publishes them. The aggregate never knows about the event bus.

### Reconstitute — DB Loading Without Validation

```rust
impl Order {
    pub(crate) fn reconstitute(
        id: OrderId,
        customer_id: CustomerId,
        items: Vec<OrderItem>,
        status: OrderStatus,
        total: Money,
        created_at: DateTime<Utc>,
    ) -> Self {
        Self {
            id,
            customer_id,
            items,
            status,
            total,
            events: Vec::new(),
            created_at,
        }
    }
}
```

`pub(crate)` restricts access to the current crate. Only the
repository implementation calls this. No validation runs, no
events are generated — the data is already trusted.

---

## Domain Events

Enums with data variants. The compiler forces exhaustive
handling via `match`.

```rust
// features/orders/events.rs

use super::models::OrderId;
use crate::shared::models::{CustomerId, Money, ProductId};

#[derive(Debug, Clone)]
pub enum OrderEvent {
    Created {
        order_id: OrderId,
        customer_id: CustomerId,
        total: Money,
    },
    Placed {
        order_id: OrderId,
    },
    ItemAdded {
        order_id: OrderId,
        product_id: ProductId,
        quantity: u32,
    },
    Cancelled {
        order_id: OrderId,
    },
    Shipped {
        order_id: OrderId,
        tracking_number: String,
    },
}

impl OrderEvent {
    pub fn order_id(&self) -> OrderId {
        match self {
            Self::Created { order_id, .. }
            | Self::Placed { order_id }
            | Self::ItemAdded { order_id, .. }
            | Self::Cancelled { order_id }
            | Self::Shipped { order_id, .. } => *order_id,
        }
    }

    pub fn event_type(&self) -> &'static str {
        match self {
            Self::Created { .. } => "order.created",
            Self::Placed { .. } => "order.placed",
            Self::ItemAdded { .. } => "order.item_added",
            Self::Cancelled { .. } => "order.cancelled",
            Self::Shipped { .. } => "order.shipped",
        }
    }
}
```

For event transport and cross-feature subscription, see
rust-architecture → references/cross-slice.md (Strategy 4:
Domain Events).

---

## Domain Errors

One `thiserror` enum per feature. Each variant carries the
data needed for a useful error message.

```rust
// features/orders/error.rs

use super::models::OrderStatus;
use crate::shared::models::DomainError;

#[derive(Debug, thiserror::Error)]
pub enum OrderError {
    #[error("Order cannot be empty")]
    EmptyOrder,
    #[error("Order is not in draft status")]
    OrderNotDraft,
    #[error("Quantity cannot be zero")]
    ZeroQuantity,
    #[error("Invalid status transition from {0:?} to {1:?}")]
    InvalidTransition(OrderStatus, OrderStatus),
    #[error("Cannot cancel order in status {0:?}")]
    CannotCancel(OrderStatus),
    #[error("Domain error: {0}")]
    Domain(#[from] DomainError),
}
```

Domain errors never mention HTTP status codes or framework
types. Mapping to HTTP happens in `shared/errors.rs` — see
references/infrastructure.md.
