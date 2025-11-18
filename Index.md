# SPRINT 0: FOUNDATION (WEEKS 1-2)

---

## WEEK 1: Docker Setup + Stripe Integration

### 📋 ENTREGABLES

**Objetivo:** Docker funcionando + Stripe integrado + Primer código Rust corriendo

**Arquitectura:**
```
Docker Compose
├── PostgreSQL (payments table)
├── Redis
├── Kafka
├── Jaeger
├── Prometheus
└── Rust App (Axum server)
```

**Deliverables:**
1. `docker-compose.yml` funcional (1 comando = todo arriba)
2. Rust service con Axum (puerto 3000)
3. Stripe webhook handler + signature verification
4. Payment table en PostgreSQL
5. Idempotency implementada
6. OpenTelemetry traces en Jaeger

**Estructura de archivos:**
```
marketflow/
├── docker-compose.yml
├── .env
├── src/
│   ├── main.rs                    # Axum server setup
│   ├── payment/
│   │   ├── mod.rs
│   │   ├── models.rs              # Payment struct
│   │   ├── repository.rs          # PaymentRepository trait + SQLx impl
│   │   └── service.rs             # PaymentService (idempotency logic)
│   ├── stripe/
│   │   ├── mod.rs
│   │   └── webhook.rs             # Webhook handler + verification
│   └── observability.rs           # OpenTelemetry setup
├── migrations/
│   └── 001_create_payments.sql
└── tests/
    └── integration_test.rs
```

**Database Schema:**
```sql
CREATE TABLE payments (
    id UUID PRIMARY KEY,
    stripe_payment_intent_id VARCHAR UNIQUE NOT NULL,
    idempotency_key VARCHAR UNIQUE,
    amount DECIMAL(19, 4) NOT NULL,
    status VARCHAR NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_payments_idempotency ON payments(idempotency_key);
```

**API Endpoints:**
```
POST /payments              → Create payment (idempotent)
POST /webhooks/stripe       → Stripe webhook handler
GET  /payments/:id          → Get payment status
GET  /health                → Health check
```

**Tests requeridos:**
- [ ] Webhook signature verification (success/fail)
- [ ] Idempotency (mismo key = mismo payment)
- [ ] Payment creation (happy path)
- [ ] Database connection

**Commits esperados:**
1. `feat(infra): Docker Compose + Axum server setup`
2. `feat(stripe): Payment struct + database connection`
3. `feat(stripe): Webhook verification`
4. `feat(stripe): Idempotency implementation`
5. `feat(week1): Stripe + observability working`

**Métricas de éxito:**
- `docker-compose up -d` → todos los servicios ✅
- `cargo run` → server en puerto 3000
- Webhook test → signature válida ✅
- Jaeger UI → traces visibles
- Tests → 100% pass

---

## WEEK 2: Double-Entry Ledger + Reconciliation

### 📋 ENTREGABLES

**Objetivo:** Sistema contable doble entrada + reconciliación automática

**Conceptos financieros:**
```
Debit = Money OUT (disminuye balance)
Credit = Money IN (aumenta balance)

Regla: Total Debits = Total Credits (SIEMPRE)
```

**Deliverables:**
1. `LedgerEntry` domain model
2. `Direction` enum (Debit/Credit)
3. Reconciliation engine
4. Balance verification
5. Tests (balanced/unbalanced scenarios)

**Estructura de archivos:**
```
src/
├── ledger/
│   ├── mod.rs
│   ├── models.rs              # LedgerEntry, Direction enum
│   ├── repository.rs          # LedgerRepository trait + SQLx impl
│   └── reconciliation.rs      # Verify balanced, calculate totals
└── tests/
    └── ledger_test.rs
```

**Database Schema:**
```sql
CREATE TABLE ledger_entries (
    id UUID PRIMARY KEY,
    user_id UUID NOT NULL,
    direction VARCHAR NOT NULL CHECK (direction IN ('Debit', 'Credit')),
    amount DECIMAL(19, 4) NOT NULL,
    reference_id UUID,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_ledger_user ON ledger_entries(user_id);
CREATE INDEX idx_ledger_direction ON ledger_entries(direction);
```

**Core Logic:**
```rust
// src/ledger/models.rs
#[derive(Debug, Clone, Copy)]
pub enum Direction {
    Debit,
    Credit,
}

pub struct LedgerEntry {
    pub id: Uuid,
    pub user_id: Uuid,
    pub direction: Direction,
    pub amount: Decimal,
    pub reference_id: Option<Uuid>,
}

// src/ledger/reconciliation.rs
pub async fn verify_balanced(pool: &PgPool) -> Result<bool> {
    // SELECT SUM(CASE direction = 'Debit' ...) - SUM(...)
    // Return: debits == credits
}

pub async fn get_user_balance(pool: &PgPool, user_id: Uuid) -> Result<Decimal> {
    // Credits - Debits = Balance
}
```

**Tests requeridos:**
- [ ] Ledger balanced (debits = credits)
- [ ] Ledger unbalanced (debits ≠ credits)
- [ ] User balance calculation
- [ ] Direction enum pattern matching

**Commits esperados:**
1. `feat(ledger): LedgerEntry domain model`
2. `feat(ledger): Direction enum + pattern matching`
3. `feat(ledger): Reconciliation engine`
4. `feat(ledger): Balance verification`
5. `feat(week2): Ledger + tests complete`

**Métricas de éxito:**
- Ledger always balanced ✅
- User balance = Credits - Debits ✅
- Tests → 100% pass
- No floating point errors (Decimal used)

---

**FIN SPRINT 0**

---

# SPRINT 1: E-COMMERCE CORE (WEEKS 3-5)

---

## WEEK 3: User Service (Auth + JWT)

### 📋 ENTREGABLES

**Objetivo:** Sistema de autenticación completo con registro, login y JWT

**Arquitectura:**
```
User Service
├── User Domain Model
├── Password Hashing (Argon2)
├── JWT Generation/Verification
├── Auth Middleware (Axum)
└── PostgreSQL (users table)
```

**Deliverables:**
1. User domain model + repository pattern
2. Password hashing con Argon2
3. JWT token generation/verification
4. REST endpoints (register, login)
5. Auth middleware para rutas protegidas
6. Tests (auth flow completo)

**Estructura de archivos:**
```
src/
├── user/
│   ├── mod.rs
│   ├── models.rs              # User struct
│   ├── repository.rs          # UserRepository trait + SQLx impl
│   └── service.rs             # UserService (business logic)
├── auth/
│   ├── mod.rs
│   ├── jwt.rs                 # Token generation/verification
│   ├── password.rs            # Argon2 hashing
│   └── middleware.rs          # AuthUser extractor
└── tests/
    └── auth_test.rs
```

**Database Schema:**
```sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR UNIQUE NOT NULL,
    name VARCHAR NOT NULL,
    password_hash VARCHAR NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_users_email ON users(email);
```

**API Endpoints:**
```
POST /auth/register    → { email, name, password } → { user_id, token }
POST /auth/login       → { email, password } → { user_id, token }
GET  /auth/me          → (requires JWT) → { user }
```

**Core Logic:**
```rust
// src/auth/password.rs
pub fn hash_password(password: &str) -> Result<String>;
pub fn verify_password(password: &str, hash: &str) -> Result<()>;

// src/auth/jwt.rs
pub fn generate_token(user_id: &str) -> Result<String>;
pub fn verify_token(token: &str) -> Result<Claims>;

// src/auth/middleware.rs
pub struct AuthUser {
    pub user_id: Uuid,
}
// Extractor que verifica JWT en cada request
```

**Tests requeridos:**
- [ ] Register user (success)
- [ ] Register duplicate email (fail)
- [ ] Login valid credentials (success)
- [ ] Login invalid password (fail)
- [ ] JWT verification (valid/expired)
- [ ] Protected endpoint without token (401)
- [ ] Protected endpoint with valid token (200)

**Commits esperados:**
1. `feat(user): Domain model + repository trait`
2. `feat(auth): Password hashing (Argon2)`
3. `feat(auth): JWT token management`
4. `feat(auth): REST endpoints (register/login)`
5. `feat(auth): Middleware + protected routes`
6. `feat(week3): User service complete`

**Métricas de éxito:**
- Register → user creado ✅
- Login → token válido ✅
- Protected route → 401 sin token, 200 con token ✅
- Password nunca en plaintext ✅
- Tests → 85%+ coverage

---

## WEEK 4: Product Service (Search + Caching)

### 📋 ENTREGABLES

**Objetivo:** Catálogo de productos con búsqueda full-text y caché Redis

**Arquitectura:**
```
Product Service
├── Product Domain Model
├── PostgreSQL (products table + full-text search)
├── Redis (cache layer)
└── Search API
```

**Deliverables:**
1. Product domain model + repository
2. Full-text search en PostgreSQL (tsvector)
3. Cache-aside pattern con Redis
4. REST endpoints (CRUD + search)
5. Cache metrics (hit rate)
6. Tests (search + cache)

**Estructura de archivos:**
```
src/
├── product/
│   ├── mod.rs
│   ├── models.rs              # Product struct
│   ├── repository.rs          # ProductRepository trait + SQLx impl
│   ├── cache.rs               # Redis cache layer
│   └── service.rs             # ProductService (cache-aside)
└── tests/
    └── product_test.rs
```

**Database Schema:**
```sql
CREATE TABLE products (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR NOT NULL,
    description TEXT,
    price DECIMAL(19, 4) NOT NULL,
    stock INT NOT NULL DEFAULT 0,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

-- Full-text search index
CREATE INDEX idx_products_search ON products 
USING GIN (to_tsvector('english', name || ' ' || description));
```

**API Endpoints:**
```
GET    /products?query=laptop&limit=10  → Search products
GET    /products/:id                    → Get product (cached)
POST   /products                        → Create product (admin)
PUT    /products/:id                    → Update product (admin)
DELETE /products/:id                    → Delete product (admin)
```

**Core Logic:**
```rust
// src/product/repository.rs
pub async fn search_products(
    pool: &PgPool,
    query: &str,
    limit: i64,
) -> Result<Vec<Product>>;

// src/product/cache.rs
pub async fn get_product_cached(
    id: Uuid,
    pool: &PgPool,
    redis: &redis::Connection,
) -> Result<Product>;
// 1. Try Redis
// 2. Fallback to PostgreSQL
// 3. Update Redis (TTL 1h)
```

**Tests requeridos:**
- [ ] Search "laptop" → results found
- [ ] Search empty query → all products
- [ ] Get product (cache miss) → DB hit + Redis update
- [ ] Get product (cache hit) → Redis only
- [ ] Create product → cache invalidated
- [ ] Update product → cache invalidated

**Commits esperados:**
1. `feat(product): Domain model + repository`
2. `feat(product): Full-text search (PostgreSQL)`
3. `feat(product): Redis cache layer`
4. `feat(product): REST endpoints (CRUD)`
5. `feat(product): Cache metrics + tests`
6. `feat(week4): Product service complete`

**Métricas de éxito:**
- Search latency: <50ms (cached), <200ms (DB) ✅
- Cache hit rate: >70% ✅
- Full-text search working ✅
- Tests → 80%+ coverage

---

## WEEK 5: Cart Service (Redis + PostgreSQL)

### 📋 ENTREGABLES

**Objetivo:** Carrito de compras con persistencia dual (Redis + PostgreSQL)

**Arquitectura:**
```
Cart Service
├── Redis (active carts, TTL 24h)
├── PostgreSQL (persistence backup)
└── Cart API
```

**Deliverables:**
1. Cart domain model
2. Redis para carts activos (fast access)
3. PostgreSQL para persistencia (backup)
4. REST endpoints (add/remove/view)
5. TTL management (24h expiration)
6. Tests (cart operations)

**Estructura de archivos:**
```
src/
├── cart/
│   ├── mod.rs
│   ├── models.rs              # Cart, CartItem structs
│   ├── repository.rs          # CartRepository trait
│   ├── redis_repo.rs          # Redis implementation
│   ├── postgres_repo.rs       # PostgreSQL backup
│   └── service.rs             # CartService (dual storage)
└── tests/
    └── cart_test.rs
```

**Database Schema:**
```sql
CREATE TABLE carts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE cart_items (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    cart_id UUID NOT NULL REFERENCES carts(id) ON DELETE CASCADE,
    product_id UUID NOT NULL,
    quantity INT NOT NULL,
    price DECIMAL(19, 4) NOT NULL,
    UNIQUE(cart_id, product_id)
);

CREATE INDEX idx_carts_user ON carts(user_id);
```

**API Endpoints:**
```
POST   /cart/items              → Add item (auth required)
DELETE /cart/items/:product_id  → Remove item (auth required)
PATCH  /cart/items/:product_id  → Update quantity (auth required)
GET    /cart                    → View cart (auth required)
DELETE /cart                    → Clear cart (auth required)
```

**Core Logic:**
```rust
// src/cart/models.rs
pub struct Cart {
    pub id: Uuid,
    pub user_id: Uuid,
    pub items: Vec<CartItem>,
}

pub struct CartItem {
    pub product_id: Uuid,
    pub quantity: i32,
    pub price: Decimal,
}

// src/cart/service.rs
pub async fn add_item(
    user_id: Uuid,
    product_id: Uuid,
    quantity: i32,
) -> Result<Cart>;
// 1. Update Redis (TTL 24h)
// 2. Persist to PostgreSQL (backup)
```

**Tests requeridos:**
- [ ] Add item → cart updated (Redis + PG)
- [ ] Add same item → quantity increased
- [ ] Remove item → item deleted
- [ ] Update quantity → quantity changed
- [ ] Get cart → items retrieved
- [ ] TTL expiration → cart removed from Redis
- [ ] Redis down → fallback to PostgreSQL

**Commits esperados:**
1. `feat(cart): Domain models`
2. `feat(cart): Redis repository`
3. `feat(cart): PostgreSQL repository`
4. `feat(cart): Dual storage service`
5. `feat(cart): REST endpoints`
6. `feat(week5): Cart service complete`

**Métricas de éxito:**
- Cart read latency: <10ms (Redis) ✅
- PostgreSQL sync: <100ms ✅
- TTL working (24h) ✅
- Tests → 85%+ coverage

---

**FIN SPRINT 1**

---

# SPRINT 2: ORDERS & PAYMENT (WEEKS 6-9)

---

## WEEK 6: Order Service

### 📋 ENTREGABLES

**Objetivo:** Sistema de órdenes con state machine y event sourcing

**Arquitectura:**
```
Order Service
├── Order Domain Model (state machine)
├── Event Sourcing (audit trail)
├── PostgreSQL (orders + events)
└── Order API
```

**Deliverables:**
1. Order domain model con state machine
2. OrderStatus enum (states + transitions)
3. Event sourcing para auditoría
4. Order creation flow
5. State transition logic
6. Tests (state machine transitions)

**Estructura de archivos:**
```
src/
├── order/
│   ├── mod.rs
│   ├── models.rs              # Order, OrderStatus, OrderItem
│   ├── repository.rs          # OrderRepository trait + SQLx impl
│   ├── service.rs             # OrderService (business logic)
│   └── events.rs              # Event sourcing logic
└── tests/
    └── order_test.rs
```

**Database Schema:**
```sql
CREATE TABLE orders (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL,
    total_amount DECIMAL(19, 4) NOT NULL,
    status VARCHAR NOT NULL CHECK (status IN ('pending', 'confirmed', 'paid', 'shipped', 'delivered', 'cancelled')),
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE order_items (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id UUID NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
    product_id UUID NOT NULL,
    quantity INT NOT NULL,
    price DECIMAL(19, 4) NOT NULL,
    UNIQUE(order_id, product_id)
);

CREATE TABLE order_events (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id UUID NOT NULL REFERENCES orders(id),
    event_type VARCHAR NOT NULL,
    from_status VARCHAR,
    to_status VARCHAR NOT NULL,
    metadata JSONB,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_orders_user ON orders(user_id);
CREATE INDEX idx_orders_status ON orders(status);
CREATE INDEX idx_events_order ON order_events(order_id);
```

**API Endpoints:**
```
POST   /orders              → Create order (auth required)
GET    /orders              → List user orders (auth required)
GET    /orders/:id          → Get order details (auth required)
PATCH  /orders/:id/status   → Update order status (auth required)
DELETE /orders/:id          → Cancel order (auth required)
```

**Core Logic:**
```rust
// src/order/models.rs
#[derive(Debug, Clone, Copy, Serialize, Deserialize, sqlx::Type)]
#[sqlx(type_name = "varchar")]
pub enum OrderStatus {
    Pending,
    Confirmed,
    Paid,
    Shipped,
    Delivered,
    Cancelled,
}

impl OrderStatus {
    pub fn can_transition_to(&self, next: OrderStatus) -> bool {
        match (self, next) {
            (OrderStatus::Pending, OrderStatus::Confirmed) => true,
            (OrderStatus::Confirmed, OrderStatus::Paid) => true,
            (OrderStatus::Paid, OrderStatus::Shipped) => true,
            (OrderStatus::Shipped, OrderStatus::Delivered) => true,
            (OrderStatus::Pending, OrderStatus::Cancelled) => true,
            (OrderStatus::Confirmed, OrderStatus::Cancelled) => true,
            _ => false,
        }
    }
}

pub struct Order {
    pub id: Uuid,
    pub user_id: Uuid,
    pub items: Vec<OrderItem>,
    pub total_amount: Decimal,
    pub status: OrderStatus,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

pub struct OrderItem {
    pub product_id: Uuid,
    pub quantity: i32,
    pub price: Decimal,
}

// src/order/events.rs
pub struct OrderEvent {
    pub id: Uuid,
    pub order_id: Uuid,
    pub event_type: String,
    pub from_status: Option<OrderStatus>,
    pub to_status: OrderStatus,
    pub metadata: serde_json::Value,
    pub created_at: DateTime<Utc>,
}

pub async fn record_event(
    pool: &PgPool,
    order_id: Uuid,
    event_type: &str,
    from_status: Option<OrderStatus>,
    to_status: OrderStatus,
) -> Result<()> {
    sqlx::query(
        "INSERT INTO order_events (id, order_id, event_type, from_status, to_status, metadata)
         VALUES ($1, $2, $3, $4, $5, '{}'::jsonb)"
    )
    .bind(Uuid::new_v4())
    .bind(order_id)
    .bind(event_type)
    .bind(from_status)
    .bind(to_status)
    .execute(pool)
    .await?;
    Ok(())
}
```

**Tests requeridos:**
- [ ] Create order from cart
- [ ] State transition (pending → confirmed)
- [ ] Invalid state transition rejected
- [ ] Cancel order
- [ ] Event sourcing (audit trail)
- [ ] Calculate total amount
- [ ] List user orders

**Commits esperados:**
1. `feat(order): Domain models + state machine`
2. `feat(order): OrderRepository + SQLx impl`
3. `feat(order): Event sourcing implementation`
4. `feat(order): State transition logic`
5. `feat(order): REST endpoints`
6. `feat(week6): Order service complete`

**Métricas de éxito:**
- State machine transitions working ✅
- Event sourcing capturing all changes ✅
- Invalid transitions rejected ✅
- Tests → 85%+ coverage

---

## WEEK 7: Webhook Management

### 📋 ENTREGABLES

**Objetivo:** Sistema robusto de webhooks con retry y DLQ

**Arquitectura:**
```
Webhook Management
├── Retry Logic (exponential backoff)
├── Dead Letter Queue (DLQ)
├── Idempotency Keys
└── PostgreSQL (webhook_events table)
```

**Deliverables:**
1. Webhook event model
2. Retry logic con exponential backoff
3. Dead Letter Queue
4. Idempotency enforcement
5. Webhook status tracking
6. Tests (retry scenarios)

**Estructura de archivos:**
```
src/
├── webhook/
│   ├── mod.rs
│   ├── models.rs              # WebhookEvent
│   ├── repository.rs          # WebhookRepository
│   ├── retry.rs               # Retry logic + backoff
│   └── handler.rs             # Webhook processing
└── tests/
    └── webhook_test.rs
```

**Database Schema:**
```sql
CREATE TABLE webhook_events (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_id VARCHAR UNIQUE NOT NULL,
    event_type VARCHAR NOT NULL,
    payload JSONB NOT NULL,
    status VARCHAR NOT NULL CHECK (status IN ('pending', 'processing', 'completed', 'failed', 'dlq')),
    retry_count INT NOT NULL DEFAULT 0,
    max_retries INT NOT NULL DEFAULT 3,
    next_retry_at TIMESTAMP,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE webhook_dlq (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    webhook_event_id UUID NOT NULL REFERENCES webhook_events(id),
    error_message TEXT NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_webhooks_status ON webhook_events(status);
CREATE INDEX idx_webhooks_retry ON webhook_events(next_retry_at) WHERE status = 'pending';
```

**Core Logic:**
```rust
// src/webhook/models.rs
#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum WebhookStatus {
    Pending,
    Processing,
    Completed,
    Failed,
    DeadLetter,
}

pub struct WebhookEvent {
    pub id: Uuid,
    pub event_id: String,
    pub event_type: String,
    pub payload: serde_json::Value,
    pub status: WebhookStatus,
    pub retry_count: i32,
    pub max_retries: i32,
    pub next_retry_at: Option<DateTime<Utc>>,
}

// src/webhook/retry.rs
pub fn calculate_backoff(retry_count: i32) -> Duration {
    let base_delay = Duration::from_secs(2);
    let max_delay = Duration::from_secs(60);
    
    let delay = base_delay * 2_u32.pow(retry_count as u32);
    delay.min(max_delay)
}

pub async fn retry_webhook(
    pool: &PgPool,
    webhook_id: Uuid,
) -> Result<()> {
    let webhook = get_webhook(pool, webhook_id).await?;
    
    if webhook.retry_count >= webhook.max_retries {
        move_to_dlq(pool, webhook_id).await?;
        return Ok(());
    }
    
    let backoff = calculate_backoff(webhook.retry_count);
    let next_retry = Utc::now() + backoff;
    
    sqlx::query(
        "UPDATE webhook_events 
         SET retry_count = retry_count + 1, 
             next_retry_at = $1,
             status = 'pending'
         WHERE id = $2"
    )
    .bind(next_retry)
    .bind(webhook_id)
    .execute(pool)
    .await?;
    
    Ok(())
}

pub async fn move_to_dlq(
    pool: &PgPool,
    webhook_id: Uuid,
) -> Result<()> {
    let mut tx = pool.begin().await?;
    
    sqlx::query(
        "UPDATE webhook_events SET status = 'dlq' WHERE id = $1"
    )
    .bind(webhook_id)
    .execute(&mut *tx)
    .await?;
    
    sqlx::query(
        "INSERT INTO webhook_dlq (id, webhook_event_id, error_message)
         VALUES ($1, $2, $3)"
    )
    .bind(Uuid::new_v4())
    .bind(webhook_id)
    .bind("Max retries exceeded")
    .execute(&mut *tx)
    .await?;
    
    tx.commit().await?;
    Ok(())
}
```

**Tests requeridos:**
- [ ] Process webhook (success)
- [ ] Retry on failure (exponential backoff)
- [ ] Max retries → DLQ
- [ ] Idempotency (duplicate event_id rejected)
- [ ] Backoff calculation

**Commits esperados:**
1. `feat(webhook): Domain models`
2. `feat(webhook): Retry logic + backoff`
3. `feat(webhook): DLQ implementation`
4. `feat(webhook): Idempotency enforcement`
5. `feat(week7): Webhook management complete`

**Métricas de éxito:**
- Webhooks retried automatically ✅
- Exponential backoff working ✅
- DLQ captures failed webhooks ✅
- No duplicate processing ✅
- Tests → 90%+ coverage

---

## WEEK 8: Security Hardening

### 📋 ENTREGABLES

**Objetivo:** Security layer completo (rate limiting, validation, secrets)

**Arquitectura:**
```
Security Layer
├── Rate Limiting (governor)
├── Input Validation (validator)
├── CORS + Security Headers
└── Secrets Management
```

**Deliverables:**
1. Rate limiting middleware
2. Input validation
3. CORS configuration
4. Security headers
5. Secrets management (env vars)
6. Tests (rate limit scenarios)

**Estructura de archivos:**
```
src/
├── security/
│   ├── mod.rs
│   ├── rate_limit.rs          # Rate limiting middleware
│   ├── validation.rs          # Input validation
│   └── headers.rs             # CORS + security headers
└── tests/
    └── security_test.rs
```

**Core Logic:**
```rust
// src/security/rate_limit.rs
use governor::{Quota, RateLimiter};
use std::num::NonZeroU32;

pub struct RateLimitLayer {
    limiter: RateLimiter<String, DefaultKeyedStateStore<String>>,
}

impl RateLimitLayer {
    pub fn new(requests_per_minute: u32) -> Self {
        let quota = Quota::per_minute(NonZeroU32::new(requests_per_minute).unwrap());
        Self {
            limiter: RateLimiter::keyed(quota),
        }
    }
    
    pub async fn check(&self, key: &str) -> Result<(), RateLimitError> {
        self.limiter.check_key(&key.to_string())
            .map_err(|_| RateLimitError::TooManyRequests)?;
        Ok(())
    }
}

// src/security/validation.rs
use validator::Validate;

#[derive(Deserialize, Validate)]
pub struct CreateUserRequest {
    #[validate(email)]
    pub email: String,
    
    #[validate(length(min = 3, max = 50))]
    pub name: String,
    
    #[validate(length(min = 8))]
    pub password: String,
}

pub async fn create_user_handler(
    Json(req): Json<CreateUserRequest>,
) -> Result<Json<User>, ValidationError> {
    req.validate()?;
    // Process validated input
}

// src/security/headers.rs
use tower_http::cors::{CorsLayer, Any};

pub fn cors_layer() -> CorsLayer {
    CorsLayer::new()
        .allow_origin(Any)
        .allow_methods([Method::GET, Method::POST, Method::PUT, Method::DELETE])
        .allow_headers([AUTHORIZATION, CONTENT_TYPE])
}

pub fn security_headers() -> HeaderMap {
    let mut headers = HeaderMap::new();
    headers.insert("X-Content-Type-Options", "nosniff".parse().unwrap());
    headers.insert("X-Frame-Options", "DENY".parse().unwrap());
    headers.insert("X-XSS-Protection", "1; mode=block".parse().unwrap());
    headers
}
```

**Tests requeridos:**
- [ ] Rate limit (under limit → 200)
- [ ] Rate limit (over limit → 429)
- [ ] Invalid email rejected
- [ ] Short password rejected
- [ ] CORS headers present
- [ ] Security headers present

**Commits esperados:**
1. `feat(security): Rate limiting middleware`
2. `feat(security): Input validation`
3. `feat(security): CORS configuration`
4. `feat(security): Security headers`
5. `feat(week8): Security hardening complete`

**Métricas de éxito:**
- Rate limiting working ✅
- Invalid input rejected ✅
- CORS configured ✅
- Security headers set ✅
- Tests → 85%+ coverage

---

## WEEK 9: Resilience Patterns

### 📋 ENTREGABLES

**Objetivo:** Resilience completo (circuit breaker, timeouts, fallbacks)

**Arquitectura:**
```
Resilience Layer
├── Circuit Breaker (states: Closed, Open, HalfOpen)
├── Timeouts
├── Retry with Backoff
└── Graceful Degradation
```

**Deliverables:**
1. Circuit breaker implementation
2. Timeout management
3. Retry logic con exponential backoff
4. Fallback strategies
5. Health checks
6. Tests (failure scenarios)

**Estructura de archivos:**
```
src/
├── resilience/
│   ├── mod.rs
│   ├── circuit_breaker.rs     # Circuit breaker pattern
│   ├── timeout.rs             # Timeout wrapper
│   └── retry.rs               # Retry logic
└── tests/
    └── resilience_test.rs
```

**Core Logic:**
```rust
// src/resilience/circuit_breaker.rs
use std::sync::Arc;
use tokio::sync::Mutex;
use std::time::{Duration, Instant};

#[derive(Debug, Clone, Copy, PartialEq)]
pub enum CircuitState {
    Closed,
    Open,
    HalfOpen,
}

pub struct CircuitBreaker {
    state: Arc<Mutex<CircuitState>>,
    failure_threshold: u32,
    failure_count: Arc<Mutex<u32>>,
    last_failure_time: Arc<Mutex<Option<Instant>>>,
    timeout: Duration,
}

impl CircuitBreaker {
    pub fn new(failure_threshold: u32, timeout: Duration) -> Self {
        Self {
            state: Arc::new(Mutex::new(CircuitState::Closed)),
            failure_threshold,
            failure_count: Arc::new(Mutex::new(0)),
            last_failure_time: Arc::new(Mutex::new(None)),
            timeout,
        }
    }
    
    pub async fn call<F, T, E>(&self, f: F) -> Result<T, E>
    where
        F: Future<Output = Result<T, E>>,
    {
        let mut state = self.state.lock().await;
        
        match *state {
            CircuitState::Open => {
                let last_failure = self.last_failure_time.lock().await;
                if let Some(time) = *last_failure {
                    if time.elapsed() > self.timeout {
                        *state = CircuitState::HalfOpen;
                    } else {
                        return Err(/* CircuitOpen error */);
                    }
                }
            }
            CircuitState::HalfOpen => {
                drop(state);
                match f.await {
                    Ok(result) => {
                        let mut state = self.state.lock().await;
                        *state = CircuitState::Closed;
                        *self.failure_count.lock().await = 0;
                        return Ok(result);
                    }
                    Err(e) => {
                        let mut state = self.state.lock().await;
                        *state = CircuitState::Open;
                        return Err(e);
                    }
                }
            }
            CircuitState::Closed => {
                drop(state);
                match f.await {
                    Ok(result) => {
                        *self.failure_count.lock().await = 0;
                        Ok(result)
                    }
                    Err(e) => {
                        let mut count = self.failure_count.lock().await;
                        *count += 1;
                        
                        if *count >= self.failure_threshold {
                            let mut state = self.state.lock().await;
                            *state = CircuitState::Open;
                            *self.last_failure_time.lock().await = Some(Instant::now());
                        }
                        
                        Err(e)
                    }
                }
            }
        }
    }
}

// src/resilience/timeout.rs
pub async fn with_timeout<F, T>(
    future: F,
    duration: Duration,
) -> Result<T, TimeoutError>
where
    F: Future<Output = Result<T>>,
{
    match tokio::time::timeout(duration, future).await {
        Ok(Ok(result)) => Ok(result),
        Ok(Err(e)) => Err(e.into()),
        Err(_) => Err(TimeoutError::Elapsed),
    }
}

// src/resilience/retry.rs
pub async fn retry_with_backoff<F, T>(
    mut f: F,
    max_attempts: u32,
) -> Result<T>
where
    F: FnMut() -> Pin<Box<dyn Future<Output = Result<T>>>>,
{
    let mut backoff = Duration::from_millis(100);
    
    for attempt in 1..=max_attempts {
        match f().await {
            Ok(result) => return Ok(result),
            Err(e) if attempt < max_attempts => {
                tokio::time::sleep(backoff).await;
                backoff = (backoff * 2).min(Duration::from_secs(10));
            }
            Err(e) => return Err(e),
        }
    }
    
    Err(Error::MaxRetriesExceeded)
}
```

**Tests requeridos:**
- [ ] Circuit breaker (Closed → Open after failures)
- [ ] Circuit breaker (Open → HalfOpen after timeout)
- [ ] Circuit breaker (HalfOpen → Closed on success)
- [ ] Timeout triggers correctly
- [ ] Retry with backoff
- [ ] Graceful degradation

**Commits esperados:**
1. `feat(resilience): Circuit breaker states`
2. `feat(resilience): Timeout management`
3. `feat(resilience): Retry with backoff`
4. `feat(resilience): Fallback strategies`
5. `feat(week9): Resilience patterns complete`

**Métricas de éxito:**
- Circuit breaker transitions working ✅
- Timeouts prevent hanging ✅
- Retries with exponential backoff ✅
- Graceful degradation tested ✅
- Tests → 90%+ coverage

---

**FIN SPRINT 2**

---

## SPRINT 3: OBSERVABILITY & DEPLOYMENT (WEEKS 10-12)

---

## WEEK 10: OpenTelemetry + Logs

### 📋 ENTREGABLES (5h build)

**Objetivo:** Instrumentación completa + logs estructurados en Loki

**Deliverables:**
1. Tracing con `#[instrument]` en todos los handlers
2. Logs estructurados (info/warn/error)
3. OpenTelemetry export a Jaeger
4. Prometheus metrics scraping
5. Dashboards en Grafana

**Estructura de archivos:**
```
src/
├── observability/
│   ├── mod.rs
│   ├── tracing.rs             # OpenTelemetry setup
│   └── metrics.rs             # Prometheus metrics
└── main.rs                     # Init tracing subscriber
```

**Core Logic:**
```rust
// src/observability/tracing.rs
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};
use tracing_opentelemetry::OpenTelemetryLayer;

pub fn init_tracing() {
    let tracer = opentelemetry_jaeger::new_pipeline()
        .with_service_name("marketflow")
        .install_simple()
        .unwrap();
    
    tracing_subscriber::registry()
        .with(tracing_subscriber::fmt::layer())
        .with(OpenTelemetryLayer::new(tracer))
        .init();
}

// Usage in handlers
#[instrument(skip(pool))]
pub async fn create_payment(
    pool: &PgPool,
    amount: Decimal,
) -> Result<Payment> {
    info!("Creating payment: amount={}", amount);
    // ...
    Ok(payment)
}
```

**Tareas:**
- **Mon (1h):** Add `#[instrument]` a payment/user/product handlers
- **Tue (1h):** Structured logging con `tracing::info!`
- **Wed (1h):** OpenTelemetry export setup
- **Thu-Fri (2h):** Dashboards en Grafana (latency, throughput, errors)

**Tests requeridos:**
- [ ] Traces visibles en Jaeger
- [ ] Logs queryables en Loki
- [ ] Metrics scrapeables en Prometheus

**Commits esperados:**
1. `feat(observability): Tracing instrumentation`
2. `feat(observability): Structured logging`
3. `feat(observability): OpenTelemetry export`
4. `feat(week10): Observability complete`

**Métricas de éxito:**
- Traces en Jaeger ✅
- Logs en Loki ✅
- Metrics en Prometheus ✅
- Dashboards en Grafana ✅

---

## WEEK 11: Load Testing + Optimization

### 📋 ENTREGABLES (4h build)

**Objetivo:** 3000+ req/sec sostenidos + optimizaciones

**Deliverables:**
1. Load testing con `wrk`
2. Análisis de cuellos de botella
3. Optimizaciones (queries, índices, cache)
4. Re-test y validación

**Estructura de archivos:**
```
tests/
├── load/
│   ├── wrk_scripts/
│   │   ├── products.lua
│   │   └── payments.lua
│   └── results/
│       └── baseline.txt
└── benchmarks/
    └── criterion_bench.rs
```

**Load Test Scripts:**
```bash
# tests/load/run.sh
#!/bin/bash

echo "Testing /products endpoint..."
wrk -t4 -c100 -d60s http://localhost:3000/products

echo "Testing /payments endpoint..."
wrk -t4 -c100 -d60s -s tests/load/wrk_scripts/payments.lua http://localhost:3000/payments
```

**Optimizaciones típicas:**
```sql
-- Añadir índices faltantes
CREATE INDEX idx_payments_created ON payments(created_at);
CREATE INDEX idx_orders_user_created ON orders(user_id, created_at);

-- Query optimization
-- BEFORE: SELECT * FROM products
-- AFTER: SELECT id, name, price FROM products WHERE stock > 0
```

**Tareas:**
- **Mon (1h):** Setup load testing (wrk scripts)
- **Tue-Wed (2h):** Run tests, identificar cuellos de botella
- **Thu-Fri (1h):** Optimizar, re-test, validar 3000+ req/sec

**Tests requeridos:**
- [ ] Baseline load test
- [ ] Optimization applied
- [ ] Post-optimization test (3000+ req/sec)

**Commits esperados:**
1. `feat(perf): Load testing setup`
2. `feat(perf): Database query optimization`
3. `feat(perf): Cache optimization`
4. `feat(week11): 3000+ req/sec achieved`

**Métricas de éxito:**
- Throughput: 3000+ req/sec ✅
- P50 latency: <50ms ✅
- P99 latency: <100ms ✅
- Memory stable ✅

---

## WEEK 12: Documentation + Final

### 📋 ENTREGABLES (5h build)

**Objetivo:** OpenAPI docs + Swagger UI + Portfolio ready

**Deliverables:**
1. Code docs (`///` comments)
2. OpenAPI spec auto-generated (utoipa)
3. Swagger UI endpoint
4. README completo
5. Demo prep

**Estructura de archivos:**
```
src/
├── main.rs                     # Swagger UI endpoint
├── docs/
│   └── openapi.rs              # OpenAPI spec
└── README.md                   # Comprehensive docs

docs/
├── ARCHITECTURE.md
├── API.md
└── DEPLOYMENT.md
```

**OpenAPI Setup:**
```rust
// src/docs/openapi.rs
use utoipa::{OpenApi, ToSchema};

#[derive(OpenApi)]
#[openapi(
    paths(
        crate::payment::handlers::create_payment,
        crate::payment::handlers::get_payment,
        crate::user::handlers::register,
        crate::user::handlers::login,
    ),
    components(
        schemas(Payment, User, CreatePaymentRequest)
    ),
    tags(
        (name = "payments", description = "Payment operations"),
        (name = "auth", description = "Authentication endpoints")
    )
)]
struct ApiDoc;

// src/main.rs
use utoipa_swagger_ui::SwaggerUi;

let app = Router::new()
    .merge(SwaggerUi::new("/swagger-ui")
        .url("/api-docs/openapi.json", ApiDoc::openapi()));
```

**Tareas:**
- **Mon (1h):** Doc comments + setup utoipa
- **Tue (1.5h):** OpenAPI spec auto-generation
- **Wed-Thu (1.5h):** Swagger UI endpoint working
- **Fri (1h):** Demo prep + final polish

**API Endpoints Documentation:**
```
# AUTH
POST   /auth/register     → { email, name, password } → { user_id, token }
POST   /auth/login        → { email, password } → { user_id, token }

# PRODUCTS
GET    /products?query=X&limit=10  → [Product]
GET    /products/{id}              → Product

# CART
POST   /cart/items        → { product_id, quantity } → Cart
GET    /cart              → Cart
DELETE /cart/items/{id}   → 204

# ORDERS
POST   /orders            → { cart_id } → Order
GET    /orders            → [Order]
GET    /orders/{id}       → Order

# PAYMENTS
POST   /payments          → { order_id, amount } → Payment
POST   /payments/webhook  → (Stripe signature) → 200
GET    /payments/{id}     → Payment

# ADMIN
GET    /admin/ledger/balance     → Balance
GET    /admin/ledger/entries     → [LedgerEntry]
```

**Tests requeridos:**
- [ ] Swagger UI accessible at `/swagger-ui`
- [ ] All endpoints documented
- [ ] Examples working

**Commits esperados:**
1. `feat(docs): Code documentation`
2. `feat(docs): OpenAPI spec generation`
3. `feat(docs): Swagger UI integration`
4. `feat(week12): Documentation complete`

**Métricas de éxito:**
- Swagger UI functional ✅
- All endpoints documented ✅
- README comprehensive ✅
- Portfolio ready ✅

---

**FIN SPRINT 3**

---

## SPRINT 4: FINAL POLISH (WEEKS 13-15)

---

## WEEK 13: Advanced Features

### 📋 ENTREGABLES (35h)

**Objetivo:** Features avanzados + optimización de performance

**Deliverables:**
1. Search filters avanzados (price range, categories)
2. Product pagination mejorada
3. Order history con filtros
4. Cache warming strategy
5. Background job processing
6. Performance benchmarks

**Estructura de archivos:**
```
src/
├── product/
│   ├── filters.rs             # Advanced search filters
│   └── pagination.rs          # Cursor-based pagination
├── jobs/
│   ├── mod.rs
│   └── cache_warmer.rs        # Cache warming background job
└── tests/
    └── advanced_test.rs
```

**Database Enhancements:**
```sql
-- Product categories
CREATE TABLE categories (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR NOT NULL UNIQUE,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

ALTER TABLE products ADD COLUMN category_id UUID REFERENCES categories(id);
CREATE INDEX idx_products_category ON products(category_id);
CREATE INDEX idx_products_price ON products(price);

-- Order history index optimization
CREATE INDEX idx_orders_user_status_created ON orders(user_id, status, created_at DESC);
```

**API Endpoints:**
```
GET /products/advanced?category=electronics&min_price=100&max_price=500&cursor=xyz
GET /orders/history?status=delivered&from=2024-01-01&to=2024-12-31
POST /admin/cache/warm    → Trigger cache warming
```

**Core Logic:**
```rust
// src/product/filters.rs
#[derive(Deserialize)]
pub struct AdvancedFilters {
    pub category_id: Option<Uuid>,
    pub min_price: Option<Decimal>,
    pub max_price: Option<Decimal>,
    pub in_stock: Option<bool>,
}

pub async fn search_with_filters(
    pool: &PgPool,
    filters: AdvancedFilters,
    cursor: Option<Uuid>,
    limit: i64,
) -> Result<Vec<Product>> {
    let mut query = QueryBuilder::new("SELECT * FROM products WHERE 1=1");
    
    if let Some(cat) = filters.category_id {
        query.push(" AND category_id = ").push_bind(cat);
    }
    if let Some(min) = filters.min_price {
        query.push(" AND price >= ").push_bind(min);
    }
    if let Some(max) = filters.max_price {
        query.push(" AND price <= ").push_bind(max);
    }
    if filters.in_stock == Some(true) {
        query.push(" AND stock > 0");
    }
    if let Some(cursor) = cursor {
        query.push(" AND id > ").push_bind(cursor);
    }
    
    query.push(" ORDER BY id LIMIT ").push_bind(limit);
    
    query.build_query_as::<Product>()
        .fetch_all(pool)
        .await
}

// src/jobs/cache_warmer.rs
pub async fn warm_cache(redis: &redis::Connection, pool: &PgPool) -> Result<()> {
    info!("Starting cache warming...");
    
    // Warm top products
    let top_products = sqlx::query_as::<_, Product>(
        "SELECT * FROM products ORDER BY stock DESC LIMIT 100"
    )
    .fetch_all(pool)
    .await?;
    
    for product in top_products {
        let key = format!("product:{}", product.id);
        redis.set_ex(key, serde_json::to_string(&product)?, 3600)?;
    }
    
    info!("Cache warming complete");
    Ok(())
}
```

**Tests requeridos:**
- [ ] Advanced filters (category + price range)
- [ ] Cursor pagination working
- [ ] Order history filters
- [ ] Cache warming successful
- [ ] Performance benchmarks (before/after)

**Commits esperados:**
1. `feat(product): Advanced search filters`
2. `feat(product): Cursor-based pagination`
3. `feat(order): History filters`
4. `feat(cache): Warming strategy`
5. `feat(perf): Performance benchmarks`
6. `feat(week13): Advanced features complete`

**Métricas de éxito:**
- Advanced search: <100ms ✅
- Pagination efficient (no OFFSET) ✅
- Cache hit rate: 80%+ ✅
- 3000+ req/sec sustained ✅

---

## WEEK 14: Security Audit + Hardening

### 📋 ENTREGABLES (35h)

**Objetivo:** Security audit completo + hardening final

**Deliverables:**
1. `cargo audit` zero vulnerabilities
2. SQL injection prevention audit
3. OWASP Top 10 compliance check
4. Secrets rotation mechanism
5. Security logging
6. Penetration testing results

**Estructura de archivos:**
```
src/
├── security/
│   ├── audit.rs               # Security audit utilities
│   ├── secrets.rs             # Secrets management
│   └── logging.rs             # Security event logging
├── scripts/
│   ├── audit.sh               # Security audit script
│   └── rotate_secrets.sh      # Secret rotation
└── docs/
    └── SECURITY.md            # Security documentation
```

**Security Checklist:**
```markdown
# OWASP Top 10 Compliance

## A01:2021 – Broken Access Control
- [ ] JWT verification on all protected endpoints
- [ ] User can only access their own data
- [ ] Admin routes properly protected
- [ ] Rate limiting per user/IP

## A02:2021 – Cryptographic Failures
- [ ] Passwords hashed with Argon2
- [ ] Secrets not in code/logs
- [ ] TLS/HTTPS enforced
- [ ] Sensitive data encrypted at rest

## A03:2021 – Injection
- [ ] All SQL queries use parameterized queries (SQLx)
- [ ] No raw SQL string concatenation
- [ ] Input validation on all endpoints
- [ ] NoSQL injection prevention (N/A)

## A04:2021 – Insecure Design
- [ ] Double-entry ledger prevents financial errors
- [ ] Idempotency prevents duplicate charges
- [ ] State machine prevents invalid transitions
- [ ] Circuit breaker prevents cascading failures

## A05:2021 – Security Misconfiguration
- [ ] Docker containers run as non-root
- [ ] Unnecessary ports not exposed
- [ ] Error messages don't leak info
- [ ] Security headers set correctly

## A06:2021 – Vulnerable Components
- [ ] cargo audit: zero vulnerabilities
- [ ] Dependencies updated regularly
- [ ] No deprecated crates used

## A07:2021 – Authentication Failures
- [ ] Password strength enforced
- [ ] JWT expiration working
- [ ] No default credentials
- [ ] Session management secure

## A08:2021 – Data Integrity Failures
- [ ] Webhook signature verification
- [ ] Payment amount validation
- [ ] Double-entry ledger validation
- [ ] Event sourcing audit trail

## A09:2021 – Logging Failures
- [ ] All auth events logged
- [ ] Payment events logged
- [ ] No PII in logs
- [ ] Logs queryable in Loki

## A10:2021 – SSRF
- [ ] Webhook URLs validated
- [ ] No internal URL access from user input
```

**Core Logic:**
```rust
// src/security/audit.rs
pub async fn audit_permissions(pool: &PgPool) -> Result<AuditReport> {
    let mut report = AuditReport::new();
    
    // Check for users with admin role
    let admin_count = sqlx::query_scalar::<_, i64>(
        "SELECT COUNT(*) FROM users WHERE role = 'admin'"
    )
    .fetch_one(pool)
    .await?;
    
    report.add_finding("admin_users", admin_count);
    
    // Check for payments without idempotency key
    let unprotected_payments = sqlx::query_scalar::<_, i64>(
        "SELECT COUNT(*) FROM payments WHERE idempotency_key IS NULL"
    )
    .fetch_one(pool)
    .await?;
    
    if unprotected_payments > 0 {
        report.add_critical("payments_without_idempotency", unprotected_payments);
    }
    
    Ok(report)
}

// src/security/logging.rs
pub fn log_security_event(event: SecurityEvent) {
    match event {
        SecurityEvent::UnauthorizedAccess { user_id, resource } => {
            warn!(
                user_id = %user_id,
                resource = %resource,
                "Unauthorized access attempt"
            );
        }
        SecurityEvent::RateLimitExceeded { ip, endpoint } => {
            warn!(
                ip = %ip,
                endpoint = %endpoint,
                "Rate limit exceeded"
            );
        }
        SecurityEvent::InvalidToken { token_prefix } => {
            warn!(
                token = %token_prefix,
                "Invalid JWT token"
            );
        }
    }
}
```

**Audit Script:**
```bash
#!/bin/bash
# scripts/audit.sh

echo "Running security audit..."

# Dependency vulnerabilities
echo "Checking for vulnerable dependencies..."
cargo audit

# Check for hardcoded secrets
echo "Checking for hardcoded secrets..."
grep -r "api_key\|password\|secret" src/ --exclude-dir=target | grep -v "// SAFE"

# Check for SQL injection risks
echo "Checking for SQL injection risks..."
grep -r "format!\|concat!\|push_str" src/ | grep "SELECT\|INSERT\|UPDATE\|DELETE"

# Check for unsafe code
echo "Checking for unsafe code..."
grep -r "unsafe" src/

echo "Audit complete!"
```

**Tests requeridos:**
- [ ] cargo audit: zero vulnerabilities
- [ ] SQL injection attempts blocked
- [ ] Unauthorized access denied
- [ ] Rate limit prevents brute force
- [ ] Secrets not in logs
- [ ] Security events logged

**Commits esperados:**
1. `feat(security): OWASP compliance audit`
2. `feat(security): SQL injection prevention`
3. `feat(security): Security logging`
4. `feat(security): Secrets rotation mechanism`
5. `feat(security): Penetration test fixes`
6. `feat(week14): Security hardening complete`

**Métricas de éxito:**
- cargo audit: zero vulnerabilities ✅
- OWASP Top 10: compliant ✅
- Penetration test: passed ✅
- Security events: logged ✅

---

## WEEK 15: Final Integration + Portfolio

### 📋 ENTREGABLES (35h)

**Objetivo:** Portfolio-ready + demo preparation + final polish

**Deliverables:**
1. README completo con screenshots
2. Architecture diagram
3. Performance proof (screenshots)
4. GitHub clean history
5. Demo video
6. Portfolio presentation

**Estructura de archivos:**
```
marketflow/
├── README.md                  # Comprehensive project docs
├── docs/
│   ├── ARCHITECTURE.md        # System architecture
│   ├── API.md                 # API documentation
│   ├── DEPLOYMENT.md          # Deployment guide
│   ├── PERFORMANCE.md         # Performance metrics
│   └── SECURITY.md            # Security documentation
├── screenshots/
│   ├── grafana_dashboard.png
│   ├── jaeger_traces.png
│   ├── swagger_ui.png
│   └── load_test.png
└── demo/
    └── demo_video.mp4
```

**README.md Template:**
```markdown
# MarketFlow - Production Payment Marketplace

> High-performance payment marketplace built with Rust, achieving 3000+ req/sec with full observability

## 🎯 Key Features

- **Payment Processing**: Stripe integration with idempotency
- **Double-Entry Ledger**: ACID-compliant accounting
- **Resilience Patterns**: Circuit breaker, retry, timeout
- **Observability**: Full LGTM stack (Loki, Grafana, Tempo, Mimir)
- **Performance**: 3000+ req/sec, P99 < 100ms

## 🏗️ Architecture

[Architecture diagram]

## 📊 Performance Metrics

- **Throughput**: 3,247 req/sec
- **Latency P50**: 42ms
- **Latency P99**: 87ms
- **Uptime**: 99.9%

[Load test screenshot]

## 🚀 Quick Start

```bash
docker-compose up -d
cargo run
```

## 📚 API Documentation

Swagger UI: http://localhost:3000/swagger-ui

## 🔐 Security

- Argon2 password hashing
- JWT authentication
- Rate limiting (1000 req/min per user)
- OWASP Top 10 compliant

## 📈 Observability

- Logs: Loki (http://localhost:3100)
- Traces: Jaeger (http://localhost:16686)
- Metrics: Prometheus (http://localhost:9090)
- Dashboards: Grafana (http://localhost:3000)

## 🧪 Testing

```bash
cargo test                    # Unit tests
cargo test --test '*'         # Integration tests
./scripts/load_test.sh        # Load tests
```

## 📦 Tech Stack

- **Language**: Rust
- **Web**: Axum, Tonic (gRPC)
- **Database**: PostgreSQL, Redis
- **Events**: Kafka
- **Observability**: OpenTelemetry, LGTM stack
- **Payments**: Stripe

## 🎓 Learning Journey

Built in 15 weeks as part of intensive Rust learning:
- 75+ commits
- 7,500 LOC
- 90%+ test coverage
- Zero compiler warnings

## 📝 License

MIT
```

**Architecture Diagram:**
```
┌─────────────────────────────────────────────────────────────┐
│                        Client Layer                          │
│  (Web Browser, Mobile App, API Client)                      │
└────────────┬────────────────────────────────────────────────┘
             │
             ▼
┌─────────────────────────────────────────────────────────────┐
│                      API Gateway (Axum)                      │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐           │
│  │ Rate Limit  │ │    Auth     │ │   Logging   │           │
│  │ Middleware  │ │ Middleware  │ │ Middleware  │           │
│  └─────────────┘ └─────────────┘ └─────────────┘           │
└────────────┬────────────────────────────────────────────────┘
             │
             ▼
┌─────────────────────────────────────────────────────────────┐
│                      Service Layer                           │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐       │
│  │  User    │ │ Product  │ │   Cart   │ │  Order   │       │
│  │ Service  │ │ Service  │ │ Service  │ │ Service  │       │
│  └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘       │
│       │            │            │            │              │
│  ┌────┴─────┐ ┌────┴─────┐ ┌────┴─────┐ ┌────┴─────┐       │
│  │ Payment  │ │  Ledger  │ │ Webhook  │ │Resilience│       │
│  │ Service  │ │ Service  │ │ Service  │ │ Patterns │       │
│  └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘       │
└───────┼────────────┼────────────┼────────────┼─────────────┘
        │            │            │            │
        ▼            ▼            ▼            ▼
┌─────────────────────────────────────────────────────────────┐
│                      Data Layer                              │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐        │
│  │ PostgreSQL   │ │    Redis     │ │    Kafka     │        │
│  │ (Primary DB) │ │   (Cache)    │ │  (Events)    │        │
│  └──────────────┘ └──────────────┘ └──────────────┘        │
└─────────────────────────────────────────────────────────────┘
             │
             ▼
┌─────────────────────────────────────────────────────────────┐
│                   Observability Layer                        │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐       │
│  │   Loki   │ │Prometheus│ │  Jaeger  │ │ Grafana  │       │
│  │  (Logs)  │ │ (Metrics)│ │ (Traces) │ │(Dashboard│       │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘       │
└─────────────────────────────────────────────────────────────┘
             │
             ▼
┌─────────────────────────────────────────────────────────────┐
│                   External Services                          │
│  ┌──────────────────────────────────────────────────────┐   │
│  │                  Stripe API                           │   │
│  │  (Payment Processing + Webhooks)                     │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

**Demo Checklist:**
```markdown
# Demo Script (5 minutes)

## 1. Introduction (30s)
- Project overview
- Tech stack highlights
- Key metrics

## 2. Live Demo (2 min)
1. Start services: `docker-compose up -d`
2. Show Grafana dashboard (live metrics)
3. Create user → Login → Get JWT
4. Search products → Add to cart
5. Create order → Process payment
6. Show Stripe webhook received

## 3. Observability (1 min)
1. Jaeger: Show distributed trace
2. Loki: Query payment logs
3. Prometheus: Show request rate

## 4. Performance (1 min)
1. Run load test
2. Show 3000+ req/sec
3. Show P99 latency < 100ms

## 5. Code Quality (30s)
1. Show GitHub commits (75+)
2. Show test coverage (90%+)
3. Show zero compiler warnings
```

**GitHub Profile README:**
```markdown
# MarketFlow

Production-grade payment marketplace built with Rust.

**Highlights:**
- 🚀 3000+ req/sec throughput
- 📊 Full observability (LGTM stack)
- 🔐 OWASP Top 10 compliant
- 💰 Stripe integration with idempotency
- 📈 90%+ test coverage

[View Project →](https://github.com/username/marketflow)
```

**Tasks:**
- **Mon (8h):** README + architecture docs
- **Tue (7h):** Screenshots + performance proof
- **Wed (8h):** Demo video recording
- **Thu (7h):** GitHub cleanup + final commits
- **Fri (5h):** Portfolio presentation prep

**Commits esperados:**
1. `docs: Comprehensive README`
2. `docs: Architecture diagram`
3. `docs: Performance documentation`
4. `docs: Security documentation`
5. `demo: Demo video + screenshots`
6. `feat(week15): Portfolio ready`

**Métricas de éxito:**
- README comprehensive ✅
- Architecture documented ✅
- Performance proven (screenshots) ✅
- Demo video < 5min ✅
- GitHub clean history ✅
- Portfolio presentation ready ✅

---

## 🎯 SPRINT 4 COMPLETION CRITERIA

### Code Quality
- [ ] 75+ commits on GitHub
- [ ] 7-9K LOC production code
- [ ] 90%+ test coverage
- [ ] Zero compiler warnings
- [ ] cargo clippy: zero issues

### Performance
- [ ] 3000+ req/sec sustained
- [ ] P50 latency < 50ms
- [ ] P99 latency < 100ms
- [ ] Memory usage stable

### Security
- [ ] cargo audit: zero vulnerabilities
- [ ] OWASP Top 10 compliant
- [ ] Penetration test passed
- [ ] Security events logged

### Documentation
- [ ] README complete
- [ ] API documented (Swagger)
- [ ] Architecture diagram
- [ ] Performance proof
- [ ] Demo video

### Portfolio Ready
- [ ] GitHub profile updated
- [ ] Screenshots collected
- [ ] Demo prepared
- [ ] Presentation ready

---

**FIN SPRINT 4**
