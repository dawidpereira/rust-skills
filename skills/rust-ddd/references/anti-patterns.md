# DDD Anti-Patterns

Common mistakes when applying DDD in Rust, with fixes.

---

## 1. Anemic Domain Model

Aggregate has public fields and no behavior. All logic lives
in a service struct that reads and writes fields directly.

```rust
// Bad: aggregate is just a data bag
pub struct Order {
    pub id: Uuid,
    pub status: String,
    pub items: Vec<OrderItem>,
    pub total: f64,
}

// Service does everything
impl OrderService {
    pub fn cancel(&self, order: &mut Order) -> Result<()> {
        if order.status == "placed" {
            order.status = "cancelled".to_string();
        }
        Ok(())
    }
}
```

```rust
// Good: aggregate owns its behavior
pub struct Order {
    id: OrderId,          // private
    status: OrderStatus,  // private
    items: Vec<OrderItem>,
    total: Money,
    events: Vec<OrderEvent>,
}

impl Order {
    pub fn cancel(&mut self) -> Result<(), OrderError> {
        match self.status {
            OrderStatus::Draft | OrderStatus::Placed => {
                self.status = OrderStatus::Cancelled;
                self.events.push(OrderEvent::Cancelled { order_id: self.id });
                Ok(())
            }
            _ => Err(OrderError::CannotCancel(self.status)),
        }
    }
}
```

The aggregate enforces its own rules. Outside code cannot set
`status` to an invalid value because the field is private.

---

## 2. God Service

One service struct with 15+ methods and 10 injected traits.
Every method pulls in ALL dependencies even though each uses
only 2-3.

```rust
// Bad: one struct does everything
pub struct OrderService {
    order_repo: Arc<dyn OrderRepository>,
    customer_repo: Arc<dyn CustomerRepository>,
    product_repo: Arc<dyn ProductRepository>,
    payment_gateway: Arc<dyn PaymentGateway>,
    email_sender: Arc<dyn EmailSender>,
    // ... 5 more
}
```

Split into file-per-use-case. Each handler takes only its
deps via Axum State extractors:

```rust
// features/orders/create_order.rs — takes only what it needs
pub async fn handle(
    State(repo): State<Arc<dyn OrderRepository>>,
    Json(body): Json<CreateOrderRequest>,
) -> Result<impl IntoResponse, AppError> { /* ... */ }
```

---

## 3. Leaking DB Types into Domain

Domain models derive `sqlx::FromRow` or contain DB-specific
types. The domain becomes coupled to the persistence layer.

```rust
// Bad: domain model IS the DB row
#[derive(sqlx::FromRow)]
pub struct Order {
    pub id: Uuid,
    pub status: String,     // stringly typed
    pub total_cents: i64,   // leaks storage format
}
```

Keep domain models pure. Create separate DB row structs in
infrastructure and map via `reconstitute()`:

```rust
// infrastructure/persistence/pg_order_repository.rs
#[derive(sqlx::FromRow)]
struct OrderRow { /* DB columns */ }

// Map: OrderRow → domain value objects → Order::reconstitute()
```

---

## 4. Forcing MediatR / Mediator

Porting the C# MediatR pattern to Rust with complex trait
machinery, generic dispatchers, and type-erased handlers.

Rust does not need MediatR because:
- Axum extractors ARE dependency injection
- The Router IS the dispatcher
- Direct function calls are traceable (IDE go-to-definition)
- No runtime service locator to configure

Call use case functions directly. The router wires them.

---

## 5. Business Logic in Handlers

The Axum handler validates input, queries the database,
applies business rules, calculates totals, and saves — all
in one function.

Handlers should orchestrate, not decide:

```rust
// Handler orchestrates
pub async fn handle(State(repo): ..., Json(body): ...) -> ... {
    body.validate()?;                         // validate input
    let items = map_to_domain(body.items)?;   // map DTOs
    let mut order = Order::create(...)?;      // aggregate decides
    repo.save(&order).await?;                 // persist
    Ok(response)                              // respond
}
```

If the handler contains `if/else` branches about domain
rules, those rules belong on the aggregate.

---

## 6. Using String for Everything

```rust
// Bad: primitive obsession
pub fn create_order(
    customer_id: String,    // could be anything
    email: String,          // no validation
    amount: f64,            // what currency? cents?
) { /* ... */ }
```

Use newtype value objects with validated constructors:

```rust
pub fn create_order(
    customer_id: CustomerId,  // type-safe
    email: Email,             // validated at construction
    amount: Money,            // currency-aware
) { /* ... */ }
```

The compiler catches mix-ups between `CustomerId` and
`ProductId` at compile time.

---

## 7. Skipping Reconstitute

Using the factory constructor to load from DB. This re-runs
validation on already-valid data and generates spurious events.

```rust
// Bad: factory method for DB loading
let order = Order::create(row.customer_id, items, currency)?;
// ↑ validates again, generates OrderCreated event from DB load!

// Good: reconstitute skips validation and events
let order = Order::reconstitute(id, customer_id, items, status, total, created_at);
```

`reconstitute()` is `pub(crate)` so only infrastructure code
can call it.

---

## 8. Domain Files Importing Framework Crates

```rust
// Bad: domain model depends on web framework
// features/orders/models.rs
use axum::extract::State;     // why?
use sqlx::FromRow;            // DB concern
```

Domain files (`models.rs`, `events.rs`, `repository.rs`,
`error.rs`) import only:
- Standard library types
- `serde` for serialization traits
- `chrono`, `uuid` for domain value types
- `thiserror` for error derives
- Other domain modules within the same feature
- `shared/models.rs` for cross-feature value objects

Framework imports belong in use case files and infrastructure.

---

## 9. Premature File Splitting

Creating `aggregates/`, `value_objects/`, `events/` folders
when the feature has 3 files with 100 lines each.

Start flat:
```
features/orders/
├── models.rs       # aggregate + entities + VOs (~200 lines)
├── events.rs       # one enum
├── error.rs        # one enum
└── repository.rs   # one trait
```

Promote to folder only when a file exceeds ~300 lines or has
2+ distinct concerns. When you promote, imports from outside
the feature don't change.

---

## 10. Shared Business Logic in `shared/`

Putting domain rules in `shared/` because two features need
similar logic.

`shared/models.rs` is for value objects and IDs only — things
with no behavior beyond construction and arithmetic. If two
features need the same business rule, either:
- It is a value object method (put it on `Money`, `Email`, etc.)
- It signals a missing bounded context (extract a new feature)
- The features should duplicate the logic (often the right call)

---

## 11. Separate Top-Level `domain/` Folder

```
# Bad: domain scattered away from features
src/
├── domain/
│   ├── order.rs
│   └── customer.rs
├── features/
│   ├── orders/
│   └── customers/
```

Domain models live INSIDE their feature. Each feature is a
self-contained bounded context:

```
# Good: feature owns its domain
src/
├── features/
│   ├── orders/
│   │   ├── models.rs    # Order aggregate lives here
│   │   └── ...
│   └── customers/
│       ├── models.rs    # Customer aggregate lives here
│       └── ...
```

---

## 12. Using `mod.rs` Files

```
# Bad: mod.rs convention
features/
├── orders/
│   ├── mod.rs          # which mod.rs? hard to tell in tabs
│   └── models.rs
```

Use `name.rs` + `name/` folder convention (Rust 2018+):

```
# Good: name.rs convention
features/
├── orders.rs           # module root, clear in editor tabs
├── orders/
│   └── models.rs
```

`orders.rs` sits NEXT TO its `orders/` folder and contains
`pub mod` declarations. See rust-architecture for the full
convention.

---

## 13. Treating Enums as C-Style Labels

Using flat enums with no data and scattering `if status == X`
checks across services and handlers. This ignores Rust's enum
as a sum type and recreates the C#/Java pattern.

```rust
// Bad: flat labels, external checks everywhere
#[derive(PartialEq)]
pub enum OrderStatus { Draft, Placed, Shipped }

// In service.rs
if order.status() == &OrderStatus::Draft {
    // do draft things
}

// In handler.rs (same check, duplicated)
if order.status() == &OrderStatus::Draft {
    // do other draft things
}
```

```rust
// Good: data-carrying variants with behavior on the enum
pub enum OrderState {
    Draft { items: Vec<OrderItem> },
    Placed { items: Vec<OrderItem>, placed_at: DateTime<Utc> },
    Shipped { tracking: String, shipped_at: DateTime<Utc> },
}

impl OrderState {
    pub fn can_cancel(&self) -> bool {
        matches!(self, Self::Draft { .. } | Self::Placed { .. })
    }

    pub fn place(self) -> Result<Self, OrderError> {
        match self {
            Self::Draft { items } => Ok(Self::Placed {
                items,
                placed_at: Utc::now(),
            }),
            _ => Err(OrderError::InvalidTransition),
        }
    }
}
```

```rust
// Advanced: typestate — invalid transitions are compile errors
impl DraftOrder {
    pub fn place(self) -> Result<PlacedOrder, OrderError> { /* ... */ }
}
// PlacedOrder has no `place()` — calling it won't compile
```

The rule: if your enum has no variant data and your codebase
is full of `== Status::X` comparisons, you're using a Rust
enum like a C enum. Put the data and behavior on the variants.
