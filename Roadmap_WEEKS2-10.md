# MARKETFLOW - ROADMAP OPTIMIZADO (WEEKS 2-10)

---

# SPRINT 0: FOUNDATION (Weeks 1-2) [Week 1 ✅ COMPLETADA]

---

## WEEK 2: LEDGER + AUTH (43h)

### 1. OBJETIVO
Sistema de ledger double-entry + autenticación JWT + tablas de transacciones + 13 tests (ALTA + MEDIA prioridad)

### 2. DELIVERABLES
1. Double-entry ledger table (debits/credits ACID-guaranteed)
2. Transaction table (audit trail)
3. JWT authentication (Bearer token)
4. Argon2 password hashing
5. User table + role-based access
6. LedgerService (debit/credit operations)
7. Auth middleware (verify JWT)
8. 13 tests (idempotency ledger, auth, transactional integrity)

### 3. ESTRUCTURA DE ARCHIVOS
```
marketflow/
├── src/
│   ├── auth/
│   │   ├── mod.rs
│   │   ├── models.rs              # User, Claims, TokenPayload
│   │   ├── repository.rs          # UserRepository trait
│   │   ├── service.rs             # AuthService (JWT generation)
│   │   └── middleware.rs          # JWT verification middleware
│   ├── ledger/
│   │   ├── mod.rs
│   │   ├── models.rs              # LedgerEntry, EntryType
│   │   ├── repository.rs          # LedgerRepository trait
│   │   └── service.rs             # LedgerService (debit/credit)
│   ├── transaction/
│   │   ├── mod.rs
│   │   ├── models.rs              # Transaction model
│   │   └── repository.rs          # TransactionRepository
│   └── payment/
│       └── service.rs             # ← actualizar para usar ledger
├── migrations/
│   ├── 20240115_create_payments.sql
│   ├── 20240116_create_users.sql
│   ├── 20240116_create_ledger.sql
│   └── 20240116_create_transactions.sql
└── tests/
    ├── auth_tests.rs
    ├── ledger_tests.rs
    └── transaction_tests.rs
```

### 4. DATABASE SCHEMA
```sql
-- migrations/20240116_create_users.sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    username VARCHAR(255) UNIQUE NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    role VARCHAR(50) NOT NULL DEFAULT 'user' CHECK (role IN ('user', 'admin')),
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_users_email ON users(email);

-- migrations/20240116_create_ledger.sql
CREATE TABLE ledger_entries (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    transaction_id UUID NOT NULL,
    account VARCHAR(100) NOT NULL,  -- 'revenue', 'expense', 'asset', 'liability', 'equity'
    entry_type VARCHAR(50) NOT NULL CHECK (entry_type IN ('debit', 'credit')),
    amount DECIMAL(19, 4) NOT NULL CHECK (amount > 0),
    description VARCHAR(500),
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    CONSTRAINT ledger_uq UNIQUE (transaction_id, account, entry_type)
);

CREATE INDEX idx_ledger_transaction ON ledger_entries(transaction_id);
CREATE INDEX idx_ledger_account ON ledger_entries(account);

-- migrations/20240116_create_transactions.sql
CREATE TABLE transactions (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    payment_id UUID NOT NULL REFERENCES payments(id),
    user_id UUID NOT NULL REFERENCES users(id),
    status VARCHAR(50) NOT NULL DEFAULT 'pending',
    ledger_balanced BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_transactions_payment ON transactions(payment_id);
CREATE INDEX idx_transactions_user ON transactions(user_id);
CREATE INDEX idx_transactions_status ON transactions(status);
```

### 5. API ENDPOINTS
```
# Authentication
POST /auth/register
{
  "username": "john_doe",
  "email": "john@example.com",
  "password": "secure_password_123"
}
Response 201:
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "username": "john_doe",
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}

POST /auth/login
{
  "email": "john@example.com",
  "password": "secure_password_123"
}
Response 200:
{
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "user": { "id", "username", "email" }
}

# Ledger (requires auth)
POST /ledger/entries
Authorization: Bearer <token>
{
  "transaction_id": "550e8400-e29b-41d4-a716-446655440000",
  "account": "revenue",
  "entry_type": "credit",
  "amount": 100.00,
  "description": "Payment for order #123"
}
Response 201:
{
  "id": "...",
  "transaction_id": "...",
  "account": "revenue",
  "entry_type": "credit",
  "amount": 100.00
}

GET /ledger/verify/:transaction_id
Response 200:
{
  "balanced": true,
  "total_debits": 100.00,
  "total_credits": 100.00
}
```

### 6. CORE LOGIC
```rust
// src/auth/models.rs
use uuid::Uuid;
use serde::{Deserialize, Serialize};

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct User {
    pub id: Uuid,
    pub username: String,
    pub email: String,
    pub role: String,
}

#[derive(Debug, Deserialize)]
pub struct RegisterRequest {
    pub username: String,
    pub email: String,
    pub password: String,
}

#[derive(Debug, Serialize)]
pub struct AuthResponse {
    pub user: User,
    pub token: String,
}

#[derive(Debug, Clone)]
pub struct Claims {
    pub sub: String,  // user_id
    pub exp: i64,     // expiration
    pub iat: i64,     // issued at
}

// src/auth/service.rs
use argon2::{
    password_hash::{Ident, ParamString, PasswordHasher, SaltString},
    Argon2,
};
use jsonwebtoken::{encode, decode, Header, Validation, EncodingKey, DecodingKey};
use tracing::instrument;

pub struct AuthService {
    repository: Arc<dyn UserRepository>,
    jwt_secret: String,
}

impl AuthService {
    #[instrument(skip(self, req))]
    pub async fn register(&self, req: RegisterRequest) -> Result<AuthResponse> {
        // Hash password
        let salt = SaltString::generate(rand::thread_rng());
        let argon2 = Argon2::default();
        let password_hash = argon2
            .hash_password(req.password.as_bytes(), &salt)
            .map_err(|e| AppError::InternalError(e.to_string()))?
            .to_string();

        // Create user
        let user = User {
            id: Uuid::new_v4(),
            username: req.username,
            email: req.email,
            role: "user".to_string(),
        };
        
        self.repository.create_user(&user, &password_hash).await?;

        // Generate JWT
        let token = self.generate_token(&user)?;
        
        Ok(AuthResponse { user, token })
    }

    fn generate_token(&self, user: &User) -> Result<String> {
        let now = chrono::Utc::now().timestamp();
        let claims = Claims {
            sub: user.id.to_string(),
            exp: now + 86400,  // 24 hours
            iat: now,
        };

        encode(
            &Header::default(),
            &claims,
            &EncodingKey::from_secret(self.jwt_secret.as_bytes()),
        )
        .map_err(|e| AppError::InternalError(e.to_string()))
    }

    pub fn verify_token(&self, token: &str) -> Result<Claims> {
        decode(
            token,
            &DecodingKey::from_secret(self.jwt_secret.as_bytes()),
            &Validation::default(),
        )
        .map(|data| data.claims)
        .map_err(|e| AppError::InvalidInput(format!("Invalid token: {}", e)))
    }
}

// src/ledger/service.rs
use rust_decimal::Decimal;

pub struct LedgerService {
    repository: Arc<dyn LedgerRepository>,
}

impl LedgerService {
    #[instrument(skip(self))]
    pub async fn record_entry(
        &self,
        transaction_id: Uuid,
        account: &str,
        entry_type: &str,
        amount: Decimal,
    ) -> Result<()> {
        // Create debit entry
        let entry = LedgerEntry {
            id: Uuid::new_v4(),
            transaction_id,
            account: account.to_string(),
            entry_type: entry_type.to_string(),
            amount,
            description: None,
            created_at: Utc::now(),
        };
        
        self.repository.create_entry(&entry).await?;
        Ok(())
    }

    #[instrument(skip(self))]
    pub async fn verify_transaction_balanced(&self, transaction_id: Uuid) -> Result<bool> {
        let debits = self.repository
            .sum_entries(transaction_id, "debit")
            .await?;
        let credits = self.repository
            .sum_entries(transaction_id, "credit")
            .await?;
        
        Ok(debits == credits)
    }
}

// Flujo de pago con ledger:
// 1. create_payment() → Payment {status: Pending}
// 2. Stripe webhook → payment succeeded
// 3. create_transaction() → Transaction
// 4. record_entry(debit, "asset", 100)  → dinero entra
// 5. record_entry(credit, "revenue", 100) → se registra ingreso
// 6. verify_transaction_balanced() → debits == credits ✓
```

### 7. TESTS REQUERIDOS (13 tests)
```rust
// ALTA prioridad (critical path)
- [ ] test_register_user_success
      → register → user created → token generated

- [ ] test_register_duplicate_email
      → register with same email twice → second fails with 409

- [ ] test_login_success
      → login with correct credentials → token returned

- [ ] test_login_wrong_password
      → login with wrong password → 401

- [ ] test_jwt_verification_valid
      → valid JWT → decoded successfully

- [ ] test_jwt_verification_expired
      → expired JWT → verification fails

- [ ] test_jwt_verification_invalid_signature
      → tampered JWT → verification fails

- [ ] test_ledger_double_entry_debit_credit
      → record debit → record credit → balanced == true

- [ ] test_ledger_unbalanced_detection
      → record only debit (no credit) → balanced == false

- [ ] test_ledger_prevents_duplicate_entries
      → same transaction/account/type twice → second fails

- [ ] test_payment_with_ledger_recording
      → create payment → record entries → verify balanced

// MEDIA prioridad
- [ ] test_auth_middleware_blocks_missing_token
      → request sin Authorization header → 401

- [ ] test_auth_middleware_extracts_user_from_token
      → valid JWT → user_id extracted and available in handler
```

### 8. COMMITS ESPERADOS
```
1. feat(auth): User model, repository, registration flow
2. feat(auth): JWT generation and token verification
3. feat(auth): Argon2 password hashing
4. feat(ledger): Ledger entry table and repository
5. feat(ledger): Double-entry validation (debit == credit)
6. feat(ledger): Integrate ledger with payment flow
7. test(auth): Authentication and JWT tests
8. test(ledger): Double-entry and transaction tests
```

### 9. MÉTRICAS DE ÉXITO
```
✅ User registration working (POST /auth/register)
✅ Login returns JWT token
✅ Protected endpoints require valid token
✅ Ledger entries always balanced (debits == credits)
✅ Password hashed with Argon2 (never stored plain)
✅ cargo test → 13/13 tests passing
✅ Jaeger traces include auth & ledger operations
✅ Zero SQL errors (proper migrations)
```

### 10. DISTRIBUCIÓN DE HORAS (43h = Week 1 + 3h extra)
```
Lunes (9h):
  - User table + auth models (3h)
  - UserRepository SQLx impl (3h)
  - Argon2 password hashing (2h)
  - Basic register endpoint (1h)

Martes (9h):
  - JWT token generation (3h)
  - JWT verification (2h)
  - Auth middleware (2h)
  - Login endpoint (2h)

Miércoles (9h):
  - Ledger table + models (2h)
  - LedgerRepository SQLx impl (3h)
  - LedgerService (double-entry logic) (3h)
  - Ledger endpoints (1h)

Jueves (9h):
  - Integration: Payment + Ledger flow (3h)
  - Transaction table + service (3h)
  - Verify transaction balanced endpoint (2h)
  - Error handling refinement (1h)

Viernes (7h):
  - Auth tests (5 tests) (3h)
  - Ledger tests (5 tests) (3h)
  - Integration tests (transaction flow) (1h)
```

---

# SPRINT 1: E-COMMERCE CORE (Weeks 3-5)

---

## WEEK 3: PRODUCTS + SEARCH (40h)

### 1. OBJETIVO
Tabla de productos + full-text search en PostgreSQL + API de listado/filtrado + 8 tests

### 2. DELIVERABLES
1. Products table (name, description, price, stock, category)
2. Full-text search indexes (tsvector en PostgreSQL)
3. ProductRepository (CRUD + search)
4. ProductService (business logic)
5. REST endpoints (GET /products, /search?q=...)
6. Category filtering
7. Pagination (limit, offset)
8. 8 tests (search, filtering, pagination)

### 3. ESTRUCTURA DE ARCHIVOS
```
marketflow/
├── src/
│   ├── product/
│   │   ├── mod.rs
│   │   ├── models.rs              # Product, SearchQuery, ProductFilter
│   │   ├── repository.rs          # ProductRepository trait
│   │   ├── service.rs             # ProductService (search, filter)
│   │   └── handlers.rs            # GET /products, /search
│   └── ...
├── migrations/
│   └── 20240117_create_products.sql
└── tests/
    └── product_tests.rs
```

### 4. DATABASE SCHEMA
```sql
CREATE TABLE categories (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name VARCHAR(100) UNIQUE NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

CREATE TABLE products (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name VARCHAR(255) NOT NULL,
    description TEXT,
    price DECIMAL(19, 4) NOT NULL CHECK (price > 0),
    stock INTEGER NOT NULL DEFAULT 0 CHECK (stock >= 0),
    category_id UUID NOT NULL REFERENCES categories(id),
    search_vector tsvector,  -- Full-text search
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_products_search ON products USING gin(search_vector);
CREATE INDEX idx_products_category ON products(category_id);
CREATE INDEX idx_products_price ON products(price);
```

### 5. API ENDPOINTS
```
GET /products?category=electronics&limit=20&offset=0
Response:
{
  "items": [
    {
      "id": "...",
      "name": "Laptop",
      "price": 999.99,
      "stock": 5,
      "category": "electronics"
    }
  ],
  "total": 150,
  "limit": 20,
  "offset": 0
}

GET /search?q=laptop
Response: [{ "id", "name", "price", ... }]

GET /products/:id
Response: { "id", "name", "description", "price", "stock", ... }
```

### 6. CORE LOGIC
```rust
// Full-text search query
pub async fn search(
    &self,
    query: &str,
    limit: i64,
    offset: i64,
) -> Result<(Vec<Product>, i64)> {
    let search_query = format!("%{}%", query);
    
    let products = sqlx::query_as::<_, Product>(
        "SELECT * FROM products 
         WHERE to_tsvector('english', name || ' ' || coalesce(description, '')) @@ 
               plainto_tsquery('english', $1)
         LIMIT $2 OFFSET $3"
    )
    .bind(query)
    .bind(limit)
    .bind(offset)
    .fetch_all(&self.pool)
    .await?;
    
    let total = sqlx::query_scalar::<_, i64>(
        "SELECT COUNT(*) FROM products 
         WHERE to_tsvector('english', name || ' ' || coalesce(description, '')) @@ 
               plainto_tsquery('english', $1)"
    )
    .bind(query)
    .fetch_one(&self.pool)
    .await?;
    
    Ok((products, total))
}
```

### 7. TESTS REQUERIDOS (8 tests)
```
- [ ] test_search_products_by_name
- [ ] test_search_products_by_description
- [ ] test_filter_by_category
- [ ] test_filter_by_price_range
- [ ] test_pagination_limit_offset
- [ ] test_search_empty_results
- [ ] test_product_stock_availability
- [ ] test_search_performance_1000_products
```

### 8-10. [Similar format to Week 2]

---

## WEEK 4: CART + ORDERS (40h)

### 1. OBJETIVO
Shopping cart (Redis) + Orders table + Order state machine + 8 tests

### 2. DELIVERABLES
1. Cart service (Redis hash: user_id → product_id → quantity)
2. Order table (items, total, status)
3. OrderItem table (junction: order → product)
4. OrderState machine (pending → payment_received → processing → shipped)
5. REST endpoints (POST /cart/add, GET /cart, POST /orders)
6. 8 tests (cart operations, order creation, state transitions)

### 3. ESTRUCTURA
```
marketflow/
├── src/
│   ├── cart/
│   │   ├── models.rs          # CartItem, Cart
│   │   ├── service.rs         # CartService (Redis ops)
│   │   └── handlers.rs        # POST/GET /cart
│   ├── order/
│   │   ├── models.rs          # Order, OrderItem, OrderStatus
│   │   ├── repository.rs
│   │   ├── service.rs         # OrderService (state machine)
│   │   └── handlers.rs        # POST /orders
```

### 4. DATABASE SCHEMA
```sql
CREATE TABLE orders (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id),
    total DECIMAL(19, 4) NOT NULL,
    status VARCHAR(50) NOT NULL DEFAULT 'pending',
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

CREATE TABLE order_items (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    order_id UUID NOT NULL REFERENCES orders(id),
    product_id UUID NOT NULL REFERENCES products(id),
    quantity INTEGER NOT NULL,
    price_at_purchase DECIMAL(19, 4) NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);
```

### 5. API ENDPOINTS
```
POST /cart/add
{ "product_id": "...", "quantity": 2 }

GET /cart
Response: { "items": [{ "product_id", "quantity", "price" }], "total": 1999.98 }

POST /orders
{ "payment_method": "stripe" }
Response: { "id": "...", "total": 1999.98, "status": "pending" }

GET /orders/:id
Response: { "id", "total", "status", "items": [...] }
```

---

## WEEK 5: RESILIENCE + SECURITY (40h)

### 1. OBJETIVO
Circuit breaker para Stripe + Rate limiting + Input validation + OWASP Top 10 awareness + 6 tests

### 2. DELIVERABLES
1. Circuit breaker pattern (Stripe client failover)
2. Rate limiter middleware (1000 req/min/user)
3. Input validation (validator crate)
4. CORS configuration
5. SQL injection prevention (SQLx typed queries ✓)
6. CSRF token (si aplica)
7. 6 tests (circuit breaker, rate limiting, validation)

### 3. CÓDIGO
```rust
// Circuit breaker
pub struct CircuitBreaker<T> {
    failures: Arc<AtomicU32>,
    last_failure_time: Arc<Mutex<Option<Instant>>>,
    threshold: u32,
    timeout: Duration,
    operation: Box<dyn Fn() -> Pin<Box<dyn Future<Output = Result<T>>>> + Send + Sync>,
}

// Rate limiter middleware
pub async fn rate_limit_middleware(
    Request(req): Request<Body>,
) -> Result<Request<Body>> {
    let user_id = extract_user_id(&req)?;
    let key = format!("rate_limit:{}", user_id);
    
    let count: u32 = redis_client.incr(&key).await?;
    if count == 1 {
        redis_client.expire(&key, 60).await?;
    }
    
    if count > 1000 {
        return Err(AppError::RateLimitExceeded);
    }
    
    Ok(req)
}
```

---

# SPRINT 2: OBSERVABILITY + POLISH (Weeks 6-8)

---

## WEEK 6: OBSERVABILITY IDEAL (48h = 40h + 8h extra)

### 1. OBJETIVO
OpenTelemetry completo + Jaeger traces end-to-end + Prometheus metrics + Grafana 3 dashboards + screenshots

### 2. DELIVERABLES
1. Complete OTEL instrumentation
2. Jaeger distributed tracing (all requests)
3. Prometheus metrics (request latency, payment success rate, error rate)
4. Grafana dashboards:
   - Request latency (p50, p95, p99)
   - Payment success rate
   - Database connection pool usage
5. Custom metrics (payments processed, ledger entries created)
6. Screenshots proof

### 3. ESTRUCTURA
```
marketflow/
├── src/
│   └── observability/
│       ├── tracing.rs          # OTEL + Jaeger
│       ├── metrics.rs          # Prometheus metrics
│       └── middleware.rs        # Auto-instrument all requests
├── prometheus.yml              # Config
└── grafana/
    └── dashboards/
        ├── latency.json
        ├── payments.json
        └── database.json
```

### 4. MÉTRICAS IMPLEMENTADAS
```rust
// Prometheus metrics
static PAYMENT_COUNTER: AtomicU64 = AtomicU64::new(0);  // total payments
static PAYMENT_FAILURE_COUNTER: AtomicU64 = AtomicU64::new(0);
static REQUEST_DURATION: Histogram; // request latency
static DATABASE_POOL_SIZE: Gauge;   // active connections

// En handlers:
#[instrument]
pub async fn create_payment(...) -> Result<...> {
    PAYMENT_COUNTER.fetch_add(1, Ordering::Relaxed);
    // ... lógica ...
}
```

### 5. JAEGER TRACES
```
Cada request genera:
    GET /products?category=electronics
        ├─ route_matching (0.5ms)
        ├─ jwt_verification (2ms)
        ├─ database_query (15ms)
        │   ├─ connection_acquire (0.1ms)
        │   ├─ sql_execute (14ms)
        │   └─ result_parsing (0.9ms)
        ├─ serialization (0.5ms)
        └─ response (0ms)
    Total: 18ms
```

### 6. GRAFANA DASHBOARDS
```
Dashboard 1: Latency
  - Graph: Request latency (p50, p95, p99) over time
  - Legend shows values
  - Alert threshold at 200ms p99

Dashboard 2: Payments
  - Success rate (% of succeeded vs failed)
  - Payments per minute (throughput)
  - Average payment amount
  - Error distribution (Stripe errors, validation, etc)

Dashboard 3: Database
  - Active connections
  - Query latency
  - Connection pool usage
  - Transaction count per hour
```

### 7. TESTS REQUERIDOS
```
- [ ] test_jaeger_traces_received
      → create payment → verify trace in Jaeger

- [ ] test_prometheus_metrics_published
      → make 10 requests → verify counter incremented in Prometheus

- [ ] test_request_latency_tracked
      → slow request → latency recorded in histogram

- [ ] test_error_metrics_recorded
      → failed payment → error counter incremented

- [ ] test_dashboard_renders
      → Grafana available → dashboards have data
```

### 8. DISTRIBUCIÓN DE HORAS (48h)
```
Lunes (10h):
  - Complete OTEL instrumentation (4h)
  - Custom metrics (payments, errors) (3h)
  - Middleware for auto-tracing (3h)

Martes (10h):
  - Jaeger configuration (2h)
  - Trace context propagation (3h)
  - Verify end-to-end tracing (3h)
  - Jaeger UI screenshots (2h)

Miércoles (10h):
  - Prometheus metrics exposition (3h)
  - Dashboard 1: Latency (4h)
  - Dashboard 2: Payments (3h)

Jueves (10h):
  - Dashboard 3: Database (3h)
  - Grafana panels refinement (3h)
  - Alert configuration (2h)
  - Screenshots proof (2h)

Viernes (8h):
  - Documentation of observability (3h)
  - Integration tests for metrics (3h)
  - Load test to populate metrics (2h)
```

---

## WEEK 7: PERFORMANCE + LOAD TESTS (40h)

### 1. OBJETIVO
Alcanzar 3000+ req/sec sostenidos + profiling + Load tests con Criterion + benchmarks

### 2. DELIVERABLES
1. Criterion benchmarks (payment creation, product search, order creation)
2. Load test scenarios (3000 req/sec)
3. Profiling with `perf` / `flamegraph`
4. Database query optimization
5. Connection pooling tuning
6. Cache hit rate optimization
7. Report: Before/After metrics

### 3. LOAD TEST SETUP
```bash
# Using wrk tool
wrk -t12 -c400 -d30s http://localhost:3000/products

# Target: 3000 req/sec
# Latency goal: P99 < 100ms
```

### 4. TESTS REQUERIDOS
```
- [ ] test_3000_requests_per_second
      → sustained load → throughput >= 3000 req/sec

- [ ] test_p99_latency_under_100ms
      → at 3000 req/sec → P99 latency < 100ms

- [ ] test_zero_panic_under_load
      → 1 hour sustained load → zero panics

- [ ] test_memory_leak_detection
      → overnight run → memory stable

- [ ] test_database_connection_pool_efficiency
      → peak load → all connections in use
```

---

## WEEK 8: DOCUMENTATION + PORTFOLIO (40h)

### 1. OBJETIVO
README profesional + OpenAPI/Swagger UI + Architecture diagrams + Demo video

### 2. DELIVERABLES
1. Comprehensive README (install, run, test, deploy)
2. OpenAPI spec (Utoipa) + Swagger UI on /swagger-ui
3. Architecture diagrams (Mermaid)
4. API documentation (request/response examples)
5. Performance report (3000 req/sec benchmark)
6. Deploy guide (Docker → production)
7. Demo video (5-10 min)

### 3. README STRUCTURE
```markdown
# MarketFlow

## Overview
Rust-based marketplace with payments, high performance (3000+ req/sec)

## Quick Start
git clone ...
docker-compose up -d
cargo run
curl http://localhost:3000/health

## Architecture
[Diagram here]

## API Documentation
[Swagger UI link]

## Performance Metrics
- Throughput: 3247 req/sec
- P99 Latency: 87ms
- Memory: 120MB

## Testing
cargo test --release
cargo bench

## Deployment
[Instructions]
```

---

# WEEKS 9-10: BUFFER & FINAL ADJUSTMENTS

## WEEK 9: AJUSTES FINALES (30h)

### Actividades
1. Bug fixes (si aplica)
2. Refactoring (si aplica)
3. Missing tests (si aplica)
4. Polish documentation
5. Código limpio (fmt, clippy, warnings)

---

## WEEK 10: BUFFER (Contingency)

Reservado para:
- Unexpected issues
- Performance tuning
- Final polish
- Demo preparation

---

# RESUMEN: ROADMAP COMPLETO

```
WEEK 1 (40h):  ✅ Docker + Stripe + Payments + Tests
WEEK 2 (43h):  🟡 Ledger + Auth + Transactions + Tests
WEEK 3 (40h):  🟡 Products + Search + Tests
WEEK 4 (40h):  🟡 Cart + Orders + Tests
WEEK 5 (40h):  🟡 Resilience + Security + Tests
WEEK 6 (48h):  🟡 Observability IDEAL + Dashboards
WEEK 7 (40h):  🟡 Performance + Load Tests
WEEK 8 (40h):  🟡 Documentation + Portfolio
WEEK 9 (30h):  🟡 Buffer & Final adjustments
WEEK 10:       🟡 Final contingency

TOTAL: ~351 horas (~9.5 semanas)

Proyección: Fin de Week 8 = Project DONE
Weeks 9-10 = Safety margin
```

---

# KPIs DE ÉXITO FINAL

- ✅ 3000+ req/sec sostenidos (probado)
- ✅ 90%+ test coverage
- ✅ Zero runtime panics
- ✅ Complete observability (Jaeger + Prometheus + Grafana)
- ✅ Professional README + Architecture docs
- ✅ Demo video funcionando
- ✅ All endpoints on Swagger UI
- ✅ Production-grade code (no warnings)

EOF
cat /home/claude/WEEKS_2_TO_10_ROADMAP.md
Salida

# MARKETFLOW - ROADMAP OPTIMIZADO (WEEKS 2-10)

---

# SPRINT 0: FOUNDATION (Weeks 1-2) [Week 1 ✅ COMPLETADA]

---

## WEEK 2: LEDGER + AUTH (43h)

### 1. OBJETIVO
Sistema de ledger double-entry + autenticación JWT + tablas de transacciones + 13 tests (ALTA + MEDIA prioridad)

### 2. DELIVERABLES
1. Double-entry ledger table (debits/credits ACID-guaranteed)
2. Transaction table (audit trail)
3. JWT authentication (Bearer token)
4. Argon2 password hashing
5. User table + role-based access
6. LedgerService (debit/credit operations)
7. Auth middleware (verify JWT)
8. 13 tests (idempotency ledger, auth, transactional integrity)

### 3. ESTRUCTURA DE ARCHIVOS
```
marketflow/
├── src/
│   ├── auth/
│   │   ├── mod.rs
│   │   ├── models.rs              # User, Claims, TokenPayload
│   │   ├── repository.rs          # UserRepository trait
│   │   ├── service.rs             # AuthService (JWT generation)
│   │   └── middleware.rs          # JWT verification middleware
│   ├── ledger/
│   │   ├── mod.rs
│   │   ├── models.rs              # LedgerEntry, EntryType
│   │   ├── repository.rs          # LedgerRepository trait
│   │   └── service.rs             # LedgerService (debit/credit)
│   ├── transaction/
│   │   ├── mod.rs
│   │   ├── models.rs              # Transaction model
│   │   └── repository.rs          # TransactionRepository
│   └── payment/
│       └── service.rs             # ← actualizar para usar ledger
├── migrations/
│   ├── 20240115_create_payments.sql
│   ├── 20240116_create_users.sql
│   ├── 20240116_create_ledger.sql
│   └── 20240116_create_transactions.sql
└── tests/
    ├── auth_tests.rs
    ├── ledger_tests.rs
    └── transaction_tests.rs
```

### 4. DATABASE SCHEMA
```sql
-- migrations/20240116_create_users.sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    username VARCHAR(255) UNIQUE NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    role VARCHAR(50) NOT NULL DEFAULT 'user' CHECK (role IN ('user', 'admin')),
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_users_email ON users(email);

-- migrations/20240116_create_ledger.sql
CREATE TABLE ledger_entries (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    transaction_id UUID NOT NULL,
    account VARCHAR(100) NOT NULL,  -- 'revenue', 'expense', 'asset', 'liability', 'equity'
    entry_type VARCHAR(50) NOT NULL CHECK (entry_type IN ('debit', 'credit')),
    amount DECIMAL(19, 4) NOT NULL CHECK (amount > 0),
    description VARCHAR(500),
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    CONSTRAINT ledger_uq UNIQUE (transaction_id, account, entry_type)
);

CREATE INDEX idx_ledger_transaction ON ledger_entries(transaction_id);
CREATE INDEX idx_ledger_account ON ledger_entries(account);

-- migrations/20240116_create_transactions.sql
CREATE TABLE transactions (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    payment_id UUID NOT NULL REFERENCES payments(id),
    user_id UUID NOT NULL REFERENCES users(id),
    status VARCHAR(50) NOT NULL DEFAULT 'pending',
    ledger_balanced BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_transactions_payment ON transactions(payment_id);
CREATE INDEX idx_transactions_user ON transactions(user_id);
CREATE INDEX idx_transactions_status ON transactions(status);
```

### 5. API ENDPOINTS
```
# Authentication
POST /auth/register
{
  "username": "john_doe",
  "email": "john@example.com",
  "password": "secure_password_123"
}
Response 201:
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "username": "john_doe",
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}

POST /auth/login
{
  "email": "john@example.com",
  "password": "secure_password_123"
}
Response 200:
{
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "user": { "id", "username", "email" }
}

# Ledger (requires auth)
POST /ledger/entries
Authorization: Bearer <token>
{
  "transaction_id": "550e8400-e29b-41d4-a716-446655440000",
  "account": "revenue",
  "entry_type": "credit",
  "amount": 100.00,
  "description": "Payment for order #123"
}
Response 201:
{
  "id": "...",
  "transaction_id": "...",
  "account": "revenue",
  "entry_type": "credit",
  "amount": 100.00
}

GET /ledger/verify/:transaction_id
Response 200:
{
  "balanced": true,
  "total_debits": 100.00,
  "total_credits": 100.00
}
```

### 6. CORE LOGIC
```rust
// src/auth/models.rs
use uuid::Uuid;
use serde::{Deserialize, Serialize};

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct User {
    pub id: Uuid,
    pub username: String,
    pub email: String,
    pub role: String,
}

#[derive(Debug, Deserialize)]
pub struct RegisterRequest {
    pub username: String,
    pub email: String,
    pub password: String,
}

#[derive(Debug, Serialize)]
pub struct AuthResponse {
    pub user: User,
    pub token: String,
}

#[derive(Debug, Clone)]
pub struct Claims {
    pub sub: String,  // user_id
    pub exp: i64,     // expiration
    pub iat: i64,     // issued at
}

// src/auth/service.rs
use argon2::{
    password_hash::{Ident, ParamString, PasswordHasher, SaltString},
    Argon2,
};
use jsonwebtoken::{encode, decode, Header, Validation, EncodingKey, DecodingKey};
use tracing::instrument;

pub struct AuthService {
    repository: Arc<dyn UserRepository>,
    jwt_secret: String,
}

impl AuthService {
    #[instrument(skip(self, req))]
    pub async fn register(&self, req: RegisterRequest) -> Result<AuthResponse> {
        // Hash password
        let salt = SaltString::generate(rand::thread_rng());
        let argon2 = Argon2::default();
        let password_hash = argon2
            .hash_password(req.password.as_bytes(), &salt)
            .map_err(|e| AppError::InternalError(e.to_string()))?
            .to_string();

        // Create user
        let user = User {
            id: Uuid::new_v4(),
            username: req.username,
            email: req.email,
            role: "user".to_string(),
        };
        
        self.repository.create_user(&user, &password_hash).await?;

        // Generate JWT
        let token = self.generate_token(&user)?;
        
        Ok(AuthResponse { user, token })
    }

    fn generate_token(&self, user: &User) -> Result<String> {
        let now = chrono::Utc::now().timestamp();
        let claims = Claims {
            sub: user.id.to_string(),
            exp: now + 86400,  // 24 hours
            iat: now,
        };

        encode(
            &Header::default(),
            &claims,
            &EncodingKey::from_secret(self.jwt_secret.as_bytes()),
        )
        .map_err(|e| AppError::InternalError(e.to_string()))
    }

    pub fn verify_token(&self, token: &str) -> Result<Claims> {
        decode(
            token,
            &DecodingKey::from_secret(self.jwt_secret.as_bytes()),
            &Validation::default(),
        )
        .map(|data| data.claims)
        .map_err(|e| AppError::InvalidInput(format!("Invalid token: {}", e)))
    }
}

// src/ledger/service.rs
use rust_decimal::Decimal;

pub struct LedgerService {
    repository: Arc<dyn LedgerRepository>,
}

impl LedgerService {
    #[instrument(skip(self))]
    pub async fn record_entry(
        &self,
        transaction_id: Uuid,
        account: &str,
        entry_type: &str,
        amount: Decimal,
    ) -> Result<()> {
        // Create debit entry
        let entry = LedgerEntry {
            id: Uuid::new_v4(),
            transaction_id,
            account: account.to_string(),
            entry_type: entry_type.to_string(),
            amount,
            description: None,
            created_at: Utc::now(),
        };
        
        self.repository.create_entry(&entry).await?;
        Ok(())
    }

    #[instrument(skip(self))]
    pub async fn verify_transaction_balanced(&self, transaction_id: Uuid) -> Result<bool> {
        let debits = self.repository
            .sum_entries(transaction_id, "debit")
            .await?;
        let credits = self.repository
            .sum_entries(transaction_id, "credit")
            .await?;
        
        Ok(debits == credits)
    }
}

// Flujo de pago con ledger:
// 1. create_payment() → Payment {status: Pending}
// 2. Stripe webhook → payment succeeded
// 3. create_transaction() → Transaction
// 4. record_entry(debit, "asset", 100)  → dinero entra
// 5. record_entry(credit, "revenue", 100) → se registra ingreso
// 6. verify_transaction_balanced() → debits == credits ✓
```

### 7. TESTS REQUERIDOS (13 tests)
```rust
// ALTA prioridad (critical path)
- [ ] test_register_user_success
      → register → user created → token generated

- [ ] test_register_duplicate_email
      → register with same email twice → second fails with 409

- [ ] test_login_success
      → login with correct credentials → token returned

- [ ] test_login_wrong_password
      → login with wrong password → 401

- [ ] test_jwt_verification_valid
      → valid JWT → decoded successfully

- [ ] test_jwt_verification_expired
      → expired JWT → verification fails

- [ ] test_jwt_verification_invalid_signature
      → tampered JWT → verification fails

- [ ] test_ledger_double_entry_debit_credit
      → record debit → record credit → balanced == true

- [ ] test_ledger_unbalanced_detection
      → record only debit (no credit) → balanced == false

- [ ] test_ledger_prevents_duplicate_entries
      → same transaction/account/type twice → second fails

- [ ] test_payment_with_ledger_recording
      → create payment → record entries → verify balanced

// MEDIA prioridad
- [ ] test_auth_middleware_blocks_missing_token
      → request sin Authorization header → 401

- [ ] test_auth_middleware_extracts_user_from_token
      → valid JWT → user_id extracted and available in handler
```

### 8. COMMITS ESPERADOS
```
1. feat(auth): User model, repository, registration flow
2. feat(auth): JWT generation and token verification
3. feat(auth): Argon2 password hashing
4. feat(ledger): Ledger entry table and repository
5. feat(ledger): Double-entry validation (debit == credit)
6. feat(ledger): Integrate ledger with payment flow
7. test(auth): Authentication and JWT tests
8. test(ledger): Double-entry and transaction tests
```

### 9. MÉTRICAS DE ÉXITO
```
✅ User registration working (POST /auth/register)
✅ Login returns JWT token
✅ Protected endpoints require valid token
✅ Ledger entries always balanced (debits == credits)
✅ Password hashed with Argon2 (never stored plain)
✅ cargo test → 13/13 tests passing
✅ Jaeger traces include auth & ledger operations
✅ Zero SQL errors (proper migrations)
```

### 10. DISTRIBUCIÓN DE HORAS (43h = Week 1 + 3h extra)
```
Lunes (9h):
  - User table + auth models (3h)
  - UserRepository SQLx impl (3h)
  - Argon2 password hashing (2h)
  - Basic register endpoint (1h)

Martes (9h):
  - JWT token generation (3h)
  - JWT verification (2h)
  - Auth middleware (2h)
  - Login endpoint (2h)

Miércoles (9h):
  - Ledger table + models (2h)
  - LedgerRepository SQLx impl (3h)
  - LedgerService (double-entry logic) (3h)
  - Ledger endpoints (1h)

Jueves (9h):
  - Integration: Payment + Ledger flow (3h)
  - Transaction table + service (3h)
  - Verify transaction balanced endpoint (2h)
  - Error handling refinement (1h)

Viernes (7h):
  - Auth tests (5 tests) (3h)
  - Ledger tests (5 tests) (3h)
  - Integration tests (transaction flow) (1h)
```

---

# SPRINT 1: E-COMMERCE CORE (Weeks 3-5)

---

## WEEK 3: PRODUCTS + SEARCH (40h)

### 1. OBJETIVO
Tabla de productos + full-text search en PostgreSQL + API de listado/filtrado + 8 tests

### 2. DELIVERABLES
1. Products table (name, description, price, stock, category)
2. Full-text search indexes (tsvector en PostgreSQL)
3. ProductRepository (CRUD + search)
4. ProductService (business logic)
5. REST endpoints (GET /products, /search?q=...)
6. Category filtering
7. Pagination (limit, offset)
8. 8 tests (search, filtering, pagination)

### 3. ESTRUCTURA DE ARCHIVOS
```
marketflow/
├── src/
│   ├── product/
│   │   ├── mod.rs
│   │   ├── models.rs              # Product, SearchQuery, ProductFilter
│   │   ├── repository.rs          # ProductRepository trait
│   │   ├── service.rs             # ProductService (search, filter)
│   │   └── handlers.rs            # GET /products, /search
│   └── ...
├── migrations/
│   └── 20240117_create_products.sql
└── tests/
    └── product_tests.rs
```

### 4. DATABASE SCHEMA
```sql
CREATE TABLE categories (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name VARCHAR(100) UNIQUE NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

CREATE TABLE products (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name VARCHAR(255) NOT NULL,
    description TEXT,
    price DECIMAL(19, 4) NOT NULL CHECK (price > 0),
    stock INTEGER NOT NULL DEFAULT 0 CHECK (stock >= 0),
    category_id UUID NOT NULL REFERENCES categories(id),
    search_vector tsvector,  -- Full-text search
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_products_search ON products USING gin(search_vector);
CREATE INDEX idx_products_category ON products(category_id);
CREATE INDEX idx_products_price ON products(price);
```

### 5. API ENDPOINTS
```
GET /products?category=electronics&limit=20&offset=0
Response:
{
  "items": [
    {
      "id": "...",
      "name": "Laptop",
      "price": 999.99,
      "stock": 5,
      "category": "electronics"
    }
  ],
  "total": 150,
  "limit": 20,
  "offset": 0
}

GET /search?q=laptop
Response: [{ "id", "name", "price", ... }]

GET /products/:id
Response: { "id", "name", "description", "price", "stock", ... }
```

### 6. CORE LOGIC
```rust
// Full-text search query
pub async fn search(
    &self,
    query: &str,
    limit: i64,
    offset: i64,
) -> Result<(Vec<Product>, i64)> {
    let search_query = format!("%{}%", query);
    
    let products = sqlx::query_as::<_, Product>(
        "SELECT * FROM products 
         WHERE to_tsvector('english', name || ' ' || coalesce(description, '')) @@ 
               plainto_tsquery('english', $1)
         LIMIT $2 OFFSET $3"
    )
    .bind(query)
    .bind(limit)
    .bind(offset)
    .fetch_all(&self.pool)
    .await?;
    
    let total = sqlx::query_scalar::<_, i64>(
        "SELECT COUNT(*) FROM products 
         WHERE to_tsvector('english', name || ' ' || coalesce(description, '')) @@ 
               plainto_tsquery('english', $1)"
    )
    .bind(query)
    .fetch_one(&self.pool)
    .await?;
    
    Ok((products, total))
}
```

### 7. TESTS REQUERIDOS (8 tests)
```
- [ ] test_search_products_by_name
- [ ] test_search_products_by_description
- [ ] test_filter_by_category
- [ ] test_filter_by_price_range
- [ ] test_pagination_limit_offset
- [ ] test_search_empty_results
- [ ] test_product_stock_availability
- [ ] test_search_performance_1000_products
```

### 8-10. [Similar format to Week 2]

---

## WEEK 4: CART + ORDERS (40h)

### 1. OBJETIVO
Shopping cart (Redis) + Orders table + Order state machine + 8 tests

### 2. DELIVERABLES
1. Cart service (Redis hash: user_id → product_id → quantity)
2. Order table (items, total, status)
3. OrderItem table (junction: order → product)
4. OrderState machine (pending → payment_received → processing → shipped)
5. REST endpoints (POST /cart/add, GET /cart, POST /orders)
6. 8 tests (cart operations, order creation, state transitions)

### 3. ESTRUCTURA
```
marketflow/
├── src/
│   ├── cart/
│   │   ├── models.rs          # CartItem, Cart
│   │   ├── service.rs         # CartService (Redis ops)
│   │   └── handlers.rs        # POST/GET /cart
│   ├── order/
│   │   ├── models.rs          # Order, OrderItem, OrderStatus
│   │   ├── repository.rs
│   │   ├── service.rs         # OrderService (state machine)
│   │   └── handlers.rs        # POST /orders
```

### 4. DATABASE SCHEMA
```sql
CREATE TABLE orders (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id),
    total DECIMAL(19, 4) NOT NULL,
    status VARCHAR(50) NOT NULL DEFAULT 'pending',
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

CREATE TABLE order_items (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    order_id UUID NOT NULL REFERENCES orders(id),
    product_id UUID NOT NULL REFERENCES products(id),
    quantity INTEGER NOT NULL,
    price_at_purchase DECIMAL(19, 4) NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);
```

### 5. API ENDPOINTS
```
POST /cart/add
{ "product_id": "...", "quantity": 2 }

GET /cart
Response: { "items": [{ "product_id", "quantity", "price" }], "total": 1999.98 }

POST /orders
{ "payment_method": "stripe" }
Response: { "id": "...", "total": 1999.98, "status": "pending" }

GET /orders/:id
Response: { "id", "total", "status", "items": [...] }
```

---

## WEEK 5: RESILIENCE + SECURITY (40h)

### 1. OBJETIVO
Circuit breaker para Stripe + Rate limiting + Input validation + OWASP Top 10 awareness + 6 tests

### 2. DELIVERABLES
1. Circuit breaker pattern (Stripe client failover)
2. Rate limiter middleware (1000 req/min/user)
3. Input validation (validator crate)
4. CORS configuration
5. SQL injection prevention (SQLx typed queries ✓)
6. CSRF token (si aplica)
7. 6 tests (circuit breaker, rate limiting, validation)

### 3. CÓDIGO
```rust
// Circuit breaker
pub struct CircuitBreaker<T> {
    failures: Arc<AtomicU32>,
    last_failure_time: Arc<Mutex<Option<Instant>>>,
    threshold: u32,
    timeout: Duration,
    operation: Box<dyn Fn() -> Pin<Box<dyn Future<Output = Result<T>>>> + Send + Sync>,
}

// Rate limiter middleware
pub async fn rate_limit_middleware(
    Request(req): Request<Body>,
) -> Result<Request<Body>> {
    let user_id = extract_user_id(&req)?;
    let key = format!("rate_limit:{}", user_id);
    
    let count: u32 = redis_client.incr(&key).await?;
    if count == 1 {
        redis_client.expire(&key, 60).await?;
    }
    
    if count > 1000 {
        return Err(AppError::RateLimitExceeded);
    }
    
    Ok(req)
}
```

---

# SPRINT 2: OBSERVABILITY + POLISH (Weeks 6-8)

---

## WEEK 6: OBSERVABILITY IDEAL (48h = 40h + 8h extra)

### 1. OBJETIVO
OpenTelemetry completo + Jaeger traces end-to-end + Prometheus metrics + Grafana 3 dashboards + screenshots

### 2. DELIVERABLES
1. Complete OTEL instrumentation
2. Jaeger distributed tracing (all requests)
3. Prometheus metrics (request latency, payment success rate, error rate)
4. Grafana dashboards:
   - Request latency (p50, p95, p99)
   - Payment success rate
   - Database connection pool usage
5. Custom metrics (payments processed, ledger entries created)
6. Screenshots proof

### 3. ESTRUCTURA
```
marketflow/
├── src/
│   └── observability/
│       ├── tracing.rs          # OTEL + Jaeger
│       ├── metrics.rs          # Prometheus metrics
│       └── middleware.rs        # Auto-instrument all requests
├── prometheus.yml              # Config
└── grafana/
    └── dashboards/
        ├── latency.json
        ├── payments.json
        └── database.json
```

### 4. MÉTRICAS IMPLEMENTADAS
```rust
// Prometheus metrics
static PAYMENT_COUNTER: AtomicU64 = AtomicU64::new(0);  // total payments
static PAYMENT_FAILURE_COUNTER: AtomicU64 = AtomicU64::new(0);
static REQUEST_DURATION: Histogram; // request latency
static DATABASE_POOL_SIZE: Gauge;   // active connections

// En handlers:
#[instrument]
pub async fn create_payment(...) -> Result<...> {
    PAYMENT_COUNTER.fetch_add(1, Ordering::Relaxed);
    // ... lógica ...
}
```

### 5. JAEGER TRACES
```
Cada request genera:
    GET /products?category=electronics
        ├─ route_matching (0.5ms)
        ├─ jwt_verification (2ms)
        ├─ database_query (15ms)
        │   ├─ connection_acquire (0.1ms)
        │   ├─ sql_execute (14ms)
        │   └─ result_parsing (0.9ms)
        ├─ serialization (0.5ms)
        └─ response (0ms)
    Total: 18ms
```

### 6. GRAFANA DASHBOARDS
```
Dashboard 1: Latency
  - Graph: Request latency (p50, p95, p99) over time
  - Legend shows values
  - Alert threshold at 200ms p99

Dashboard 2: Payments
  - Success rate (% of succeeded vs failed)
  - Payments per minute (throughput)
  - Average payment amount
  - Error distribution (Stripe errors, validation, etc)

Dashboard 3: Database
  - Active connections
  - Query latency
  - Connection pool usage
  - Transaction count per hour
```

### 7. TESTS REQUERIDOS
```
- [ ] test_jaeger_traces_received
      → create payment → verify trace in Jaeger

- [ ] test_prometheus_metrics_published
      → make 10 requests → verify counter incremented in Prometheus

- [ ] test_request_latency_tracked
      → slow request → latency recorded in histogram

- [ ] test_error_metrics_recorded
      → failed payment → error counter incremented

- [ ] test_dashboard_renders
      → Grafana available → dashboards have data
```

### 8. DISTRIBUCIÓN DE HORAS (48h)
```
Lunes (10h):
  - Complete OTEL instrumentation (4h)
  - Custom metrics (payments, errors) (3h)
  - Middleware for auto-tracing (3h)

Martes (10h):
  - Jaeger configuration (2h)
  - Trace context propagation (3h)
  - Verify end-to-end tracing (3h)
  - Jaeger UI screenshots (2h)

Miércoles (10h):
  - Prometheus metrics exposition (3h)
  - Dashboard 1: Latency (4h)
  - Dashboard 2: Payments (3h)

Jueves (10h):
  - Dashboard 3: Database (3h)
  - Grafana panels refinement (3h)
  - Alert configuration (2h)
  - Screenshots proof (2h)

Viernes (8h):
  - Documentation of observability (3h)
  - Integration tests for metrics (3h)
  - Load test to populate metrics (2h)
```

---

## WEEK 7: PERFORMANCE + LOAD TESTS (40h)

### 1. OBJETIVO
Alcanzar 3000+ req/sec sostenidos + profiling + Load tests con Criterion + benchmarks

### 2. DELIVERABLES
1. Criterion benchmarks (payment creation, product search, order creation)
2. Load test scenarios (3000 req/sec)
3. Profiling with `perf` / `flamegraph`
4. Database query optimization
5. Connection pooling tuning
6. Cache hit rate optimization
7. Report: Before/After metrics

### 3. LOAD TEST SETUP
```bash
# Using wrk tool
wrk -t12 -c400 -d30s http://localhost:3000/products

# Target: 3000 req/sec
# Latency goal: P99 < 100ms
```

### 4. TESTS REQUERIDOS
```
- [ ] test_3000_requests_per_second
      → sustained load → throughput >= 3000 req/sec

- [ ] test_p99_latency_under_100ms
      → at 3000 req/sec → P99 latency < 100ms

- [ ] test_zero_panic_under_load
      → 1 hour sustained load → zero panics

- [ ] test_memory_leak_detection
      → overnight run → memory stable

- [ ] test_database_connection_pool_efficiency
      → peak load → all connections in use
```

---

## WEEK 8: DOCUMENTATION + PORTFOLIO (40h)

### 1. OBJETIVO
README profesional + OpenAPI/Swagger UI + Architecture diagrams + Demo video

### 2. DELIVERABLES
1. Comprehensive README (install, run, test, deploy)
2. OpenAPI spec (Utoipa) + Swagger UI on /swagger-ui
3. Architecture diagrams (Mermaid)
4. API documentation (request/response examples)
5. Performance report (3000 req/sec benchmark)
6. Deploy guide (Docker → production)
7. Demo video (5-10 min)

### 3. README STRUCTURE
```markdown
# MarketFlow

## Overview
Rust-based marketplace with payments, high performance (3000+ req/sec)

## Quick Start
git clone ...
docker-compose up -d
cargo run
curl http://localhost:3000/health

## Architecture
[Diagram here]

## API Documentation
[Swagger UI link]

## Performance Metrics
- Throughput: 3247 req/sec
- P99 Latency: 87ms
- Memory: 120MB

## Testing
cargo test --release
cargo bench

## Deployment
[Instructions]
```

---

# WEEKS 9-10: BUFFER & FINAL ADJUSTMENTS

## WEEK 9: AJUSTES FINALES (30h)

### Actividades
1. Bug fixes (si aplica)
2. Refactoring (si aplica)
3. Missing tests (si aplica)
4. Polish documentation
5. Código limpio (fmt, clippy, warnings)

---

## WEEK 10: BUFFER (Contingency)

Reservado para:
- Unexpected issues
- Performance tuning
- Final polish
- Demo preparation

---

# RESUMEN: ROADMAP COMPLETO

```
WEEK 1 (40h):  ✅ Docker + Stripe + Payments + Tests
WEEK 2 (43h):  🟡 Ledger + Auth + Transactions + Tests
WEEK 3 (40h):  🟡 Products + Search + Tests
WEEK 4 (40h):  🟡 Cart + Orders + Tests
WEEK 5 (40h):  🟡 Resilience + Security + Tests
WEEK 6 (48h):  🟡 Observability IDEAL + Dashboards
WEEK 7 (40h):  🟡 Performance + Load Tests
WEEK 8 (40h):  🟡 Documentation + Portfolio
WEEK 9 (30h):  🟡 Buffer & Final adjustments
WEEK 10:       🟡 Final contingency

TOTAL: ~351 horas (~9.5 semanas)

Proyección: Fin de Week 8 = Project DONE
Weeks 9-10 = Safety margin
```

---

# KPIs DE ÉXITO FINAL

- ✅ 3000+ req/sec sostenidos (probado)
- ✅ 90%+ test coverage
- ✅ Zero runtime panics
- ✅ Complete observability (Jaeger + Prometheus + Grafana)
- ✅ Professional README + Architecture docs
- ✅ Demo video funcionando
- ✅ All endpoints on Swagger UI
- ✅ Production-grade code (no warnings)
