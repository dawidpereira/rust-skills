# Request Context Propagation

Per-request data — who is calling, which tenant, correlation
ID — needs to flow from the HTTP boundary through services
and into repositories. This reference covers the pattern.

---

## The Problem

The `AuthUser` extractor (see web-middleware.md) stops at the
handler. The repository has no idea who is making the request.
This blocks:

- **Multi-tenancy** — tenant-scoped queries (`WHERE tenant_id = $1`)
- **Audit logging** — recording who changed what
- **Caller-scoped reads** — filtering by user role or permissions

The fix: a single `RequestContext` struct extracted once at the
boundary and threaded through every layer that needs it.

---

## RequestContext Struct

Framework-free. Lives in `shared/` so repository traits and
infrastructure can reference it without importing axum.

```rust
// shared/context.rs

use uuid::Uuid;

#[derive(Debug, Clone)]
pub struct RequestContext {
    pub user_id: Uuid,
    pub tenant_id: Uuid,
    pub request_id: Uuid,
    pub role: Role,
}

#[derive(Debug, Clone, PartialEq, Eq)]
pub enum Role {
    User,
    Admin,
}
```

Only `uuid` and std imports. No axum, no sqlx, no serde.

---

## Extractor

The `FromRequestParts` impl lives in a separate file that
IS allowed to import axum. This is the only place where
`RequestContext` touches the web framework.

```rust
// shared/extractors.rs

use axum::extract::FromRequestParts;
use http::{header::AUTHORIZATION, request::Parts};
use uuid::Uuid;

use super::context::RequestContext;
use super::errors::AppError;

#[async_trait::async_trait]
impl<S> FromRequestParts<S> for RequestContext
where
    S: Send + Sync,
{
    type Rejection = AppError;

    async fn from_request_parts(
        parts: &mut Parts,
        _state: &S,
    ) -> Result<Self, Self::Rejection> {
        let token = parts
            .headers
            .get(AUTHORIZATION)
            .and_then(|v| v.to_str().ok())
            .and_then(|v| v.strip_prefix("Bearer "))
            .ok_or(AppError::Unauthorized)?;

        let claims = decode_jwt(token)
            .map_err(|_| AppError::Unauthorized)?;

        let request_id = parts
            .headers
            .get("x-request-id")
            .and_then(|v| v.to_str().ok())
            .and_then(|v| Uuid::parse_str(v).ok())
            .unwrap_or_else(Uuid::new_v4);

        Ok(RequestContext {
            user_id: claims.sub,
            tenant_id: claims.tenant_id,
            request_id,
            role: claims.role,
        })
    }
}
```

If you only need auth identity without tenant or correlation,
use the simpler `AuthUser` extractor from web-middleware.md.

---

## Handler: Receiving and Passing Context

The handler extracts `RequestContext` and passes a reference
to every service or repository call that needs it.

```rust
// features/orders/create_order.rs

pub async fn handle(
    ctx: RequestContext,
    State(repo): State<Arc<dyn OrderRepository>>,
    Json(body): Json<CreateOrderRequest>,
) -> Result<impl IntoResponse, AppError> {
    body.validate()?;

    let items = map_to_domain(body.items)?;
    let mut order = Order::create(
        CustomerId::from_uuid(ctx.user_id),
        items,
        Currency::USD,
    )?;

    repo.save(&ctx, &order).await?;

    let _events = order.take_events();
    Ok((StatusCode::CREATED, Json(response)))
}
```

`ctx` is the first extractor argument. `Json<T>` must still
be last (it consumes the body).

---

## Repository Traits with Context

Add `ctx: &RequestContext` as the first parameter on every
method that touches tenant-scoped data or needs audit info.

```rust
// features/orders/repository.rs

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

The trait imports `RequestContext` from `shared/context.rs` —
no framework dependency. Domain files stay clean.

---

## Repository Implementation: Tenant Scoping + Audit

```rust
// infrastructure/persistence/pg_order_repository.rs

#[async_trait]
impl OrderRepository for PgOrderRepository {
    async fn find_by_id(
        &self,
        ctx: &RequestContext,
        id: OrderId,
    ) -> Result<Option<Order>, RepositoryError> {
        let row = sqlx::query_as::<_, OrderRow>(
            "SELECT * FROM orders WHERE id = $1 AND tenant_id = $2",
        )
        .bind(id.as_uuid())
        .bind(ctx.tenant_id)
        .fetch_optional(&self.pool)
        .await
        .map_err(|e| RepositoryError::Technical(e.into()))?;

        // ... map to domain via reconstitute()
    }

    async fn save(
        &self,
        ctx: &RequestContext,
        order: &Order,
    ) -> Result<(), RepositoryError> {
        let mut tx = self.pool.begin().await
            .map_err(|e| RepositoryError::Technical(e.into()))?;

        sqlx::query(
            "INSERT INTO orders (id, tenant_id, customer_id, status, total_cents, currency)
             VALUES ($1, $2, $3, $4, $5, $6)
             ON CONFLICT (id) DO UPDATE SET status = $4, total_cents = $5",
        )
        .bind(order.id().as_uuid())
        .bind(ctx.tenant_id)
        .bind(order.customer_id().as_uuid())
        .bind(order.status().as_str())
        .bind(order.total().amount_cents())
        .bind(order.total().currency().as_str())
        .execute(&mut *tx)
        .await
        .map_err(|e| RepositoryError::Technical(e.into()))?;

        sqlx::query(
            "INSERT INTO audit_log (entity_type, entity_id, action, actor_id, request_id)
             VALUES ($1, $2, $3, $4, $5)",
        )
        .bind("order")
        .bind(order.id().as_uuid())
        .bind("save")
        .bind(ctx.user_id)
        .bind(ctx.request_id)
        .execute(&mut *tx)
        .await
        .map_err(|e| RepositoryError::Technical(e.into()))?;

        tx.commit().await
            .map_err(|e| RepositoryError::Technical(e.into()))?;
        Ok(())
    }
}
```

Every read is tenant-scoped. Every write leaves an audit trail
with the actor and request ID.

---

## When You Don't Need Full Context

| Situation | Use | Why |
|-----------|-----|-----|
| Auth only, single-tenant | `AuthUser` extractor | Simpler, no tenant/correlation overhead |
| Correlation ID for logs only | Tracing spans | Automatic propagation, no code changes |
| Multi-tenant, audit, or caller-scoped queries | `RequestContext` | Structured data flows through all layers |

Start with `AuthUser`. Graduate to `RequestContext` when you
add multi-tenancy or audit requirements.

---

## Connection to Tracing

`RequestContext` carries structured data for business logic
(tenant scoping, audit). Tracing spans carry the same IDs for
observability. They complement each other:

```rust
pub async fn handle(
    ctx: RequestContext,
    State(repo): State<Arc<dyn OrderRepository>>,
    Json(body): Json<CreateOrderRequest>,
) -> Result<impl IntoResponse, AppError> {
    let span = info_span!("create_order",
        request_id = %ctx.request_id,
        user_id = %ctx.user_id,
        tenant_id = %ctx.tenant_id,
    );
    async {
        body.validate()?;
        let items = map_to_domain(body.items)?;
        let mut order = Order::create(
            CustomerId::from_uuid(ctx.user_id), items, Currency::USD,
        )?;
        repo.save(&ctx, &order).await?;
        Ok((StatusCode::CREATED, Json(response)))
    }
    .instrument(span)
    .await
}
```

The tracing middleware can also create a root span with the
request ID. The handler span nests inside it — same ID,
different scopes.

---

## Anti-Patterns

**Storing context in `AppState`.** `AppState` is shared across
ALL requests. Per-request data (user, tenant, correlation ID)
does not belong there.

**Using `Extension<RequestContext>`.** Runtime-checked. Panics
if the middleware that inserts it is missing or runs in the
wrong order. Use a `FromRequestParts` extractor — extraction
failure returns a proper error response.

**Importing axum in `RequestContext`.** The struct must stay
framework-free so repository traits can import it without
pulling in the web framework. Only the `FromRequestParts`
impl (in `shared/extractors.rs`) imports axum.

**Extracting context from tracing spans at runtime.** Tracing
spans are for observability. Pulling structured data back out
at runtime couples business logic to the tracing subscriber
configuration and is fragile across span boundaries.

**Creating a new repository per request.** Repositories are
`Arc`-shared and long-lived. Injecting tenant context at
construction time would mean a new instance per request,
defeating connection pool sharing. Pass context as a method
parameter instead. See rust-ddd → anti-patterns § 14.
