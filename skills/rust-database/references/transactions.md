# Transactions

## Lifecycle

sqlx transactions follow RAII: `begin()` starts the
transaction, `commit()` finalizes it, and dropping
without commit triggers an automatic rollback.

```rust
let mut tx = pool.begin().await?;

sqlx::query!(
    "INSERT INTO orders (id, user_id, total) VALUES ($1, $2, $3)",
    order_id,
    user_id,
    total
)
.execute(&mut *tx)
.await?;

sqlx::query!(
    "UPDATE inventory SET quantity = quantity - $1 WHERE product_id = $2",
    quantity,
    product_id
)
.execute(&mut *tx)
.await?;

tx.commit().await?;
```

If any query fails, `?` propagates the error, `tx` is
dropped, and the entire transaction rolls back. No
explicit rollback call is needed.

---

## Passing Transactions Through Functions

Accept `&mut Transaction` to compose multiple
operations into a single transaction:

```rust
use sqlx::{PgConnection, Postgres, Transaction};

async fn create_order(
    conn: &mut PgConnection,
    order: &NewOrder,
) -> Result<OrderId, sqlx::Error> {
    let row = sqlx::query_scalar!(
        "INSERT INTO orders (user_id, total)
         VALUES ($1, $2) RETURNING id",
        order.user_id,
        order.total
    )
    .fetch_one(&mut *conn)
    .await?;

    Ok(OrderId(row))
}

async fn create_order_items(
    conn: &mut PgConnection,
    order_id: OrderId,
    items: &[NewOrderItem],
) -> Result<(), sqlx::Error> {
    for item in items {
        sqlx::query!(
            "INSERT INTO order_items (order_id, product_id, qty)
             VALUES ($1, $2, $3)",
            order_id.0,
            item.product_id,
            item.quantity
        )
        .execute(&mut *conn)
        .await?;
    }
    Ok(())
}
```

The caller controls the transaction boundary:

```rust
let mut tx = pool.begin().await?;
let order_id = create_order(&mut tx, &new_order).await?;
create_order_items(&mut tx, order_id, &items).await?;
tx.commit().await?;
```

### Using a trait for flexibility

Functions that accept `&mut PgConnection` work with
both a pool connection and a transaction, since
`Transaction` derefs to `PgConnection`. This lets the
same function run inside or outside a transaction:

```rust
async fn find_user(
    conn: &mut PgConnection,
    id: UserId,
) -> Result<Option<User>, sqlx::Error> {
    sqlx::query_as!(
        User,
        "SELECT id, email, name FROM users WHERE id = $1",
        id.0
    )
    .fetch_optional(&mut *conn)
    .await
}

// Works with a pool connection
let mut conn = pool.acquire().await?;
let user = find_user(&mut conn, user_id).await?;

// Works inside a transaction
let mut tx = pool.begin().await?;
let user = find_user(&mut tx, user_id).await?;
tx.commit().await?;
```

---

## Savepoints (Nested Transactions)

Use `Transaction::begin()` to create a savepoint
within an existing transaction:

```rust
let mut tx = pool.begin().await?;

create_order(&mut tx, &order).await?;

let savepoint = tx.begin().await?;
match apply_discount(&mut *savepoint, &discount).await {
    Ok(()) => savepoint.commit().await?,
    Err(e) => {
        tracing::warn!(error = %e, "Discount failed, skipping");
        // savepoint is dropped, rolls back to before
        // apply_discount — outer transaction is intact
    }
};

tx.commit().await?;
```

Savepoints let you attempt optional operations without
risking the entire transaction.

---

## Deadlock Prevention

Deadlocks occur when two transactions hold locks and
each waits for the other. Prevention strategies:

### Consistent lock ordering

Always acquire locks in the same order. If you update
`orders` then `inventory`, do so everywhere:

```rust
async fn place_order(
    tx: &mut PgConnection,
    order: &NewOrder,
) -> Result<(), Error> {
    // Always: orders first, then inventory
    sqlx::query!(
        "INSERT INTO orders (id, user_id) VALUES ($1, $2)",
        order.id,
        order.user_id
    )
    .execute(&mut *tx)
    .await?;

    sqlx::query!(
        "UPDATE inventory SET qty = qty - $1 WHERE product_id = $2",
        order.quantity,
        order.product_id
    )
    .execute(&mut *tx)
    .await?;

    Ok(())
}
```

### Minimal transaction scope

Keep transactions as short as possible. Do all
validation and computation *before* starting the
transaction:

```rust
// Validate outside the transaction
let validated = validate_order(&input)?;
let price = calculate_total(&validated);

// Transaction only covers the writes
let mut tx = pool.begin().await?;
insert_order(&mut tx, &validated, price).await?;
deduct_inventory(&mut tx, &validated.items).await?;
tx.commit().await?;
```

### Lock timeouts

Set a statement timeout to fail fast instead of
waiting indefinitely:

```rust
sqlx::query("SET LOCAL lock_timeout = '5s'")
    .execute(&mut *tx)
    .await?;
```

---

## Read-Only Transactions

Use read-only transactions when you need a consistent
snapshot across multiple reads without any writes:

```rust
sqlx::query("SET TRANSACTION READ ONLY")
    .execute(&mut *tx)
    .await?;

let orders = fetch_orders(&mut tx, user_id).await?;
let totals = fetch_totals(&mut tx, user_id).await?;
// Both queries see the same snapshot

tx.commit().await?;
```

Read-only transactions let Postgres optimize locking
and prevent accidental writes in reporting paths.

---

## Common Patterns

### Unit of work

Group all changes for a single business operation
into one transaction. The caller owns the transaction
boundary, repository functions accept a connection:

```rust
pub async fn complete_checkout(
    pool: &PgPool,
    checkout: &Checkout,
) -> Result<OrderId, Error> {
    let mut tx = pool.begin().await?;

    let order_id = orders::create(&mut tx, checkout).await?;
    payments::charge(&mut tx, checkout.payment()).await?;
    inventory::deduct(&mut tx, checkout.items()).await?;
    notifications::queue(&mut tx, order_id).await?;

    tx.commit().await?;
    Ok(order_id)
}
```

### Compensation (saga-like)

When operations span external systems that can't join
a database transaction, use compensation to undo
partial work:

```rust
let mut tx = pool.begin().await?;
let order_id = orders::create(&mut tx, &checkout).await?;
tx.commit().await?;

match payment_gateway::charge(&checkout.payment).await {
    Ok(charge_id) => {
        let mut tx = pool.begin().await?;
        orders::mark_paid(&mut tx, order_id, charge_id).await?;
        tx.commit().await?;
    }
    Err(e) => {
        let mut tx = pool.begin().await?;
        orders::cancel(&mut tx, order_id, &e.to_string()).await?;
        tx.commit().await?;
    }
}
```

Keep compensation logic explicit. Don't rely on
implicit rollback for cross-system consistency.
