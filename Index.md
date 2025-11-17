**Monday Week 1:** Ya tienes base para empezar con Stripe integration.

---

## SPRINT 0: FOUNDATION (WEEKS 1-2)

### WEEK 1: Docker Setup + Stripe Integration

**Goal:** Docker fully working. Stripe integration done. First Rust code running.

---

#### 🦀 RUST TOPICS TO STUDY (6-8 hours)

**Core Rust Concepts:**
- [ ] Async/await with Tokio (already know basics)
- [ ] Traits and trait objects
- [ ] Error handling: Result<T, E> and ? operator
- [ ] Lifetimes (basic understanding)
- [ ] Pattern matching

**Frameworks & Crates:**
- [ ] **Axum**: Web framework (routing, extractors, middleware)
  - Resource: https://docs.rs/axum
  - Focus: Route definition, JSON extraction, error handling
- [ ] **Tokio**: Async runtime (already familiar)
  - Resource: https://tokio.rs
- [ ] **SQLx**: Type-safe SQL queries (compile-time checking)
  - Resource: https://github.com/launchbread/sqlx
  - Focus: Query macros, connection pooling
- [ ] **Serde**: JSON serialization/deserialization
  - Resource: https://serde.rs
  - Focus: Derive macros (#[derive(Serialize, Deserialize)])

**HTTP & REST:**
- [ ] HTTP verbs (GET, POST, PUT, DELETE)
- [ ] JSON request/response handling
- [ ] Status codes and error responses
- [ ] Middleware (logging, CORS, auth)

**Specific for This Week:**
- [ ] Docker networking from Rust perspective
- [ ] Environment variables in Rust (dotenv, std::env)
- [ ] OpenTelemetry instrumentation basics

**Learning Resources:**
- Axum Examples: https://github.com/tokio-rs/axum/tree/main/examples
- Tokio Tutorial: https://tokio.rs/tokio/tutorial
- Rust Book Ch. 17 (OOP): https://doc.rust-lang.org/book/ch17-00-oop.html

---

**Learn Objectives (8h breakdown):**

| Day | Concept | Time | Resource |
|-----|---------|------|----------|
| Mon | Axum basics + routing | 1.5h | Axum docs + examples |
| Mon | Serde + JSON handling | 1.5h | Serde book |
| Tue | SQLx + async queries | 2h | SQLx docs + tutorial |
| Tue | Error handling in Rust | 1h | Rust Book |
| Wed | Middleware + extractors | 2h | Axum examples |
| Thu | Environment + config | 1h | dotenv crate docs |
| Fri | Review + practice | 1h | Write small test endpoint |

---

**Build Objectives (30-32h)**

**Monday (1h learning + 3h build + 1h docker setup)**
- Learn: Axum routing, Serde JSON
- Build: Create basic Axum server, verify docker-compose works
- Output: Server listens on 3000, all Docker services ✅
- Rust focus: 
  ```rust
  // Learn this pattern
  use axum::{routing::post, Router};
  
  let app = Router::new()
      .route("/health", get(health_check));
  ```
- GitHub: `feat(infra): Docker Compose + Axum server setup`

**Tuesday (1.5h learning + 3.5h build)**
- Learn: SQLx async queries, connection pooling
- Build: Connect to PostgreSQL from Rust, implement PaymentService
- Output: Can query database from Rust
- Rust focus:
  ```rust
  // Learn async/await + SQLx
  #[sqlx::query_as(PaymentRecord)]
  pub async fn get_payment(id: Uuid) -> Result<Payment> {
      sqlx::query_as("SELECT * FROM payments WHERE id = $1")
          .bind(id)
          .fetch_one(&pool)
          .await
  }
  ```
- GitHub: `feat(stripe): Payment struct + database connection`

**Wednesday (1h learning + 4h build)**
- Learn: Error handling, custom Result types
- Build: Webhook handler with signature verification
- Output: Webhooks verified and stored
- Rust focus:
  ```rust
  // Learn custom error types
  #[derive(Debug)]
  pub enum PaymentError {
      InvalidSignature,
      DatabaseError(sqlx::Error),
  }
  
  impl From<sqlx::Error> for PaymentError {
      fn from(err: sqlx::Error) -> Self {
          PaymentError::DatabaseError(err)
      }
  }
  ```
- GitHub: `feat(stripe): Webhook verification`

**Thursday (1.5h learning + 3.5h build)**
- Learn: Idempotency patterns, state management
- Build: Implement idempotency in payment creation
- Output: Payments idempotent
- Rust focus:
  ```rust
  // Learn idempotent patterns
  pub async fn create_payment_idempotent(
      idempotency_key: String,
      amount: Decimal,
  ) -> Result<Payment> {
      // Check if already exists
      if let Ok(payment) = get_by_idempotency_key(&idempotency_key).await {
          return Ok(payment);
      }
      // Create new
      create_payment(amount).await
  }
  ```
- GitHub: `feat(stripe): Idempotency implementation`

**Friday (1h learning + 3h build)**
- Learn: OpenTelemetry instrumentation, tracing macros
- Build: Add tracing to payment flow
- Output: Traces visible in Jaeger
- Rust focus:
  ```rust
  // Learn tracing instrumentation
  use tracing::{info, error, instrument};
  
  #[instrument(skip(pool))]
  pub async fn create_payment(amount: Decimal) -> Result<Payment> {
      info!("Creating payment for amount: {}", amount);
      // ... code
  }
  ```
- GitHub: `feat(week1): Stripe + observability working`

**Time Summary:** 8h learning + 30h building = 38h total

**Deliverables (Portfolio Items)**
- Rust Stripe service (~/500 LOC)
- Webhook handlers + verification
- Idempotency implementation
- Tests: 100% pass
- Observability: Traces visible in Jaeger
- GitHub: 5 commits

**Database Schema**

```
payments:
  id (UUID, PK)
  stripe_payment_intent_id (VARCHAR, UNIQUE)
  amount (DECIMAL)
  status (VARCHAR) -- pending, succeeded, failed
  created_at (TIMESTAMP)
```

---

### WEEK 2: Double-Entry Ledger + Reconciliation

#### 🦀 RUST TOPICS TO STUDY (6-8 hours)

**Key Concepts:**
- [ ] Enums and pattern matching (DebitOrCredit)
- [ ] Struct composition and borrowing
- [ ] Transactions in async code
- [ ] Unit testing (test modules)
- [ ] Iterators and functional patterns

**Crates:**
- [ ] **Decimal**: Accurate financial calculations
  - Resource: https://docs.rs/rust_decimal
  - Focus: Decimal arithmetic (no floating point!)
- [ ] **UUID**: Unique identifiers
  - Resource: https://docs.rs/uuid
- [ ] **Chrono**: Date/time handling
  - Resource: https://docs.rs/chrono

**Rust Patterns:**
- [ ] Transaction patterns (BEGIN/COMMIT/ROLLBACK)
- [ ] Testing with databases (sqlx test utilities)
- [ ] Enum variants for state (Debit vs Credit)
- [ ] Result chaining with ?

**Specific for This Week:**
- [ ] Financial calculations in Rust
- [ ] Database transactions
- [ ] Test database setup

**Learning Resources:**
- Rust Book Ch. 6 (Enums): https://doc.rust-lang.org/book/ch06-00-enums-and-pattern-matching.html
- SQLx Test Support: https://github.com/launchbread/sqlx#compile-time-verification
- Decimal Crate: https://docs.rs/rust_decimal

---

**Learn Objectives (6h breakdown):**

| Day | Concept | Time | Resource |
|-----|---------|------|----------|
| Mon | Enums + pattern matching | 1.5h | Rust Book Ch. 6 |
| Tue | Decimal for finances | 1h | Decimal docs |
| Wed | Database transactions | 1.5h | SQLx docs |
| Thu | Unit testing in Rust | 1h | Rust Book Ch. 11 |
| Fri | Integration testing | 0.5h | SQLx test examples |

---

**Build Objectives (30-32h)**

**Mon-Tue (5h learning + 8h build)**
- Learn: Enums, pattern matching, Decimal
- Build: LedgerEntry struct, Debit/Credit enum, create function
- Output: Ledger schema working, entries created
- Rust focus:
  ```rust
  #[derive(Debug, Clone, Copy)]
  pub enum Direction {
      Debit,
      Credit,
  }
  
  #[derive(Debug)]
  pub struct LedgerEntry {
      pub id: Uuid,
      pub direction: Direction,
      pub amount: Decimal,
  }
  
  impl LedgerEntry {
      pub async fn record(
          db: &PgPool,
          user_id: Uuid,
          direction: Direction,
          amount: Decimal,
      ) -> Result<Self> {
          // INSERT query
      }
  }
  ```
- GitHub: `feat(ledger): LedgerEntry domain model`

**Wed-Thu (2h learning + 10h build)**
- Learn: Transactions, verification logic
- Build: Reconciliation engine, balance verification
- Output: Ledger verification working
- Rust focus:
  ```rust
  // Learn transaction pattern
  pub async fn verify_balanced(db: &PgPool) -> Result<bool> {
      // All debits must equal all credits
      let (debits, credits) = sqlx::query!(
          "SELECT 
              SUM(CASE WHEN direction = 'Debit' THEN amount ELSE 0 END) as debits,
              SUM(CASE WHEN direction = 'Credit' THEN amount ELSE 0 END) as credits
          FROM ledger_entries"
      )
      .fetch_one(db)
      .await?;
      
      Ok(debits == credits)
  }
  ```
- GitHub: `feat(ledger): Reconciliation engine`

**Fri (1h learning + 4h build)**
- Learn: Testing with databases
- Build: Tests for ledger logic
- Output: All tests pass
- Rust focus:
  ```rust
  #[tokio::test]
  async fn test_ledger_balanced() {
      let pool = setup_test_db().await;
      
      LedgerEntry::record(&pool, user_id, Debit, dec!(100)).await.unwrap();
      LedgerEntry::record(&pool, user_id, Credit, dec!(100)).await.unwrap();
      
      let balanced = verify_balanced(&pool).await.unwrap();
      assert!(balanced);
  }
  ```
- GitHub: `feat(ledger): Tests + verification`

**Deliverables**
- Double-entry ledger (~/400 LOC)
- Reconciliation engine
- Tests: 100% coverage on ledger logic
- GitHub: 5 commits

---

## SPRINT 1: E-COMMERCE CORE (WEEKS 3-5)

### WEEK 3: User Service (Auth + JWT)

#### 🦀 RUST TOPICS TO STUDY (8 hours)

**Key Concepts:**
- [ ] Traits and trait bounds
- [ ] Generic programming
- [ ] Lifetime annotations (intermediate)
- [ ] Privacy rules (pub, pub(crate))
- [ ] Module system (mod, use)

**Crates:**
- [ ] **jsonwebtoken**: JWT creation/validation
  - Resource: https://docs.rs/jsonwebtoken
  - Focus: Token generation, claims, expiration
- [ ] **argon2**: Password hashing
  - Resource: https://docs.rs/argon2
  - Focus: Secure password storage
- [ ] **async-trait**: Async trait methods
  - Resource: https://docs.rs/async-trait

**Patterns:**
- [ ] Repository pattern (traits)
- [ ] Dependency injection
- [ ] Error types (custom errors)
- [ ] Middleware (extractors in Axum)

**Specific for This Week:**
- [ ] Password security in Rust
- [ ] JWT claims structure
- [ ] Authentication middleware
- [ ] Database repositories

---

**Learn Objectives (8h breakdown):**

| Day | Concept | Time | Resource |
|-----|---------|------|----------|
| Mon | Traits + generic bounds | 1.5h | Rust Book Ch. 10 |
| Tue | argon2 + password hashing | 1.5h | argon2 docs |
| Wed | jsonwebtoken basics | 2h | JWT docs + examples |
| Thu | Repository pattern | 1.5h | Design patterns |
| Fri | Axum extractors | 1.5h | Axum docs |

---

**Build Objectives (30-32h)**

**Monday (2h learning + 2h build)**
- Learn: Traits, repository pattern
- Build: UserRepository trait, SQLx implementation
- Output: Clean architecture, testable design
- Rust focus:
  ```rust
  #[async_trait]
  pub trait UserRepository: Send + Sync {
      async fn create(&self, email: String, name: String) -> Result<User>;
      async fn get_by_email(&self, email: &str) -> Result<User>;
  }
  
  pub struct SqlxUserRepository {
      pool: PgPool,
  }
  
  #[async_trait]
  impl UserRepository for SqlxUserRepository {
      async fn create(&self, email: String, name: String) -> Result<User> {
          // INSERT query
      }
  }
  ```
- GitHub: `feat(user): Domain + repository trait`

**Tuesday (1.5h learning + 2.5h build)**
- Learn: argon2, password security
- Build: Password hashing service
- Output: Secure password handling
- Rust focus:
  ```rust
  use argon2::{Argon2, PasswordHasher};
  use password_hash::SaltString;
  
  pub fn hash_password(password: &str) -> Result<String> {
      let salt = SaltString::generate(OsRng);
      let argon2 = Argon2::default();
      let hash = argon2
          .hash_password(password.as_bytes(), &salt)?
          .to_string();
      Ok(hash)
  }
  ```
- GitHub: `feat(user): Password hashing + security`

**Wednesday (1h learning + 4h build)**
- Learn: JWT, jsonwebtoken
- Build: JWT service, token generation/verification
- Output: Tokens work end-to-end
- Rust focus:
  ```rust
  use jsonwebtoken::{encode, decode, Header, Key, TokenData};
  
  #[derive(Serialize, Deserialize)]
  pub struct Claims {
      pub sub: String,  // user_id
      pub exp: i64,     // expiration
  }
  
  pub fn generate_token(user_id: &str) -> Result<String> {
      let claims = Claims {
          sub: user_id.to_string(),
          exp: (Utc::now() + Duration::hours(24)).timestamp(),
      };
      encode(&Header::default(), &claims, &KEY)
  }
  ```
- GitHub: `feat(user): JWT token management`

**Thursday (1.5h learning + 3.5h build)**
- Learn: Axum extractors, middleware
- Build: REST endpoints (register, login)
- Output: Users can register/login, get tokens
- Rust focus:
  ```rust
  // Axum extractor for JWT
  pub struct AuthUser {
      pub user_id: String,
  }
  
  #[async_trait]
  impl<S> FromRequestParts<S> for AuthUser
  where
      S: Send + Sync,
  {
      type Rejection = AuthError;
      
      async fn from_request_parts(
          parts: &mut Parts,
          _state: &S,
      ) -> Result<Self, Self::Rejection> {
          let token = extract_bearer_token(&parts.headers)?;
          let claims = verify_jwt(&token)?;
          Ok(AuthUser {
              user_id: claims.sub,
          })
      }
  }
  ```
- GitHub: `feat(user): REST endpoints (register/login)`

**Friday (1h learning + 3h build)**
- Learn: Integration testing with databases
- Build: Full test suite
- Output: All flows tested
- Rust focus:
  ```rust
  #[tokio::test]
  async fn test_register_login_flow() {
      let pool = setup_test_db().await;
      let repo = SqlxUserRepository::new(pool);
      
      // Register
      let user = repo.create("test@example.com".into(), "Test".into()).await.unwrap();
      
      // Login
      let token = generate_token(&user.id).await.unwrap();
      let claims = verify_token(&token).unwrap();
      
      assert_eq!(claims.sub, user.id.to_string());
  }
  ```
- GitHub: `feat(week3): User service complete`

**Deliverables**
- User Service (~/1000 LOC)
- Auth flow complete
- Tests: 85%+ coverage
- GitHub: 5+ commits

---

### WEEK 4: Product Service (Search + Caching)

#### 🦀 RUST TOPICS TO STUDY (6 hours)

**Key Concepts:**
- [ ] String types (String vs &str)
- [ ] Collections (HashMap, Vec)
- [ ] Iterator adapters (map, filter, collect)
- [ ] Dereferencing and smart pointers

**Crates:**
- [ ] **redis**: Redis client
  - Resource: https://docs.rs/redis
  - Focus: Basic operations (GET, SET)
- [ ] **tokio-util**: Utilities for Tokio
  - Resource: https://docs.rs/tokio-util

**Patterns:**
- [ ] Cache-aside pattern
- [ ] Search indexing
- [ ] Connection pooling

**Specific for This Week:**
- [ ] Full-text search in PostgreSQL
- [ ] Redis caching strategy
- [ ] Cache invalidation

---

**Learn Objectives (6h breakdown):**

| Day | Concept | Time | Resource |
|-----|---------|------|----------|
| Mon | PostgreSQL full-text search | 1.5h | PG docs + tutorial |
| Tue | Redis basics | 1.5h | Redis crate docs |
| Wed | Cache-aside pattern | 1h | Design patterns |
| Thu | String searching | 1h | Rust Book Ch. 8 |
| Fri | Iterator patterns | 1h | Rust Book Ch. 13 |

---

**Build Objectives (30-32h)**

**Mon-Tue (5h learning + 8h build)**
- Learn: Full-text search, Redis
- Build: Product repository with search
- Output: Search working, caching functional
- Rust focus:
  ```rust
  // Full-text search in SQL
  pub async fn search_products(
      db: &PgPool,
      query: &str,
      limit: i64,
  ) -> Result<Vec<Product>> {
      sqlx::query_as::<_, Product>(
          "SELECT id, name, description, price, stock, created_at
          FROM products
          WHERE to_tsvector('english', name || ' ' || description) @@ 
                plainto_tsquery('english', $1)
          LIMIT $2"
      )
      .bind(query)
      .bind(limit)
      .fetch_all(db)
      .await
  }
  ```
- GitHub: `feat(product): Full-text search implementation`

**Wed-Thu (2h learning + 9h build)**
- Learn: Redis caching patterns
- Build: Cache layer for products
- Output: Cache hits visible in metrics
- Rust focus:
  ```rust
  // Cache-aside pattern
  pub async fn get_product_cached(
      id: &str,
      db: &PgPool,
      redis: &redis::Connection,
  ) -> Result<Product> {
      // Try cache first
      if let Ok(cached) = redis.get::<_, String>(format!("product:{}", id)) {
          return serde_json::from_str(&cached).map_err(|e| e.into());
      }
      
      // DB fallback
      let product = get_product_from_db(db, id).await?;
      
      // Update cache
      redis.set_ex(
          format!("product:{}", id),
          serde_json::to_string(&product)?,
          3600,  // 1 hour TTL
      )?;
      
      Ok(product)
  }
  ```
- GitHub: `feat(product): Redis caching layer`

**Fri (1h learning + 5h build)**
- Learn: Integration testing caching
- Build: Tests for search + cache
- Output: All tests pass
- GitHub: `feat(week4): Product service complete`

**Deliverables**
- Product Service (~/1200 LOC)
- Full-text search
- Redis caching with metrics
- GitHub: 5+ commits

---

### WEEK 5: Cart Service (Redis + PostgreSQL)

#### 🦀 RUST TOPICS TO STUDY (6 hours)

**Key Concepts:**
- [ ] HashMap operations
- [ ] Serde for complex types
- [ ] TTL management patterns
- [ ] Serialization/deserialization

**Crates:**
- [ ] **serde_json**: JSON operations
- [ ] **chrono**: TTL calculations

---

**Build Objectives (30-32h)**

Output: Cart fast access proven
GitHub: 5+ commits

**Deliverables**
- Cart Service (~/1000 LOC)
- Redis fast access + PostgreSQL persistence

---

## SPRINT 2: ORDERS & PAYMENT (WEEKS 6-9)

### WEEK 6: Order Service

#### 🦀 RUST TOPICS TO STUDY (8 hours)

**Key Concepts:**
- [ ] Enum state machines
- [ ] Event sourcing patterns
- [ ] Message passing in async code
- [ ] Builder pattern

**Crates:**
- [ ] **tokio-stream**: Stream utilities
- [ ] **futures**: Async utilities

---

**Build Objectives (30-32h)**

Output: Orders working, state machine proven
GitHub: 5+ commits

**Deliverables**
- Order Service (~/1500 LOC)
- State machine + event sourcing

---

### WEEK 7: Webhook Management

#### 🦀 RUST TOPICS TO STUDY (6 hours)

**Topics:**
- [ ] Exponential backoff algorithms
- [ ] Retry logic patterns
- [ ] Concurrent task management (tokio::spawn)
- [ ] Queue data structures

---

**Build Objectives (30-32h)**

Output: Webhooks reliable, no duplicates
GitHub: 4+ commits

**Deliverables**
- Retry logic (~/200 LOC)
- DLQ + idempotency

---

### WEEK 8: Security Hardening

#### 🦀 RUST TOPICS TO STUDY (6 hours)

**Topics:**
- [ ] Rate limiting algorithms
- [ ] Input validation patterns
- [ ] Secrets management (environment variables)
- [ ] CORS and security headers

**Crates:**
- [ ] **governor**: Rate limiting
- [ ] **validator**: Input validation

---

**Build Objectives (30-32h)**

Output: All security checks in place
GitHub: 4+ commits

**Deliverables**
- Security layer (~/300 LOC)
- Rate limiting + validation

---

### WEEK 9: Resilience Patterns

#### 🦀 RUST TOPICS TO STUDY (8 hours)

**Key Concepts:**
- [ ] State machines (circuit breaker states)
- [ ] Tokio tasks and channels
- [ ] Arc and Mutex for shared state
- [ ] Error propagation patterns

**Crates:**
- [ ] **tokio::sync::Mutex**: Async-safe mutex
- [ ] **tokio::time**: Timeouts and delays

**Learning Resources:**
- Tokio Sync: https://docs.rs/tokio/latest/tokio/sync/
- Tokio Time: https://docs.rs/tokio/latest/tokio/time/

---

**Learn Objectives (8h breakdown):**

| Day | Concept | Time | Resource |
|-----|---------|------|----------|
| Mon | Circuit breaker pattern | 2h | Design patterns + Tokio docs |
| Tue | State machines in Rust | 1.5h | Enum-based state pattern |
| Wed | Async timeouts | 1.5h | Tokio time module |
| Thu | Exponential backoff | 1h | Algorithm study |
| Fri | Error handling in resilience | 1.5h | Result chaining |

---

**Build Objectives (30-32h)**

**Mon-Tue (6h learning + 8h build)**
- Learn: Circuit breaker, state machines
- Build: CircuitBreaker struct with states (Closed, Open, HalfOpen)
- Output: Circuit breaker working
- Rust focus:
  ```rust
  #[derive(Debug, Clone, Copy, PartialEq)]
  pub enum CircuitState {
      Closed,
      Open,
      HalfOpen,
  }
  
  pub struct CircuitBreaker {
      state: Arc<Mutex<CircuitState>>,
      failure_count: Arc<AtomicU32>,
      last_failure_time: Arc<Mutex<Option<Instant>>>,
  }
  
  impl CircuitBreaker {
      pub async fn call<F, T>(&self, f: F) -> Result<T>
      where
          F: Future<Output = Result<T>>,
      {
          let mut state = self.state.lock().await;
          
          match *state {
              CircuitState::Closed => {
                  match f.await {
                      Ok(result) => Ok(result),
                      Err(e) => {
                          *state = CircuitState::Open;
                          Err(e)
                      }
                  }
              },
              CircuitState::Open => {
                  Err("Circuit is open".into())
              },
              CircuitState::HalfOpen => {
                  match f.await {
                      Ok(result) => {
                          *state = CircuitState::Closed;
                          Ok(result)
                      },
                      Err(e) => {
                          *state = CircuitState::Open;
                          Err(e)
                      }
                  }
              }
          }
      }
  }
  ```
- GitHub: `feat(resilience): Circuit breaker pattern`

**Wed (2h learning + 6h build)**
- Learn: Tokio timeouts, exponential backoff
- Build: Retry logic with timeout
- Output: Retries working with backoff
- Rust focus:
  ```rust
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
      
      Err("Max attempts exceeded".into())
  }
  ```
- GitHub: `feat(resilience): Retry with exponential backoff`

**Thu-Fri (1h learning + 6h build)**
- Learn: Timeout management, fallbacks
- Build: Timeout wrapper, graceful degradation
- Output: All resilience patterns working
- Rust focus:
  ```rust
  pub async fn call_with_timeout<F, T>(
      future: F,
      timeout_duration: Duration,
  ) -> Result<T>
  where
      F: Future<Output = Result<T>>,
  {
      match tokio::time::timeout(timeout_duration, future).await {
          Ok(Ok(result)) => Ok(result),
          Ok(Err(e)) => Err(e),
          Err(_) => Err("Operation timed out".into()),
      }
  }
  ```
- GitHub: `feat(resilience): Timeout management`

**Deliverables**
- Circuit breaker (~/250 LOC)
- Timeout management
- Fallback strategies
- Tests: failure scenarios proven
- GitHub: 5+ commits

---

## SPRINT 3: OBSERVABILITY & DEPLOYMENT (WEEKS 10-12)

### WEEK 10: OpenTelemetry + Logs

#### 🦀 RUST TOPICS TO STUDY (5 hours)

**Key Concepts:**
- [ ] Macro-based instrumentation (#[instrument])
- [ ] Structured logging patterns
- [ ] Context propagation
- [ ] Spans and events

**Crates:**
- [ ] **tracing**: Structured logging
- [ ] **tracing-subscriber**: Log subscribers
- [ ] **tracing-opentelemetry**: OTLP export

**Learning Resources:**
- Tracing Book: https://docs.rs/tracing
- Tokio Tracing: https://tokio.rs/tokio/topics/tracing

---

**Learn Objectives (5h breakdown):**

| Day | Concept | Time | Resource |
|-----|---------|------|----------|
| Mon | Tracing macros | 1h | Tracing docs |
| Tue | Structured logging | 1h | Tracing examples |
| Wed | Instrumentation | 1.5h | #[instrument] macro |
| Thu-Fri | OpenTelemetry | 1.5h | tracing-opentelemetry docs |

---

**Build Objectives (5h)**

All 11 Docker services already include OTEL collector:
- Jaeger (traces) ✅
- Prometheus (metrics) ✅
- Loki (logs) ✅
- Grafana (dashboards) ✅

Your job:
- **Mon**: Add #[instrument] macros to handlers (1h)
- **Tue**: Add structured logging with tracing::info! (1h)
- **Wed**: Setup OpenTelemetry export (1h)
- **Thu-Fri**: View in Grafana/Jaeger (2h)

**Rust focus:**
```rust
use tracing::{info, error, instrument};

#[instrument(skip(pool))]
pub async fn create_payment(
    pool: &PgPool,
    amount: Decimal,
) -> Result<Payment> {
    info!("Creating payment for amount: {}", amount);
    
    match payment_service.create(amount).await {
        Ok(payment) => {
            info!("Payment created: {:?}", payment.id);
            Ok(payment)
        },
        Err(e) => {
            error!("Payment creation failed: {}", e);
            Err(e)
        }
    }
}
```

Output: Logs queryable in Loki, traces in Jaeger, metrics in Prometheus
GitHub: 3+ commits

**Deliverables**
- Instrumentation (~/100 LOC)
- Working dashboards in Grafana

---

### WEEK 11: Load Testing + Optimization

#### 🦀 RUST TOPICS TO STUDY (4 hours)

**Key Concepts:**
- [ ] Performance profiling
- [ ] Benchmarking in Rust
- [ ] Query optimization

**Tools:**
- [ ] **criterion**: Benchmarking framework
- [ ] **flamegraph**: Performance profiling

---

**Build Objectives (4h)**

- **Mon**: Load test setup (1h)
- **Tue-Wed**: Run tests, analyze (2h)
- **Thu-Fri**: Optimize, re-test (1h)

```bash
# Tool already installed
wrk -t4 -c100 -d60s http://localhost:3000/products
```

Output: 3000+ req/sec proven
GitHub: 3+ commits

---

### WEEK 12: Documentation + Final

#### 🦀 RUST TOPICS TO STUDY (2 hours)

**Key Concepts:**
- [ ] Doc comments (///)
- [ ] OpenAPI/Swagger specifications
- [ ] Module documentation

**Crates:**
- [ ] **utoipa**: OpenAPI auto-generation
- [ ] **utoipa-swagger-ui**: Swagger UI

---

**Build Objectives (5h)**

**Mon (1h):** Code docs + setup utoipa
**Tue (1.5h):** OpenAPI spec auto-generated
**Wed-Thu (1.5h):** Swagger UI endpoint working
**Fri (1h):** Demo prep + final polish

---

**API Endpoints (lo que falta documentar):**

```
# AUTH
POST   /auth/register     → { user_id, token }
POST   /auth/login        → { user_id, token }

# PRODUCTS
GET    /products?query=X&limit=10
GET    /products/{id}

# CART
POST   /cart/items        (auth)
GET    /cart              (auth)
DELETE /cart/items/{id}   (auth)

# ORDERS
POST   /orders            (auth)
GET    /orders            (auth)
GET    /orders/{id}       (auth)

# PAYMENTS
POST   /payments          (auth)
POST   /payments/webhook  (stripe-sig)
GET    /payments/{id}     (auth)

# ADMIN
GET    /admin/ledger/balance     (admin)
GET    /admin/ledger/entries     (admin)
```

**Status Codes:**
- 200: OK | 201: Created | 400: Bad Request | 401: Unauthorized | 404: Not Found | 409: Conflict | 429: Rate Limited

**Headers:**
```
Authorization: Bearer <JWT>
X-RateLimit-Limit: 1000
X-Stripe-Signature: t=...,v1=...
```

Output: Swagger UI at `http://localhost:3000/swagger-ui` | Portfolio-ready
GitHub: 3+ commits

## SPRINT 4: FINAL POLISH (WEEKS 13-15)

### WEEK 13: Advanced Features

#### 🦀 RUST TOPICS TO STUDY (6 hours)

**Topics:**
- [ ] Advanced caching strategies
- [ ] Performance optimization techniques
- [ ] Concurrent testing

---

**Build Objectives (35h)**

Output: 3000+ req/sec sustained
GitHub: 5+ commits

---

### WEEK 14: Security Audit + Hardening

#### 🦀 RUST TOPICS TO STUDY (4 hours)

**Topics:**
- [ ] Security best practices
- [ ] Dependency auditing
- [ ] OWASP patterns in Rust

---

**Build Objectives (35h)**

Output: Production-ready
GitHub: 4+ commits

---

### WEEK 15: Final Integration + Portfolio

#### 🦀 RUST TOPICS TO STUDY (2 hours)

**Topics:**
- [ ] Final optimization techniques

---

**Build Objectives (35h)**

Output: Portfolio ready
GitHub: 5+ commits

---

## FINAL METRICS

### Code Quality
- 75+ commits on GitHub
- 7-9K lines of production code
- 90%+ test coverage
- Zero compiler warnings

### Performance
- 3000+ req/sec local
- P50 latency: <50ms
- P99 latency: <100ms

### Portfolio Items
1. GitHub: 75+ commits, clean history
2. Microservices: 5 services
3. Stripe: Full integration
4. Observability: Grafana + Jaeger + Prometheus
5. Resilience: Circuit breaker, retry, timeout
6. Performance: 3000+ req/sec proof
7. Security: OWASP audit complete
8. Docker: Production-ready docker-compose.yml

---

## SUMMARY

**Timeline:**
- Friday before Week 1: Setup Docker (1 hour)
- Week 1 Monday: `docker-compose up -d` + `cargo run`
- Week 15 Friday: Portfolio ready
- Month 5: Job offer $180-200K

**What You Build:**
- Production marketplace system
- 75+ commits
- 7-9K lines Rust code
- Full-stack backend

**GitHub Story:**
"I built a production payment marketplace in Rust: 5 microservices, Stripe integration, double-entry ledger, 3000+ req/sec, production observability. Developed locally with Docker Compose (no Kubernetes). Mastered: Axum, async/await, SQLx, error handling, resilience patterns, OpenTelemetry. 75+ commits over 15 weeks."

**That gets you hired at $180K+** 🎯
