# Axum Middleware and Extractors

Axum's request handling builds on Tower's `Service` trait.
Extractors pull data from requests at compile time.
Middleware wraps services to add cross-cutting behavior.

---

## Axum Extractors

### Ordering Rule

Body-consuming extractors (`Json`, `Form`, `Bytes`,
`String`, `Request`) must be the LAST argument. The
request body can only be consumed once.

```rust
async fn create(
    State(db): State<PgPool>,  // non-body: any order
    Path(id): Path<Uuid>,      // non-body: any order
    Json(body): Json<CreateRequest>,  // body: MUST be last
) -> Result<impl IntoResponse, AppError> { ... }
```

### Built-in Extractors

| Extractor | Source | Body? |
|-----------|--------|-------|
| `Path<T>` | URL path parameters | No |
| `Query<T>` | Query string `?key=val` | No |
| `Json<T>` | JSON request body | Yes |
| `Form<T>` | URL-encoded body | Yes |
| `Bytes` | Raw body bytes | Yes |
| `String` | Body as UTF-8 string | Yes |
| `State<T>` | App state via `.with_state()` | No |
| `Extension<T>` | Request extensions | No |
| `HeaderMap` | All request headers | No |
| `ConnectInfo<T>` | Client address | No |
| `Request` | Full `http::Request` | Yes |

### Custom Extractors

Implement `FromRequestParts` when you only need
headers, path, query, or extensions (no body access):

```rust
pub struct AuthUser { pub user_id: Uuid, pub role: Role }

#[async_trait]
impl<S> FromRequestParts<S> for AuthUser
where S: Send + Sync,
{
    type Rejection = AppError;

    async fn from_request_parts(
        parts: &mut Parts, _state: &S,
    ) -> Result<Self, Self::Rejection> {
        let token = parts.headers
            .get(AUTHORIZATION)
            .and_then(|v| v.to_str().ok())
            .and_then(|v| v.strip_prefix("Bearer "))
            .ok_or(AppError::Unauthorized)?;
        let claims = decode_jwt(token)
            .map_err(|_| AppError::Unauthorized)?;
        Ok(AuthUser {
            user_id: claims.sub,
            role: claims.role,
        })
    }
}

// Usage — just add to any handler signature
async fn get_profile(
    auth: AuthUser, State(db): State<PgPool>,
) -> Result<Json<Profile>, AppError> { ... }
```

Implement `FromRequest` when you need body access (e.g.,
validated JSON). Same pattern but receives the full
`Request` instead of `Parts`, and consumes the body:

```rust
#[async_trait]
impl<S> FromRequest<S> for ValidatedJson<T>
where T: DeserializeOwned + Validate, S: Send + Sync,
{
    type Rejection = AppError;
    async fn from_request(
        req: Request, state: &S,
    ) -> Result<Self, Self::Rejection> {
        let Json(value) = Json::<T>::from_request(req, state)
            .await.map_err(|e| AppError::Validation(e.to_string()))?;
        value.validate().map_err(|e| AppError::Validation(e.to_string()))?;
        Ok(ValidatedJson(value))
    }
}
```

### Rejection Handling

When extraction fails, Axum returns the extractor's
`Rejection` type. Control error responses by setting
`type Rejection = AppError;` in your extractor impl and
implementing `IntoResponse` for `AppError`. This maps
extraction failures to your standard error format.
See rust-errors and the vertical-slices reference for
the `AppError` `IntoResponse` implementation.

---

## Tower Middleware

### The Layer Concept

A `Layer` wraps a `Service` to add behavior. Axum uses
Tower's middleware system — every middleware is a Layer
that produces a new Service wrapping the inner one.

### `from_fn` — Simplest Custom Middleware

For most cases, `axum::middleware::from_fn` is enough:

```rust
use axum::middleware::{self, Next};

async fn require_auth(
    request: Request, next: Next,
) -> Result<Response, AppError> {
    let token = request.headers()
        .get(AUTHORIZATION)
        .and_then(|v| v.to_str().ok())
        .ok_or(AppError::Unauthorized)?;
    validate_token(token)?;
    Ok(next.run(request).await)
}

// With state access — State extractor before Request
async fn rate_limit(
    State(limiter): State<Arc<RateLimiter>>,
    request: Request, next: Next,
) -> Result<Response, AppError> {
    limiter.check_request(&request)?;
    Ok(next.run(request).await)
}
```

### Middleware Ordering

Layers wrap like an onion — outermost layer runs first
on the request and last on the response:

```rust
Router::new()
    .route("/api/items", get(list_items))
    .layer(CompressionLayer::new())       // 4th: compress
    .layer(middleware::from_fn(auth))      // 3rd: auth
    .layer(TraceLayer::new_for_http())     // 2nd: tracing
    .layer(CorsLayer::permissive())        // 1st: CORS
```

Request flows: CORS → tracing → auth → compression → handler.
Response flows back in reverse.

**Rule of thumb:** CORS and tracing outermost, auth in
the middle, compression and body transforms innermost.

### Writing a Tower Layer from Scratch

When `from_fn` is insufficient (response body
transformation, backpressure), implement two structs:
a **Layer** (creates the wrapper) and a **Service**
(does the work).

```rust
#[derive(Clone)]
pub struct MyLayer;

impl<S> Layer<S> for MyLayer {
    type Service = MyService<S>;
    fn layer(&self, inner: S) -> Self::Service {
        MyService { inner }
    }
}

#[derive(Clone)]
pub struct MyService<S> { inner: S }

impl<S, B> Service<Request<B>> for MyService<S>
where
    S: Service<Request<B>, Response = Response>
        + Clone + Send + 'static,
    S::Future: Send + 'static,
    B: Send + 'static,
{
    type Response = S::Response;
    type Error = S::Error;
    type Future = Pin<Box<dyn Future<
        Output = Result<Response, S::Error>
    > + Send>>;

    fn poll_ready(
        &mut self, cx: &mut Context<'_>,
    ) -> Poll<Result<(), Self::Error>> {
        self.inner.poll_ready(cx)
    }

    fn call(&mut self, req: Request<B>) -> Self::Future {
        let future = self.inner.call(req);
        Box::pin(async move {
            let resp = future.await?;
            // transform request/response here
            Ok(resp)
        })
    }
}
```

Prefer `from_fn` unless you need `poll_ready` control
or response body transformation.

---

## State Management

### `State<T>` with `FromRef`

Use `State<T>` for compile-time checked shared data.
`FromRef` lets handlers extract substates:

```rust
#[derive(Clone)]
pub struct AppState {
    pub db: PgPool,
    pub config: Arc<AppConfig>,
}

impl FromRef<AppState> for PgPool {
    fn from_ref(s: &AppState) -> Self { s.db.clone() }
}

impl FromRef<AppState> for Arc<AppConfig> {
    fn from_ref(s: &AppState) -> Self { s.config.clone() }
}

// Handlers extract only what they need
async fn list_users(
    State(db): State<PgPool>,
) -> Result<Json<Vec<User>>, AppError> { ... }
```

### Why `State` over `Extension`

`Extension<T>` is runtime-checked — panics if the type
is missing. `State<T>` is compile-time checked — if the
state type doesn't implement `FromRef`, the code won't
compile. Always prefer `State`.

---

## Authentication Patterns

### Auth via Custom Extractor

Extract the authenticated user as a handler argument.
If auth fails, the handler never runs. Define `AuthUser`
as a `FromRequestParts` extractor (see Custom Extractors
above), then add it to any handler that needs auth:

```rust
async fn create_order(
    auth: AuthUser,
    State(db): State<PgPool>,
    Json(body): Json<CreateOrderRequest>,
) -> Result<Json<Order>, AppError> { ... }
```

### Route-Level vs Router-Level Middleware

Protect specific route groups with `.route_layer()`:

```rust
let public = Router::new()
    .route("/health", get(health))
    .route("/login", post(login));

let protected = Router::new()
    .route("/orders", post(create_order).get(list_orders))
    .route("/profile", get(profile))
    .route_layer(middleware::from_fn(require_auth));

let app = Router::new()
    .merge(public)
    .merge(protected)
    .with_state(state);
```

---

## Common Middleware Stack

| Concern | Crate | Layer |
|---------|-------|-------|
| CORS | `tower_http::cors` | `CorsLayer::new().allow_origin(Any).allow_methods([...])` |
| Compression | `tower_http::compression` | `CompressionLayer::new()` |
| Body size limit | `tower_http::limit` | `RequestBodyLimitLayer::new(10 * 1024 * 1024)` |
| Timeout | `tower::timeout` | `TimeoutLayer::new(Duration::from_secs(30))` |
| Tracing | `tower_http::trace` | `TraceLayer::new_for_http()` |

### Request ID Propagation

Use a `from_fn` middleware: read `x-request-id` header
or generate a UUID, create an `info_span!` with the ID,
instrument the `next.run(request)` call, and copy the
ID to the response header.

### Full Stack Assembly

```rust
let app = Router::new()
    .merge(public_routes())
    .merge(protected_routes())
    .layer(CompressionLayer::new())         // 6: compress
    .layer(middleware::from_fn(require_auth)) // 5: auth
    .layer(RequestBodyLimitLayer::new(5 << 20)) // 4: limit
    .layer(TimeoutLayer::new(Duration::from_secs(30))) // 3
    .layer(TraceLayer::new_for_http())       // 2: trace
    .layer(cors)                             // 1: CORS
    .with_state(state);
```

---

## Anti-Patterns

**Business logic in middleware.** Middleware is for
cross-cutting concerns. If it grows beyond ~20 lines
of logic, move it to a service.

**Consuming the body twice.** `Json<T>` in middleware
AND handler fails. If middleware needs the body, use
`Bytes`, clone, rebuild with `Request::from_parts`.

**Wrong middleware order.** Auth after the handler does
nothing. Tracing innermost misses early rejections.
Verify the onion order matches intended execution.

**`Extension` for required state.** Panics at runtime
if missing. Use `State<T>` with `FromRef` instead —
fails at compile time. Reserve `Extension` for optional
per-request data injected by middleware.

**Overly broad auth middleware.** Applying auth to the
entire router blocks health checks. Use `.route_layer()`
on protected groups, not `.layer()` on the root.
