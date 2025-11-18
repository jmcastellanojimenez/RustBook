# 🦀 GUÍA DE ESTUDIO RUST - SPRINT 0

---

## WEEK 1: Async/Await, Traits, Error Handling, Axum, SQLx

**Total estudio:** 12 horas

### DAY 1 (Monday): Axum + Serde (3h)

**Axum Routing (1.5h)**
```rust
use axum::{routing::{get, post}, Router};

async fn health_check() -> &'static str {
    "OK"
}

#[tokio::main]
async fn main() {
    let app = Router::new()
        .route("/health", get(health_check));
    
    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

**Serde JSON (1.5h)**
```rust
use serde::{Deserialize, Serialize};
use axum::Json;

#[derive(Deserialize)]
pub struct CreatePaymentRequest {
    pub amount: String,
}

#[derive(Serialize)]
pub struct PaymentResponse {
    pub id: String,
    pub status: String,
}

async fn create_payment(Json(req): Json<CreatePaymentRequest>) -> Json<PaymentResponse> {
    Json(PaymentResponse {
        id: "payment_123".into(),
        status: "pending".into(),
    })
}
```

**Recursos:**
- https://docs.rs/axum/latest/axum/
- https://serde.rs

**Checkpoint:**
- [ ] Crear Router con 2 rutas
- [ ] Extraer JSON de request
- [ ] Retornar JSON response

---

### DAY 2 (Tuesday): SQLx + Async Queries (3h)

**Connection Pool (1h)**
```rust
use sqlx::PgPool;

pub async fn create_pool() -> Result<PgPool> {
    let pool = PgPool::connect("postgres://user:pass@localhost/db").await?;
    Ok(pool)
}
```

**Async Queries (2h)**
```rust
use sqlx::FromRow;
use uuid::Uuid;

#[derive(FromRow)]
pub struct Payment {
    pub id: Uuid,
    pub amount: Decimal,
    pub status: String,
}

pub async fn get_payment(pool: &PgPool, id: Uuid) -> Result<Payment> {
    sqlx::query_as::<_, Payment>("SELECT * FROM payments WHERE id = $1")
        .bind(id)
        .fetch_one(pool)
        .await
}

pub async fn create_payment(pool: &PgPool, amount: Decimal) -> Result<Payment> {
    sqlx::query_as::<_, Payment>(
        "INSERT INTO payments (id, amount, status) VALUES ($1, $2, $3) RETURNING *"
    )
    .bind(Uuid::new_v4())
    .bind(amount)
    .bind("pending")
    .fetch_one(pool)
    .await
}
```

**Recursos:**
- https://github.com/launchbadge/sqlx

**Checkpoint:**
- [ ] Connect to PostgreSQL
- [ ] SELECT query
- [ ] INSERT query with RETURNING

---

### DAY 3 (Wednesday): Error Handling + Custom Types (2h)

**Result<T, E> (1h)**
```rust
pub fn divide(a: f64, b: f64) -> Result<f64, String> {
    if b == 0.0 {
        Err("Division by zero".to_string())
    } else {
        Ok(a / b)
    }
}

// Usage
match divide(10.0, 2.0) {
    Ok(result) => println!("{}", result),
    Err(e) => eprintln!("Error: {}", e),
}
```

**Custom Error Types (1h)**
```rust
#[derive(Debug)]
pub enum PaymentError {
    InvalidAmount,
    DatabaseError(sqlx::Error),
    StripeError(String),
}

impl From<sqlx::Error> for PaymentError {
    fn from(err: sqlx::Error) -> Self {
        PaymentError::DatabaseError(err)
    }
}

pub async fn create_payment(pool: &PgPool) -> Result<Payment, PaymentError> {
    let payment = sqlx::query_as("SELECT * FROM payments")
        .fetch_one(pool)
        .await?;  // Converts sqlx::Error to PaymentError
    Ok(payment)
}
```

**Recursos:**
- Rust Book Ch. 9: https://doc.rust-lang.org/book/ch09-00-error-handling.html

**Checkpoint:**
- [ ] Definir custom error enum
- [ ] Implementar From trait
- [ ] Usar ? operator

---

### DAY 4 (Thursday): Traits + Async Traits (2h)

**Trait Definition (1h)**
```rust
#[async_trait]
pub trait PaymentRepository: Send + Sync {
    async fn get(&self, id: Uuid) -> Result<Payment>;
    async fn create(&self, amount: Decimal) -> Result<Payment>;
}
```

**Implementation (1h)**
```rust
use async_trait::async_trait;

pub struct SqlxPaymentRepository {
    pool: PgPool,
}

#[async_trait]
impl PaymentRepository for SqlxPaymentRepository {
    async fn get(&self, id: Uuid) -> Result<Payment> {
        sqlx::query_as("SELECT * FROM payments WHERE id = $1")
            .bind(id)
            .fetch_one(&self.pool)
            .await
    }
    
    async fn create(&self, amount: Decimal) -> Result<Payment> {
        // INSERT logic
    }
}
```

**Recursos:**
- Rust Book Ch. 10: https://doc.rust-lang.org/book/ch10-00-generics.html
- async-trait: https://docs.rs/async-trait

**Checkpoint:**
- [ ] Definir trait con async methods
- [ ] Implementar trait para struct
- [ ] Usar trait bound

---

### DAY 5 (Friday): Tokio + Async/Await (2h)

**Async Runtime (1h)**
```rust
#[tokio::main]
async fn main() {
    let result = my_async_function().await;
    println!("{}", result);
}

async fn my_async_function() -> String {
    tokio::time::sleep(Duration::from_secs(1)).await;
    "Done".to_string()
}
```

**Spawning Tasks (1h)**
```rust
use tokio::task;

#[tokio::main]
async fn main() {
    let handle = task::spawn(async {
        println!("Running in background");
    });
    
    handle.await.unwrap();
}
```

**Recursos:**
- https://tokio.rs/tokio/tutorial

**Checkpoint:**
- [ ] Entender #[tokio::main]
- [ ] Usar .await
- [ ] Spawn async task

---

## WEEK 2: Enums, Pattern Matching, Collections, Iterators, Testing

**Total estudio:** 8 horas

### DAY 1 (Monday): Enums + Pattern Matching (2h)

**Enum Basics (1h)**
```rust
#[derive(Debug, Clone, Copy)]
pub enum Direction {
    Debit,
    Credit,
}

#[derive(Debug)]
pub enum TransactionStatus {
    Pending,
    Confirmed,
    Failed(String),
}
```

**Pattern Matching (1h)**
```rust
pub fn calculate_impact(direction: Direction, amount: Decimal) -> Decimal {
    match direction {
        Direction::Debit => -amount,
        Direction::Credit => amount,
    }
}

// With Option
pub fn get_balance(maybe_balance: Option<Decimal>) -> Decimal {
    match maybe_balance {
        Some(balance) => balance,
        None => Decimal::ZERO,
    }
}
```

**Recursos:**
- Rust Book Ch. 6: https://doc.rust-lang.org/book/ch06-00-enums.html

**Checkpoint:**
- [ ] Definir enum con variants
- [ ] Match exhaustivo
- [ ] if-let shorthand

---

### DAY 2 (Tuesday): Decimal + Financial Math (1.5h)

**Decimal Type (1.5h)**
```rust
use rust_decimal::Decimal;
use rust_decimal_macros::dec;

let amount = dec!(100.50);
let fee = dec!(2.5);
let total = amount + fee;  // dec!(103.00)

// NO USAR f64 para dinero
let wrong = 0.1 + 0.2;  // 0.30000000000000004 ❌
let correct = dec!(0.1) + dec!(0.2);  // 0.3 ✅
```

**Recursos:**
- https://docs.rs/rust_decimal

**Checkpoint:**
- [ ] Usar dec! macro
- [ ] Arithmetic con Decimal
- [ ] Entender por qué no f64

---

### DAY 3 (Wednesday): Collections (2h)

**Vec (1h)**
```rust
let mut entries = Vec::new();
entries.push(LedgerEntry { direction: Debit, amount: dec!(100) });

// Iteration
for entry in &entries {
    println!("{:?}", entry);
}

// Filter
let debits: Vec<_> = entries.iter()
    .filter(|e| matches!(e.direction, Direction::Debit))
    .collect();
```

**HashMap (1h)**
```rust
use std::collections::HashMap;

let mut balances: HashMap<Uuid, Decimal> = HashMap::new();
balances.insert(user_id, dec!(1000));

// Lookup
if let Some(balance) = balances.get(&user_id) {
    println!("{}", balance);
}
```

**Checkpoint:**
- [ ] Vec push/iter
- [ ] HashMap insert/get
- [ ] Filter + collect

---

### DAY 4 (Thursday): Iterators + Functional Patterns (1.5h)

**Iterator Chaining (1.5h)**
```rust
let total: Decimal = entries.iter()
    .filter(|e| matches!(e.direction, Direction::Debit))
    .map(|e| e.amount)
    .sum();

// Fold
let balance = entries.iter().fold(Decimal::ZERO, |acc, entry| {
    match entry.direction {
        Direction::Debit => acc - entry.amount,
        Direction::Credit => acc + entry.amount,
    }
});
```

**Recursos:**
- Rust Book Ch. 13: https://doc.rust-lang.org/book/ch13-00-functional-features.html

**Checkpoint:**
- [ ] Chain map/filter/collect
- [ ] Usar fold
- [ ] Calcular totals

---

### DAY 5 (Friday): Unit Testing (1h)

**Test Module (1h)**
```rust
#[cfg(test)]
mod tests {
    use super::*;
    
    #[test]
    fn test_balanced_ledger() {
        let entries = vec![
            LedgerEntry { direction: Debit, amount: dec!(100) },
            LedgerEntry { direction: Credit, amount: dec!(100) },
        ];
        
        let debits: Decimal = entries.iter()
            .filter(|e| matches!(e.direction, Direction::Debit))
            .map(|e| e.amount)
            .sum();
        
        let credits: Decimal = entries.iter()
            .filter(|e| matches!(e.direction, Direction::Credit))
            .map(|e| e.amount)
            .sum();
        
        assert_eq!(debits, credits);
    }
}
```

**Checkpoint:**
- [ ] Escribir test function
- [ ] assert_eq!
- [ ] cargo test

---

**FIN SPRINT 0**

---

# 🦀 GUÍA DE ESTUDIO RUST - SPRINT 1: E-COMMERCE CORE

---

## WEEK 3: User Service (Auth + JWT)

**Total estudio:** 8 horas

### DAY 1 (Monday): Traits + Generic Programming (2h)

**Trait Definition (1h)**
```rust
#[async_trait]
pub trait UserRepository: Send + Sync {
    async fn get_by_email(&self, email: &str) -> Result<User>;
    async fn create(&self, email: String, name: String) -> Result<User>;
}
```

**Generic Bounds (1h)**
```rust
pub struct UserService<R: UserRepository> {
    repo: R,
}

impl<R: UserRepository> UserService<R> {
    pub async fn get_user(&self, email: &str) -> Result<User> {
        self.repo.get_by_email(email).await
    }
}
```

**Recursos:**
- Rust Book Ch. 10: https://doc.rust-lang.org/book/ch10-00-generics.html

**Checkpoint:**
- [ ] Definir trait con múltiples métodos
- [ ] Usar trait bounds con generics
- [ ] Inyectar dependencias vía traits

---

### DAY 2 (Tuesday): Password Hashing (Argon2) (1.5h)

**Argon2 Basics (1.5h)**
```rust
use argon2::{Argon2, PasswordHasher, PasswordHash, PasswordVerifier};
use password_hash::{SaltString, rand_core::OsRng};

pub fn hash_password(password: &str) -> Result<String> {
    let salt = SaltString::generate(OsRng);
    let argon2 = Argon2::default();
    
    let hash = argon2
        .hash_password(password.as_bytes(), &salt)?
        .to_string();
    
    Ok(hash)
}

pub fn verify_password(password: &str, hash: &str) -> Result<()> {
    let parsed_hash = PasswordHash::new(hash)?;
    Argon2::default().verify_password(password.as_bytes(), &parsed_hash)?;
    Ok(())
}
```

**Recursos:**
- https://docs.rs/argon2

**Checkpoint:**
- [ ] Hash password con salt
- [ ] Verify password contra hash
- [ ] Entender por qué Argon2 (no bcrypt/md5)

---

### DAY 3 (Wednesday): JWT Tokens (jsonwebtoken) (2h)

**Claims Structure (1h)**
```rust
use jsonwebtoken::{encode, decode, Header, EncodingKey, DecodingKey, Validation};
use serde::{Deserialize, Serialize};
use chrono::{Utc, Duration};

#[derive(Serialize, Deserialize)]
pub struct Claims {
    pub sub: String,      // user_id
    pub exp: i64,         // expiration
    pub iat: i64,         // issued at
}
```

**Token Generation/Verification (1h)**
```rust
pub fn generate_token(user_id: &str) -> Result<String> {
    let now = Utc::now();
    let claims = Claims {
        sub: user_id.to_string(),
        exp: (now + Duration::hours(24)).timestamp(),
        iat: now.timestamp(),
    };
    
    let token = encode(
        &Header::default(),
        &claims,
        &EncodingKey::from_secret("secret".as_ref()),
    )?;
    
    Ok(token)
}

pub fn verify_token(token: &str) -> Result<Claims> {
    let data = decode::<Claims>(
        token,
        &DecodingKey::from_secret("secret".as_ref()),
        &Validation::default(),
    )?;
    
    Ok(data.claims)
}
```

**Recursos:**
- https://docs.rs/jsonwebtoken
- JWT.io (debugger)

**Checkpoint:**
- [ ] Crear token con claims
- [ ] Verificar firma del token
- [ ] Validar expiración
- [ ] Extraer claims del token

---

### DAY 4 (Thursday): Axum Extractors + Middleware (1.5h)

**Custom Extractor (1.5h)**
```rust
use axum::{
    async_trait,
    extract::FromRequestParts,
    http::{request::Parts, StatusCode},
};

pub struct AuthUser {
    pub user_id: Uuid,
}

#[async_trait]
impl<S> FromRequestParts<S> for AuthUser
where
    S: Send + Sync,
{
    type Rejection = (StatusCode, String);
    
    async fn from_request_parts(
        parts: &mut Parts,
        _state: &S,
    ) -> Result<Self, Self::Rejection> {
        let auth_header = parts
            .headers
            .get("Authorization")
            .ok_or((StatusCode::UNAUTHORIZED, "Missing token".into()))?;
        
        let token = auth_header
            .to_str()
            .map_err(|_| (StatusCode::UNAUTHORIZED, "Invalid token".into()))?
            .strip_prefix("Bearer ")
            .ok_or((StatusCode::UNAUTHORIZED, "Invalid format".into()))?;
        
        let claims = verify_token(token)
            .map_err(|_| (StatusCode::UNAUTHORIZED, "Invalid token".into()))?;
        
        Ok(AuthUser {
            user_id: Uuid::parse_str(&claims.sub)
                .map_err(|_| (StatusCode::UNAUTHORIZED, "Invalid user_id".into()))?,
        })
    }
}

// Usage in handler
async fn protected_route(AuthUser { user_id }: AuthUser) -> String {
    format!("Hello user: {}", user_id)
}
```

**Recursos:**
- Axum Extractors: https://docs.rs/axum/latest/axum/extract/

**Checkpoint:**
- [ ] Implementar FromRequestParts
- [ ] Extraer header Authorization
- [ ] Verificar JWT en extractor
- [ ] Usar extractor en handler

---

### DAY 5 (Friday): Integration Testing (1h)

**Auth Flow Test (1h)**
```rust
#[tokio::test]
async fn test_register_login_flow() {
    let pool = setup_test_db().await;
    let repo = SqlxUserRepository::new(pool);
    
    // Register
    let user = repo.create(
        "test@example.com".into(),
        "Test User".into(),
        "password123".into()
    ).await.unwrap();
    
    assert_eq!(user.email, "test@example.com");
    
    // Login
    let token = generate_token(&user.id.to_string()).unwrap();
    let claims = verify_token(&token).unwrap();
    
    assert_eq!(claims.sub, user.id.to_string());
}

#[tokio::test]
async fn test_protected_endpoint() {
    let app = create_test_app().await;
    
    // Without token
    let response = app.get("/auth/me").await;
    assert_eq!(response.status(), 401);
    
    // With token
    let token = generate_token("user_id").unwrap();
    let response = app
        .get("/auth/me")
        .header("Authorization", format!("Bearer {}", token))
        .await;
    
    assert_eq!(response.status(), 200);
}
```

**Checkpoint:**
- [ ] Test register flow
- [ ] Test login flow
- [ ] Test protected endpoint (con/sin token)
- [ ] Verificar password hashing

---

## WEEK 4: Product Service (Search + Caching)

**Total estudio:** 6 horas

### DAY 1 (Monday): PostgreSQL Full-Text Search (1.5h)

**tsvector + tsquery (1.5h)**
```rust
pub async fn search_products(
    pool: &PgPool,
    query: &str,
    limit: i64,
) -> Result<Vec<Product>> {
    sqlx::query_as::<_, Product>(
        "SELECT id, name, description, price, stock, created_at
         FROM products
         WHERE to_tsvector('english', name || ' ' || description) @@ 
               plainto_tsquery('english', $1)
         ORDER BY ts_rank(
             to_tsvector('english', name || ' ' || description),
             plainto_tsquery('english', $1)
         ) DESC
         LIMIT $2"
    )
    .bind(query)
    .bind(limit)
    .fetch_all(pool)
    .await
}
```

**Recursos:**
- PostgreSQL FTS: https://www.postgresql.org/docs/current/textsearch.html

**Checkpoint:**
- [ ] Crear GIN index para búsqueda
- [ ] Query con to_tsvector
- [ ] Ranking con ts_rank
- [ ] Entender 'english' dictionary

---

### DAY 2 (Tuesday): Redis Basics (1.5h)

**Connection + Basic Operations (1.5h)**
```rust
use redis::{Client, Commands, Connection};

pub fn create_redis_client() -> Result<Client> {
    let client = Client::open("redis://localhost:6379")?;
    Ok(client)
}

pub fn set_key(conn: &mut Connection, key: &str, value: &str) -> Result<()> {
    conn.set(key, value)?;
    Ok(())
}

pub fn get_key(conn: &mut Connection, key: &str) -> Result<Option<String>> {
    let value: Option<String> = conn.get(key)?;
    Ok(value)
}

pub fn set_with_ttl(conn: &mut Connection, key: &str, value: &str, ttl: usize) -> Result<()> {
    conn.set_ex(key, value, ttl)?;
    Ok(())
}
```

**Recursos:**
- https://docs.rs/redis

**Checkpoint:**
- [ ] Connect to Redis
- [ ] SET/GET operations
- [ ] TTL (expiration)
- [ ] DEL (invalidation)

---

### DAY 3 (Wednesday): Cache-Aside Pattern (1.5h)

**Cache-Aside Implementation (1.5h)**
```rust
pub async fn get_product_cached(
    id: Uuid,
    pool: &PgPool,
    redis: &mut redis::Connection,
) -> Result<Product> {
    let cache_key = format!("product:{}", id);
    
    // 1. Try cache first
    if let Ok(Some(cached)) = redis.get::<_, String>(&cache_key) {
        if let Ok(product) = serde_json::from_str::<Product>(&cached) {
            return Ok(product);
        }
    }
    
    // 2. Cache miss → DB fallback
    let product = sqlx::query_as::<_, Product>(
        "SELECT * FROM products WHERE id = $1"
    )
    .bind(id)
    .fetch_one(pool)
    .await?;
    
    // 3. Update cache (TTL 1 hour)
    let serialized = serde_json::to_string(&product)?;
    redis.set_ex(&cache_key, serialized, 3600)?;
    
    Ok(product)
}

pub async fn invalidate_cache(
    id: Uuid,
    redis: &mut redis::Connection,
) -> Result<()> {
    let cache_key = format!("product:{}", id);
    redis.del(&cache_key)?;
    Ok(())
}
```

**Checkpoint:**
- [ ] Try cache first
- [ ] Fallback to DB on miss
- [ ] Update cache after DB read
- [ ] Invalidate on write operations

---

### DAY 4 (Thursday): String Types (String vs &str) (1h)

**Ownership Rules (1h)**
```rust
// &str = borrowed, immutable
fn print_name(name: &str) {
    println!("{}", name);
}

// String = owned, mutable
fn take_ownership(name: String) {
    println!("{}", name);
}

// Conversion
let owned = String::from("hello");
let borrowed: &str = &owned;

// Redis cache key pattern
fn cache_key(id: &Uuid) -> String {
    format!("product:{}", id)  // Returns owned String
}

let key = cache_key(&product_id);
redis.set(&key, value)?;  // &key borrowing
```

**Checkpoint:**
- [ ] Entender String vs &str
- [ ] Saber cuándo usar cada uno
- [ ] format! retorna String
- [ ] Borrowing con &

---

### DAY 5 (Friday): Iterator Patterns (0.5h)

**Map/Filter/Collect (0.5h)**
```rust
let products = vec![
    Product { name: "Laptop".into(), price: dec!(1000) },
    Product { name: "Mouse".into(), price: dec!(50) },
];

// Filter expensive products
let expensive: Vec<_> = products.iter()
    .filter(|p| p.price > dec!(100))
    .collect();

// Extract names
let names: Vec<String> = products.iter()
    .map(|p| p.name.clone())
    .collect();

// Total price
let total: Decimal = products.iter()
    .map(|p| p.price)
    .sum();
```

**Checkpoint:**
- [ ] Chain iterators
- [ ] filter + collect
- [ ] map transformations
- [ ] sum() aggregation

---

## WEEK 5: Cart Service (Redis + PostgreSQL)

**Total estudio:** 6 horas

### DAY 1 (Monday): Serde Serialization (1.5h)

**Serialize/Deserialize Complex Types (1.5h)**
```rust
use serde::{Deserialize, Serialize};

#[derive(Debug, Serialize, Deserialize)]
pub struct Cart {
    pub id: Uuid,
    pub user_id: Uuid,
    pub items: Vec<CartItem>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, Serialize, Deserialize)]
pub struct CartItem {
    pub product_id: Uuid,
    pub quantity: i32,
    pub price: Decimal,
}

// To JSON
let cart = Cart { /* ... */ };
let json = serde_json::to_string(&cart)?;

// From JSON
let parsed: Cart = serde_json::from_str(&json)?;

// Redis storage
redis.set("cart:user123", json)?;
let retrieved: String = redis.get("cart:user123")?;
let cart: Cart = serde_json::from_str(&retrieved)?;
```

**Recursos:**
- https://serde.rs
- https://docs.rs/serde_json

**Checkpoint:**
- [ ] Derive Serialize + Deserialize
- [ ] to_string / from_str
- [ ] Serializar structs nested
- [ ] Store JSON in Redis

---

### DAY 2 (Tuesday): Redis TTL Management (1.5h)

**Expiration Patterns (1.5h)**
```rust
pub async fn save_cart_with_ttl(
    cart: &Cart,
    redis: &mut redis::Connection,
) -> Result<()> {
    let key = format!("cart:{}", cart.user_id);
    let json = serde_json::to_string(cart)?;
    
    // Set with 24h TTL
    redis.set_ex(&key, json, 86400)?;
    
    Ok(())
}

pub async fn extend_cart_ttl(
    user_id: Uuid,
    redis: &mut redis::Connection,
) -> Result<()> {
    let key = format!("cart:{}", user_id);
    
    // Extend TTL (touch pattern)
    redis.expire(&key, 86400)?;
    
    Ok(())
}

pub async fn get_cart_ttl(
    user_id: Uuid,
    redis: &mut redis::Connection,
) -> Result<i64> {
    let key = format!("cart:{}", user_id);
    let ttl: i64 = redis.ttl(&key)?;
    Ok(ttl)
}
```

**Checkpoint:**
- [ ] set_ex (SET + TTL atómico)
- [ ] expire (extend TTL)
- [ ] ttl (check remaining time)
- [ ] Entender 86400 = 24h

---

### DAY 3 (Wednesday): HashMap Operations (1h)

**Product Deduplication (1h)**
```rust
use std::collections::HashMap;

pub fn merge_cart_items(items: Vec<CartItem>) -> Vec<CartItem> {
    let mut map: HashMap<Uuid, CartItem> = HashMap::new();
    
    for item in items {
        map.entry(item.product_id)
            .and_modify(|existing| {
                existing.quantity += item.quantity;
            })
            .or_insert(item);
    }
    
    map.into_values().collect()
}

// Usage
let items = vec![
    CartItem { product_id: id1, quantity: 2, price: dec!(10) },
    CartItem { product_id: id1, quantity: 3, price: dec!(10) },  // Same product
];

let merged = merge_cart_items(items);
// Result: [CartItem { product_id: id1, quantity: 5, ... }]
```

**Checkpoint:**
- [ ] entry() API
- [ ] and_modify()
- [ ] or_insert()
- [ ] into_values() para Vec

---

### DAY 4 (Thursday): Database Transactions (1.5h)

**Atomic Operations (1.5h)**
```rust
pub async fn save_cart_to_postgres(
    cart: &Cart,
    pool: &PgPool,
) -> Result<()> {
    let mut tx = pool.begin().await?;
    
    // 1. Insert cart
    sqlx::query(
        "INSERT INTO carts (id, user_id, created_at)
         VALUES ($1, $2, $3)
         ON CONFLICT (user_id) DO UPDATE SET updated_at = NOW()"
    )
    .bind(cart.id)
    .bind(cart.user_id)
    .bind(cart.created_at)
    .execute(&mut *tx)
    .await?;
    
    // 2. Delete old items
    sqlx::query("DELETE FROM cart_items WHERE cart_id = $1")
        .bind(cart.id)
        .execute(&mut *tx)
        .await?;
    
    // 3. Insert new items
    for item in &cart.items {
        sqlx::query(
            "INSERT INTO cart_items (id, cart_id, product_id, quantity, price)
             VALUES ($1, $2, $3, $4, $5)"
        )
        .bind(Uuid::new_v4())
        .bind(cart.id)
        .bind(item.product_id)
        .bind(item.quantity)
        .bind(item.price)
        .execute(&mut *tx)
        .await?;
    }
    
    tx.commit().await?;
    Ok(())
}
```

**Recursos:**
- SQLx Transactions: https://docs.rs/sqlx/latest/sqlx/struct.Transaction.html

**Checkpoint:**
- [ ] pool.begin()
- [ ] tx.commit() / tx.rollback()
- [ ] Múltiples queries en transaction
- [ ] Error handling (auto rollback)

---

### DAY 5 (Friday): Integration Testing (0.5h)

**Cart Operations Test (0.5h)**
```rust
#[tokio::test]
async fn test_add_item_to_cart() {
    let pool = setup_test_db().await;
    let mut redis = setup_test_redis().await;
    
    let user_id = Uuid::new_v4();
    let product_id = Uuid::new_v4();
    
    // Add item
    add_item_to_cart(user_id, product_id, 2, &pool, &mut redis)
        .await
        .unwrap();
    
    // Verify in Redis
    let cart = get_cart(user_id, &pool, &mut redis).await.unwrap();
    assert_eq!(cart.items.len(), 1);
    assert_eq!(cart.items[0].quantity, 2);
    
    // Add same item again
    add_item_to_cart(user_id, product_id, 3, &pool, &mut redis)
        .await
        .unwrap();
    
    // Verify quantity merged
    let cart = get_cart(user_id, &pool, &mut redis).await.unwrap();
    assert_eq!(cart.items.len(), 1);
    assert_eq!(cart.items[0].quantity, 5);
}
```

**Checkpoint:**
- [ ] Test add item
- [ ] Test duplicate item (quantity merge)
- [ ] Test remove item
- [ ] Verify Redis + PostgreSQL sync

---

**FIN SPRINT 1**

**Resumen de Tópicos Estudiados:**
- Traits + Generic Programming
- Password Hashing (Argon2)
- JWT Tokens (generation/verification)
- Axum Extractors + Middleware
- PostgreSQL Full-Text Search
- Redis (cache-aside pattern)
- String types (String vs &str)
- Serde serialization
- TTL management
- HashMap operations
- Database transactions
- Integration testing

**Total horas estudio Sprint 1:** 20 horas (8h + 6h + 6h)

---

# 🦀 GUÍA DE ESTUDIO RUST - SPRINT 2: ORDERS & PAYMENT

---

## WEEK 6: Order Service (State Machines + Event Sourcing)

**Total estudio:** 8 horas

### DAY 1 (Monday): Enum State Machines (2h)

**State Machine with Enums (1h)**
```rust
#[derive(Debug, Clone, Copy, PartialEq, Serialize, Deserialize, sqlx::Type)]
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
    
    pub fn next_allowed_states(&self) -> Vec<OrderStatus> {
        match self {
            OrderStatus::Pending => vec![OrderStatus::Confirmed, OrderStatus::Cancelled],
            OrderStatus::Confirmed => vec![OrderStatus::Paid, OrderStatus::Cancelled],
            OrderStatus::Paid => vec![OrderStatus::Shipped],
            OrderStatus::Shipped => vec![OrderStatus::Delivered],
            _ => vec![],
        }
    }
}
```

**State Transitions (1h)**
```rust
pub async fn transition_order_status(
    pool: &PgPool,
    order_id: Uuid,
    new_status: OrderStatus,
) -> Result<Order> {
    // Get current order
    let order = get_order(pool, order_id).await?;
    
    // Validate transition
    if !order.status.can_transition_to(new_status) {
        return Err(Error::InvalidStateTransition {
            from: order.status,
            to: new_status,
        });
    }
    
    // Update status
    let updated = sqlx::query_as::<_, Order>(
        "UPDATE orders SET status = $1, updated_at = NOW() 
         WHERE id = $2 RETURNING *"
    )
    .bind(new_status)
    .bind(order_id)
    .fetch_one(pool)
    .await?;
    
    Ok(updated)
}
```

**Recursos:**
- Rust Book Ch. 6: https://doc.rust-lang.org/book/ch06-00-enums.html
- State Machine Pattern: https://hoverbear.org/blog/rust-state-machine-pattern/

**Checkpoint:**
- [ ] Definir OrderStatus enum con todos los estados
- [ ] Implementar can_transition_to()
- [ ] Validar transiciones inválidas
- [ ] Pattern matching exhaustivo para estados

---

### DAY 2 (Tuesday): Event Sourcing Basics (2h)

**Event Model (1h)**
```rust
#[derive(Debug, Serialize, Deserialize)]
pub struct OrderEvent {
    pub id: Uuid,
    pub order_id: Uuid,
    pub event_type: String,
    pub from_status: Option<OrderStatus>,
    pub to_status: OrderStatus,
    pub metadata: serde_json::Value,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, Serialize, Deserialize)]
pub enum OrderEventType {
    Created,
    StatusChanged,
    ItemAdded,
    ItemRemoved,
    Cancelled,
}

impl OrderEventType {
    pub fn as_str(&self) -> &str {
        match self {
            OrderEventType::Created => "created",
            OrderEventType::StatusChanged => "status_changed",
            OrderEventType::ItemAdded => "item_added",
            OrderEventType::ItemRemoved => "item_removed",
            OrderEventType::Cancelled => "cancelled",
        }
    }
}
```

**Recording Events (1h)**
```rust
pub async fn record_order_event(
    pool: &PgPool,
    order_id: Uuid,
    event_type: OrderEventType,
    from_status: Option<OrderStatus>,
    to_status: OrderStatus,
    metadata: serde_json::Value,
) -> Result<OrderEvent> {
    let event = sqlx::query_as::<_, OrderEvent>(
        "INSERT INTO order_events 
         (id, order_id, event_type, from_status, to_status, metadata, created_at)
         VALUES ($1, $2, $3, $4, $5, $6, NOW())
         RETURNING *"
    )
    .bind(Uuid::new_v4())
    .bind(order_id)
    .bind(event_type.as_str())
    .bind(from_status)
    .bind(to_status)
    .bind(metadata)
    .fetch_one(pool)
    .await?;
    
    Ok(event)
}

pub async fn get_order_history(
    pool: &PgPool,
    order_id: Uuid,
) -> Result<Vec<OrderEvent>> {
    let events = sqlx::query_as::<_, OrderEvent>(
        "SELECT * FROM order_events 
         WHERE order_id = $1 
         ORDER BY created_at ASC"
    )
    .bind(order_id)
    .fetch_all(pool)
    .await?;
    
    Ok(events)
}
```

**Recursos:**
- Event Sourcing Pattern: https://martinfowler.com/eaaDev/EventSourcing.html

**Checkpoint:**
- [ ] Crear tabla order_events
- [ ] Definir OrderEvent struct
- [ ] Record event en cada cambio de estado
- [ ] Query event history

---

### DAY 3 (Wednesday): Builder Pattern (2h)

**Order Builder (2h)**
```rust
pub struct OrderBuilder {
    user_id: Option<Uuid>,
    items: Vec<OrderItem>,
    total_amount: Decimal,
}

impl OrderBuilder {
    pub fn new() -> Self {
        Self {
            user_id: None,
            items: Vec::new(),
            total_amount: Decimal::ZERO,
        }
    }
    
    pub fn user_id(mut self, user_id: Uuid) -> Self {
        self.user_id = Some(user_id);
        self
    }
    
    pub fn add_item(mut self, product_id: Uuid, quantity: i32, price: Decimal) -> Self {
        let item = OrderItem {
            product_id,
            quantity,
            price,
        };
        self.total_amount += price * Decimal::from(quantity);
        self.items.push(item);
        self
    }
    
    pub fn build(self) -> Result<Order> {
        let user_id = self.user_id.ok_or(Error::MissingUserId)?;
        
        if self.items.is_empty() {
            return Err(Error::EmptyOrder);
        }
        
        Ok(Order {
            id: Uuid::new_v4(),
            user_id,
            items: self.items,
            total_amount: self.total_amount,
            status: OrderStatus::Pending,
            created_at: Utc::now(),
            updated_at: Utc::now(),
        })
    }
}

// Usage
let order = OrderBuilder::new()
    .user_id(user_id)
    .add_item(product_id1, 2, dec!(50.00))
    .add_item(product_id2, 1, dec!(100.00))
    .build()?;
```

**Recursos:**
- Builder Pattern: https://rust-unofficial.github.io/patterns/patterns/creational/builder.html

**Checkpoint:**
- [ ] Implementar OrderBuilder
- [ ] Métodos encadenables
- [ ] Validación en build()
- [ ] Calcular total_amount automáticamente

---

### DAY 4 (Thursday): Async Testing (1h)

**Integration Tests (1h)**
```rust
#[tokio::test]
async fn test_order_state_transitions() {
    let pool = setup_test_db().await;
    
    // Create order
    let order = create_order(&pool, user_id, items).await.unwrap();
    assert_eq!(order.status, OrderStatus::Pending);
    
    // Valid transition: Pending → Confirmed
    let order = transition_order_status(&pool, order.id, OrderStatus::Confirmed)
        .await
        .unwrap();
    assert_eq!(order.status, OrderStatus::Confirmed);
    
    // Invalid transition: Confirmed → Delivered (skip Paid/Shipped)
    let result = transition_order_status(&pool, order.id, OrderStatus::Delivered).await;
    assert!(result.is_err());
    
    // Valid transition: Confirmed → Paid
    let order = transition_order_status(&pool, order.id, OrderStatus::Paid)
        .await
        .unwrap();
    assert_eq!(order.status, OrderStatus::Paid);
}

#[tokio::test]
async fn test_event_sourcing() {
    let pool = setup_test_db().await;
    
    let order = create_order(&pool, user_id, items).await.unwrap();
    transition_order_status(&pool, order.id, OrderStatus::Confirmed).await.unwrap();
    transition_order_status(&pool, order.id, OrderStatus::Paid).await.unwrap();
    
    // Get event history
    let events = get_order_history(&pool, order.id).await.unwrap();
    assert_eq!(events.len(), 3); // created, confirmed, paid
    
    assert_eq!(events[0].event_type, "created");
    assert_eq!(events[1].from_status, Some(OrderStatus::Pending));
    assert_eq!(events[1].to_status, OrderStatus::Confirmed);
}
```

**Checkpoint:**
- [ ] Test state transitions válidas
- [ ] Test state transitions inválidas
- [ ] Test event recording
- [ ] Test event history query

---

### DAY 5 (Friday): Message Passing (1h)

**Tokio Channels for Events (1h)**
```rust
use tokio::sync::mpsc;

pub struct OrderEventPublisher {
    sender: mpsc::Sender<OrderEvent>,
}

impl OrderEventPublisher {
    pub fn new() -> (Self, mpsc::Receiver<OrderEvent>) {
        let (sender, receiver) = mpsc::channel(100);
        (Self { sender }, receiver)
    }
    
    pub async fn publish(&self, event: OrderEvent) -> Result<()> {
        self.sender.send(event).await
            .map_err(|_| Error::ChannelClosed)?;
        Ok(())
    }
}

// Background worker
pub async fn event_worker(mut receiver: mpsc::Receiver<OrderEvent>) {
    while let Some(event) = receiver.recv().await {
        // Process event (send webhook, update cache, etc)
        println!("Processing event: {:?}", event);
    }
}

// Usage
let (publisher, receiver) = OrderEventPublisher::new();
tokio::spawn(event_worker(receiver));

// Publish events
publisher.publish(order_event).await?;
```

**Recursos:**
- Tokio Channels: https://tokio.rs/tokio/tutorial/channels

**Checkpoint:**
- [ ] Crear mpsc channel
- [ ] Enviar events via channel
- [ ] Background worker con tokio::spawn
- [ ] Procesar events asíncronamente

---

## WEEK 7: Webhook Management (Retry + DLQ)

**Total estudio:** 6 horas

### DAY 1 (Monday): Exponential Backoff (2h)

**Backoff Algorithm (1h)**
```rust
use std::time::Duration;

pub fn calculate_backoff(retry_count: i32, base_delay_ms: u64, max_delay_ms: u64) -> Duration {
    let base_delay = Duration::from_millis(base_delay_ms);
    let max_delay = Duration::from_millis(max_delay_ms);
    
    // 2^retry_count * base_delay
    let delay = base_delay * 2_u32.pow(retry_count as u32);
    
    // Cap at max_delay
    delay.min(max_delay)
}

// Example backoff progression:
// Retry 0: 2^0 * 100ms = 100ms
// Retry 1: 2^1 * 100ms = 200ms
// Retry 2: 2^2 * 100ms = 400ms
// Retry 3: 2^3 * 100ms = 800ms
// Retry 4: 2^4 * 100ms = 1600ms
// Retry 5: 2^5 * 100ms = 3200ms (capped at max)
```

**Retry Logic (1h)**
```rust
pub async fn retry_webhook_with_backoff(
    pool: &PgPool,
    webhook_id: Uuid,
) -> Result<()> {
    let webhook = get_webhook(pool, webhook_id).await?;
    
    // Check if max retries exceeded
    if webhook.retry_count >= webhook.max_retries {
        move_to_dlq(pool, webhook_id, "Max retries exceeded").await?;
        return Ok(());
    }
    
    // Calculate backoff
    let backoff = calculate_backoff(webhook.retry_count, 1000, 60000);
    let next_retry = Utc::now() + chrono::Duration::from_std(backoff)?;
    
    // Schedule next retry
    sqlx::query(
        "UPDATE webhook_events 
         SET retry_count = retry_count + 1,
             next_retry_at = $1,
             status = 'pending',
             updated_at = NOW()
         WHERE id = $2"
    )
    .bind(next_retry)
    .bind(webhook_id)
    .execute(pool)
    .await?;
    
    Ok(())
}
```

**Recursos:**
- Exponential Backoff: https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/

**Checkpoint:**
- [ ] Implementar calculate_backoff()
- [ ] Entender progresión exponencial
- [ ] Cap máximo de delay
- [ ] Schedule retry con next_retry_at

---

### DAY 2 (Tuesday): Retry Patterns (2h)

**Retry with Closure (1h)**
```rust
use std::future::Future;
use std::pin::Pin;

pub async fn retry_with_policy<F, T>(
    mut operation: F,
    max_attempts: u32,
    base_delay_ms: u64,
) -> Result<T>
where
    F: FnMut() -> Pin<Box<dyn Future<Output = Result<T>>>>,
{
    let mut attempt = 0;
    
    loop {
        attempt += 1;
        
        match operation().await {
            Ok(result) => return Ok(result),
            Err(e) if attempt >= max_attempts => {
                return Err(Error::MaxRetriesExceeded {
                    attempts: attempt,
                    last_error: e.to_string(),
                });
            }
            Err(_) => {
                let backoff = calculate_backoff(attempt as i32, base_delay_ms, 60000);
                tokio::time::sleep(backoff).await;
            }
        }
    }
}

// Usage
let result = retry_with_policy(
    || Box::pin(process_webhook(webhook_id)),
    3,
    1000,
).await?;
```

**Idempotency Keys (1h)**
```rust
pub async fn process_webhook_idempotent(
    pool: &PgPool,
    event_id: &str,
    payload: serde_json::Value,
) -> Result<()> {
    // Check if already processed
    let existing = sqlx::query!(
        "SELECT id FROM webhook_events WHERE event_id = $1",
        event_id
    )
    .fetch_optional(pool)
    .await?;
    
    if existing.is_some() {
        return Ok(()); // Already processed, skip
    }
    
    // Process webhook
    let mut tx = pool.begin().await?;
    
    sqlx::query(
        "INSERT INTO webhook_events (id, event_id, payload, status, created_at)
         VALUES ($1, $2, $3, 'processing', NOW())"
    )
    .bind(Uuid::new_v4())
    .bind(event_id)
    .bind(&payload)
    .execute(&mut *tx)
    .await?;
    
    // Process logic here...
    
    tx.commit().await?;
    Ok(())
}
```

**Checkpoint:**
- [ ] Implementar retry_with_policy
- [ ] Usar closure para operation
- [ ] Idempotency check con event_id
- [ ] Evitar procesamiento duplicado

---

### DAY 3 (Wednesday): Dead Letter Queue (1h)

**DLQ Implementation (1h)**
```rust
pub async fn move_to_dlq(
    pool: &PgPool,
    webhook_id: Uuid,
    error_message: &str,
) -> Result<()> {
    let mut tx = pool.begin().await?;
    
    // Update webhook status
    sqlx::query(
        "UPDATE webhook_events 
         SET status = 'dlq', updated_at = NOW()
         WHERE id = $1"
    )
    .bind(webhook_id)
    .execute(&mut *tx)
    .await?;
    
    // Insert into DLQ
    sqlx::query(
        "INSERT INTO webhook_dlq (id, webhook_event_id, error_message, created_at)
         VALUES ($1, $2, $3, NOW())"
    )
    .bind(Uuid::new_v4())
    .bind(webhook_id)
    .bind(error_message)
    .execute(&mut *tx)
    .await?;
    
    tx.commit().await?;
    Ok(())
}

pub async fn get_dlq_items(pool: &PgPool, limit: i64) -> Result<Vec<WebhookDlqItem>> {
    let items = sqlx::query_as::<_, WebhookDlqItem>(
        "SELECT d.*, w.event_id, w.payload
         FROM webhook_dlq d
         JOIN webhook_events w ON w.id = d.webhook_event_id
         ORDER BY d.created_at DESC
         LIMIT $1"
    )
    .bind(limit)
    .fetch_all(pool)
    .await?;
    
    Ok(items)
}

pub async fn retry_from_dlq(pool: &PgPool, dlq_id: Uuid) -> Result<()> {
    sqlx::query(
        "UPDATE webhook_events 
         SET status = 'pending', retry_count = 0
         WHERE id = (SELECT webhook_event_id FROM webhook_dlq WHERE id = $1)"
    )
    .bind(dlq_id)
    .execute(pool)
    .await?;
    
    Ok(())
}
```

**Checkpoint:**
- [ ] Mover webhook a DLQ cuando max retries
- [ ] Guardar error message
- [ ] Query DLQ items
- [ ] Retry manual desde DLQ

---

### DAY 4 (Thursday): Concurrent Task Management (1h)

**Tokio Spawn for Background Jobs (1h)**
```rust
use tokio::task::JoinHandle;

pub struct WebhookProcessor {
    pool: PgPool,
}

impl WebhookProcessor {
    pub fn spawn_worker(&self) -> JoinHandle<()> {
        let pool = self.pool.clone();
        
        tokio::spawn(async move {
            loop {
                match Self::process_pending_webhooks(&pool).await {
                    Ok(count) => {
                        if count == 0 {
                            tokio::time::sleep(Duration::from_secs(5)).await;
                        }
                    }
                    Err(e) => {
                        eprintln!("Worker error: {}", e);
                        tokio::time::sleep(Duration::from_secs(10)).await;
                    }
                }
            }
        })
    }
    
    async fn process_pending_webhooks(pool: &PgPool) -> Result<usize> {
        let webhooks = sqlx::query_as::<_, WebhookEvent>(
            "SELECT * FROM webhook_events
             WHERE status = 'pending' 
             AND (next_retry_at IS NULL OR next_retry_at <= NOW())
             LIMIT 10"
        )
        .fetch_all(pool)
        .await?;
        
        let count = webhooks.len();
        
        for webhook in webhooks {
            match process_webhook(pool, &webhook).await {
                Ok(_) => {
                    mark_webhook_completed(pool, webhook.id).await?;
                }
                Err(_) => {
                    retry_webhook_with_backoff(pool, webhook.id).await?;
                }
            }
        }
        
        Ok(count)
    }
}
```

**Checkpoint:**
- [ ] Spawn background worker
- [ ] Poll pending webhooks
- [ ] Process concurrently (limit 10)
- [ ] Sleep cuando no hay work

---

### DAY 5 (Friday): Testing Retry Logic (0h - revisión)

**Test Scenarios (revisión de tests escritos)**
```rust
#[tokio::test]
async fn test_exponential_backoff() {
    assert_eq!(calculate_backoff(0, 100, 60000), Duration::from_millis(100));
    assert_eq!(calculate_backoff(1, 100, 60000), Duration::from_millis(200));
    assert_eq!(calculate_backoff(2, 100, 60000), Duration::from_millis(400));
    assert_eq!(calculate_backoff(10, 100, 60000), Duration::from_millis(60000)); // capped
}

#[tokio::test]
async fn test_retry_then_dlq() {
    let pool = setup_test_db().await;
    
    let webhook_id = create_webhook(&pool, "event_123").await.unwrap();
    
    // Retry 3 times
    for _ in 0..3 {
        retry_webhook_with_backoff(&pool, webhook_id).await.unwrap();
    }
    
    let webhook = get_webhook(&pool, webhook_id).await.unwrap();
    assert_eq!(webhook.retry_count, 3);
    
    // Next retry should move to DLQ
    retry_webhook_with_backoff(&pool, webhook_id).await.unwrap();
    
    let webhook = get_webhook(&pool, webhook_id).await.unwrap();
    assert_eq!(webhook.status, WebhookStatus::DeadLetter);
    
    // Verify DLQ entry
    let dlq_items = get_dlq_items(&pool, 10).await.unwrap();
    assert_eq!(dlq_items.len(), 1);
}
```

**Checkpoint:**
- [ ] Test backoff calculation
- [ ] Test retry progression
- [ ] Test DLQ after max retries
- [ ] Test idempotency

---

## WEEK 8: Security Hardening

**Total estudio:** 6 horas

### DAY 1 (Monday): Rate Limiting with Governor (2h)

**Governor Setup (1h)**
```rust
use governor::{Quota, RateLimiter, clock::DefaultClock, state::keyed::DefaultKeyedStateStore};
use std::num::NonZeroU32;

pub struct RateLimitConfig {
    pub requests_per_minute: u32,
}

pub type KeyedRateLimiter = RateLimiter<String, DefaultKeyedStateStore<String>, DefaultClock>;

pub fn create_rate_limiter(requests_per_minute: u32) -> KeyedRateLimiter {
    let quota = Quota::per_minute(NonZeroU32::new(requests_per_minute).unwrap());
    RateLimiter::keyed(quota)
}
```

**Rate Limit Middleware (1h)**
```rust
use axum::{
    extract::State,
    http::{Request, StatusCode},
    middleware::Next,
    response::Response,
};

pub async fn rate_limit_middleware<B>(
    State(limiter): State<Arc<KeyedRateLimiter>>,
    req: Request<B>,
    next: Next<B>,
) -> Result<Response, (StatusCode, String)> {
    // Extract user identifier (IP or user_id)
    let key = extract_rate_limit_key(&req);
    
    // Check rate limit
    match limiter.check_key(&key) {
        Ok(_) => Ok(next.run(req).await),
        Err(_) => Err((
            StatusCode::TOO_MANY_REQUESTS,
            "Rate limit exceeded".to_string(),
        )),
    }
}

fn extract_rate_limit_key<B>(req: &Request<B>) -> String {
    // Try to get user_id from auth header
    if let Some(user_id) = extract_user_id_from_request(req) {
        return format!("user:{}", user_id);
    }
    
    // Fallback to IP address
    if let Some(ip) = req.headers()
        .get("x-forwarded-for")
        .and_then(|h| h.to_str().ok())
    {
        return format!("ip:{}", ip);
    }
    
    "unknown".to_string()
}
```

**Recursos:**
- Governor: https://docs.rs/governor

**Checkpoint:**
- [ ] Crear RateLimiter con governor
- [ ] Implementar middleware
- [ ] Extract key (user_id o IP)
- [ ] Return 429 cuando excede límite

---

### DAY 2 (Tuesday): Input Validation (1.5h)

**Validator Crate (1.5h)**
```rust
use validator::{Validate, ValidationError};

#[derive(Debug, Deserialize, Validate)]
pub struct CreateUserRequest {
    #[validate(email(message = "Invalid email format"))]
    pub email: String,
    
    #[validate(length(min = 3, max = 50, message = "Name must be 3-50 characters"))]
    pub name: String,
    
    #[validate(length(min = 8, message = "Password must be at least 8 characters"))]
    #[validate(custom = "validate_password_strength")]
    pub password: String,
}

fn validate_password_strength(password: &str) -> Result<(), ValidationError> {
    let has_uppercase = password.chars().any(|c| c.is_uppercase());
    let has_lowercase = password.chars().any(|c| c.is_lowercase());
    let has_digit = password.chars().any(|c| c.is_numeric());
    
    if has_uppercase && has_lowercase && has_digit {
        Ok(())
    } else {
        Err(ValidationError::new("weak_password"))
    }
}

// In handler
async fn create_user(
    Json(req): Json<CreateUserRequest>,
) -> Result<Json<User>, ValidationError> {
    req.validate()?; // Validates all fields
    
    // Proceed with creation
    Ok(Json(user))
}
```

**Custom Validation Rules**
```rust
#[derive(Debug, Deserialize, Validate)]
pub struct CreateProductRequest {
    #[validate(length(min = 1, max = 200))]
    pub name: String,
    
    #[validate(range(min = 0.01))]
    pub price: f64,
    
    #[validate(range(min = 0))]
    pub stock: i32,
}

#[derive(Debug, Deserialize, Validate)]
pub struct CreateOrderRequest {
    #[validate(length(min = 1, message = "Order must have at least one item"))]
    pub items: Vec<OrderItemRequest>,
}
```

**Recursos:**
- Validator: https://docs.rs/validator

**Checkpoint:**
- [ ] Derive Validate para structs
- [ ] Usar validate attributes
- [ ] Custom validation functions
- [ ] Handle ValidationError

---

### DAY 3 (Wednesday): CORS + Security Headers (1.5h)

**CORS Configuration (1h)**
```rust
use tower_http::cors::{CorsLayer, Any};
use axum::http::{Method, HeaderValue};

pub fn cors_layer() -> CorsLayer {
    CorsLayer::new()
        .allow_origin([
            "http://localhost:3000".parse::<HeaderValue>().unwrap(),
            "https://myapp.com".parse::<HeaderValue>().unwrap(),
        ])
        .allow_methods([
            Method::GET,
            Method::POST,
            Method::PUT,
            Method::PATCH,
            Method::DELETE,
        ])
        .allow_headers([
            axum::http::header::AUTHORIZATION,
            axum::http::header::CONTENT_TYPE,
        ])
        .allow_credentials(true)
}

// Permissive CORS for development (NOT for production)
pub fn cors_permissive() -> CorsLayer {
    CorsLayer::permissive()
}
```

**Security Headers Middleware (0.5h)**
```rust
use axum::{
    http::{Request, Response, HeaderMap, HeaderValue},
    middleware::Next,
};

pub async fn security_headers_middleware<B>(
    req: Request<B>,
    next: Next<B>,
) -> Response {
    let mut response = next.run(req).await;
    let headers = response.headers_mut();
    
    // Prevent MIME type sniffing
    headers.insert(
        "X-Content-Type-Options",
        HeaderValue::from_static("nosniff"),
    );
    
    // Prevent clickjacking
    headers.insert(
        "X-Frame-Options",
        HeaderValue::from_static("DENY"),
    );
    
    // XSS protection
    headers.insert(
        "X-XSS-Protection",
        HeaderValue::from_static("1; mode=block"),
    );
    
    // Strict transport security
    headers.insert(
        "Strict-Transport-Security",
        HeaderValue::from_static("max-age=31536000; includeSubDomains"),
    );
    
    response
}
```

**Recursos:**
- Tower CORS: https://docs.rs/tower-http/latest/tower_http/cors/

**Checkpoint:**
- [ ] Configurar CORS restrictivo
- [ ] Allow specific origins
- [ ] Security headers middleware
- [ ] HSTS, X-Frame-Options, etc.

---

### DAY 4 (Thursday): Secrets Management (1h)

**Environment Variables (1h)**
```rust
use std::env;

#[derive(Debug, Clone)]
pub struct AppConfig {
    pub database_url: String,
    pub redis_url: String,
    pub stripe_api_key: String,
    pub stripe_webhook_secret: String,
    pub jwt_secret: String,
}

impl AppConfig {
    pub fn from_env() -> Result<Self> {
        Ok(Self {
            database_url: env::var("DATABASE_URL")
                .map_err(|_| Error::MissingEnvVar("DATABASE_URL"))?,
            redis_url: env::var("REDIS_URL")
                .unwrap_or_else(|_| "redis://localhost:6379".to_string()),
            stripe_api_key: env::var("STRIPE_API_KEY")
                .map_err(|_| Error::MissingEnvVar("STRIPE_API_KEY"))?,
            stripe_webhook_secret: env::var("STRIPE_WEBHOOK_SECRET")
                .map_err(|_| Error::MissingEnvVar("STRIPE_WEBHOOK_SECRET"))?,
            jwt_secret: env::var("JWT_SECRET")
                .map_err(|_| Error::MissingEnvVar("JWT_SECRET"))?,
        })
    }
    
    pub fn validate(&self) -> Result<()> {
        if self.jwt_secret.len() < 32 {
            return Err(Error::WeakJwtSecret);
        }
        
        if !self.database_url.starts_with("postgres://") {
            return Err(Error::InvalidDatabaseUrl);
        }
        
        Ok(())
    }
}

// Usage
#[tokio::main]
async fn main() -> Result<()> {
    dotenv::dotenv().ok();
    
    let config = AppConfig::from_env()?;
    config.validate()?;
    
    // Never log secrets
    println!("Config loaded successfully");
    
    Ok(())
}
```

**Checkpoint:**
- [ ] Load secrets from env vars
- [ ] Validate config on startup
- [ ] Never log secrets
- [ ] Fail fast si falta config

---

### DAY 5 (Friday): Security Testing (0h - revisión)

**Security Tests (revisión)**
```rust
#[tokio::test]
async fn test_rate_limiting() {
    let app = create_test_app().await;
    let client = TestClient::new(app);
    
    // Send 100 requests (limit is 60/min)
    for i in 0..100 {
        let response = client.get("/api/products").await;
        
        if i < 60 {
            assert_eq!(response.status(), 200);
        } else {
            assert_eq!(response.status(), 429); // Too Many Requests
        }
    }
}

#[tokio::test]
async fn test_invalid_input_rejected() {
    let app = create_test_app().await;
    
    // Invalid email
    let response = app.post("/auth/register")
        .json(&json!({
            "email": "not-an-email",
            "name": "Test",
            "password": "password123"
        }))
        .await;
    assert_eq!(response.status(), 400);
    
    // Short password
    let response = app.post("/auth/register")
        .json(&json!({
            "email": "test@example.com",
            "name": "Test",
            "password": "short"
        }))
        .await;
    assert_eq!(response.status(), 400);
}
```

**Checkpoint:**
- [ ] Test rate limiting enforcement
- [ ] Test input validation
- [ ] Test CORS headers
- [ ] Test security headers

---

## WEEK 9: Resilience Patterns

**Total estudio:** 8 horas

### DAY 1 (Monday): Circuit Breaker Pattern (2h)

**Circuit Breaker States (1h)**
```rust
use std::sync::Arc;
use tokio::sync::Mutex;
use std::time::{Duration, Instant};

#[derive(Debug, Clone, Copy, PartialEq)]
pub enum CircuitState {
    Closed,   // Normal operation
    Open,     // Failing, reject requests
    HalfOpen, // Testing if recovered
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
    
    pub async fn current_state(&self) -> CircuitState {
        *self.state.lock().await
    }
}
```

**Call with Circuit Breaker (1h)**
```rust
impl CircuitBreaker {
    pub async fn call<F, T, E>(&self, f: F) -> Result<T, CircuitBreakerError<E>>
    where
        F: Future<Output = Result<T, E>>,
    {
        let mut state = self.state.lock().await;
        
        match *state {
            CircuitState::Open => {
                // Check if timeout elapsed
                let last_failure = self.last_failure_time.lock().await;
                if let Some(time) = *last_failure {
                    if time.elapsed() > self.timeout {
                        drop(last_failure);
                        *state = CircuitState::HalfOpen;
                    } else {
                        return Err(CircuitBreakerError::Open);
                    }
                }
            }
            CircuitState::HalfOpen => {
                drop(state);
                
                match f.await {
                    Ok(result) => {
                        // Success → transition to Closed
                        let mut state = self.state.lock().await;
                        *state = CircuitState::Closed;
                        *self.failure_count.lock().await = 0;
                        return Ok(result);
                    }
                    Err(e) => {
                        // Failure → back to Open
                        let mut state = self.state.lock().await;
                        *state = CircuitState::Open;
                        *self.last_failure_time.lock().await = Some(Instant::now());
                        return Err(CircuitBreakerError::Failed(e));
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
                        
                        Err(CircuitBreakerError::Failed(e))
                    }
                }
            }
        }
    }
}

#[derive(Debug)]
pub enum CircuitBreakerError<E> {
    Open,
    Failed(E),
}
```

**Recursos:**
- Circuit Breaker Pattern: https://martinfowler.com/bliki/CircuitBreaker.html

**Checkpoint:**
- [ ] Implementar 3 estados
- [ ] Transición Closed → Open (failures)
- [ ] Transición Open → HalfOpen (timeout)
- [ ] Transición HalfOpen → Closed (success)

---

### DAY 2 (Tuesday): State Machines en Rust (1.5h)

**State Machine with Enums (1.5h)**
```rust
// Typestate pattern
pub struct Closed;
pub struct Open;
pub struct HalfOpen;

pub struct CircuitBreaker<State> {
    failure_count: u32,
    failure_threshold: u32,
    last_failure: Option<Instant>,
    timeout: Duration,
    _state: PhantomData<State>,
}

impl CircuitBreaker<Closed> {
    pub fn new(failure_threshold: u32, timeout: Duration) -> Self {
        Self {
            failure_count: 0,
            failure_threshold,
            last_failure: None,
            timeout,
            _state: PhantomData,
        }
    }
    
    pub async fn call<F, T, E>(&mut self, f: F) -> Result<T, CircuitBreaker<Open>>
    where
        F: Future<Output = Result<T, E>>,
    {
        match f.await {
            Ok(result) => {
                self.failure_count = 0;
                Ok(result)
            }
            Err(_) => {
                self.failure_count += 1;
                if self.failure_count >= self.failure_threshold {
                    Err(self.transition_to_open())
                } else {
                    Err(self) // Stay in Closed
                }
            }
        }
    }
    
    fn transition_to_open(self) -> CircuitBreaker<Open> {
        CircuitBreaker {
            failure_count: self.failure_count,
            failure_threshold: self.failure_threshold,
            last_failure: Some(Instant::now()),
            timeout: self.timeout,
            _state: PhantomData,
        }
    }
}
```

**Checkpoint:**
- [ ] Typestate pattern con PhantomData
- [ ] Compile-time state enforcement
- [ ] State transitions vía types

---

### DAY 3 (Wednesday): Timeout Management (1.5h)

**Timeout Wrapper (1.5h)**
```rust
use tokio::time::{timeout, Duration};

pub async fn with_timeout<F, T>(
    future: F,
    duration: Duration,
) -> Result<T, TimeoutError>
where
    F: Future<Output = Result<T>>,
{
    match timeout(duration, future).await {
        Ok(Ok(result)) => Ok(result),
        Ok(Err(e)) => Err(TimeoutError::OperationFailed(e)),
        Err(_) => Err(TimeoutError::Elapsed),
    }
}

#[derive(Debug)]
pub enum TimeoutError {
    Elapsed,
    OperationFailed(Error),
}

// Usage
let result = with_timeout(
    external_api_call(),
    Duration::from_secs(5),
).await?;
```

**Timeout with Fallback**
```rust
pub async fn with_timeout_and_fallback<F, T>(
    primary: F,
    fallback_value: T,
    duration: Duration,
) -> T
where
    F: Future<Output = Result<T>>,
{
    match timeout(duration, primary).await {
        Ok(Ok(result)) => result,
        _ => fallback_value,
    }
}

// Usage
let products = with_timeout_and_fallback(
    fetch_products_from_api(),
    vec![], // Empty vec as fallback
    Duration::from_secs(2),
).await;
```

**Recursos:**
- Tokio Time: https://docs.rs/tokio/latest/tokio/time/

**Checkpoint:**
- [ ] tokio::time::timeout usage
- [ ] Handle timeout vs operation error
- [ ] Fallback strategies
- [ ] Graceful degradation

---

### DAY 4 (Thursday): Arc + Mutex Patterns (2h)

**Shared State with Arc<Mutex<T>> (1h)**
```rust
use std::sync::Arc;
use tokio::sync::Mutex;

#[derive(Clone)]
pub struct SharedCircuitBreaker {
    inner: Arc<Mutex<CircuitBreakerState>>,
}

struct CircuitBreakerState {
    state: CircuitState,
    failure_count: u32,
    last_failure: Option<Instant>,
}

impl SharedCircuitBreaker {
    pub fn new() -> Self {
        Self {
            inner: Arc::new(Mutex::new(CircuitBreakerState {
                state: CircuitState::Closed,
                failure_count: 0,
                last_failure: None,
            })),
        }
    }
    
    pub async fn call<F, T, E>(&self, f: F) -> Result<T, E>
    where
        F: Future<Output = Result<T, E>>,
    {
        let mut state = self.inner.lock().await;
        
        // Circuit breaker logic...
        drop(state); // Release lock before awaiting
        
        f.await
    }
}
```

**AtomicU32 for Counters (1h)**
```rust
use std::sync::atomic::{AtomicU32, Ordering};

pub struct CircuitBreakerMetrics {
    pub total_calls: Arc<AtomicU32>,
    pub failed_calls: Arc<AtomicU32>,
    pub circuit_opens: Arc<AtomicU32>,
}

impl CircuitBreakerMetrics {
    pub fn new() -> Self {
        Self {
            total_calls: Arc::new(AtomicU32::new(0)),
            failed_calls: Arc::new(AtomicU32::new(0)),
            circuit_opens: Arc::new(AtomicU32::new(0)),
        }
    }
    
    pub fn record_call(&self) {
        self.total_calls.fetch_add(1, Ordering::Relaxed);
    }
    
    pub fn record_failure(&self) {
        self.failed_calls.fetch_add(1, Ordering::Relaxed);
    }
    
    pub fn record_open(&self) {
        self.circuit_opens.fetch_add(1, Ordering::Relaxed);
    }
    
    pub fn get_stats(&self) -> (u32, u32, u32) {
        (
            self.total_calls.load(Ordering::Relaxed),
            self.failed_calls.load(Ordering::Relaxed),
            self.circuit_opens.load(Ordering::Relaxed),
        )
    }
}
```

**Recursos:**
- Arc: https://doc.rust-lang.org/std/sync/struct.Arc.html
- Tokio Mutex: https://docs.rs/tokio/latest/tokio/sync/struct.Mutex.html

**Checkpoint:**
- [ ] Arc para shared ownership
- [ ] Tokio Mutex (no std::sync::Mutex)
- [ ] Drop lock antes de .await
- [ ] AtomicU32 para counters

---

### DAY 5 (Friday): Error Propagation Patterns (1h)

**Result Chaining (1h)**
```rust
pub async fn complex_operation_with_resilience(
    user_id: Uuid,
) -> Result<OrderResponse> {
    // Circuit breaker
    let user = circuit_breaker.call(|| {
        fetch_user(user_id)
    }).await
    .map_err(|e| Error::CircuitOpen)?;
    
    // Timeout
    let cart = with_timeout(
        fetch_cart(user_id),
        Duration::from_secs(2),
    ).await
    .unwrap_or_else(|_| Cart::empty());
    
    // Retry
    let products = retry_with_backoff(
        || fetch_products(&cart.product_ids),
        3,
        1000,
    ).await?;
    
    // Calculate total
    let total = calculate_total(&products)?;
    
    Ok(OrderResponse {
        user,
        cart,
        products,
        total,
    })
}
```

**Checkpoint:**
- [ ] Chain resilience patterns
- [ ] Use ? for error propagation
- [ ] Fallback values con unwrap_or_else
- [ ] Compose circuit breaker + timeout + retry

---

**FIN SPRINT 2**

**Resumen de Tópicos Estudiados:**
- Enum state machines
- Event sourcing patterns
- Builder pattern
- Message passing (tokio channels)
- Exponential backoff algorithms
- Retry logic patterns
- Dead Letter Queue
- Concurrent task management (tokio::spawn)
- Rate limiting (governor)
- Input validation (validator)
- CORS + Security headers
- Secrets management
- Circuit breaker pattern
- Timeout management
- Arc + Mutex for shared state
- Error propagation strategies

**Total horas estudio Sprint 2:** 28 horas (8h + 6h + 6h + 8h)

---

# 🦀 GUÍA DE ESTUDIO RUST - SPRINT 3: OBSERVABILITY & DEPLOYMENT

---

## WEEK 10: OpenTelemetry + Logs

**Total estudio:** 5 horas

### DAY 1 (Monday): Tracing Macros (1h)

**#[instrument] Basics (1h)**
```rust
use tracing::{info, warn, error, instrument};

#[instrument]
pub async fn create_payment(amount: Decimal) -> Result<Payment> {
    info!("Creating payment for amount: {}", amount);
    // Function automatically traced
    Ok(payment)
}

#[instrument(skip(pool))]
pub async fn get_user(pool: &PgPool, user_id: Uuid) -> Result<User> {
    info!("Fetching user: {}", user_id);
    // Skip pool from trace (too verbose)
    sqlx::query_as("SELECT * FROM users WHERE id = $1")
        .bind(user_id)
        .fetch_one(pool)
        .await
}

#[instrument(fields(user_id = %user_id, action = "login"))]
pub async fn login(user_id: Uuid) -> Result<String> {
    // Custom fields in trace
    Ok(generate_token(&user_id))
}
```

**Recursos:**
- Tracing: https://docs.rs/tracing

**Checkpoint:**
- [ ] Usar #[instrument] en handlers
- [ ] Skip verbose parameters con skip()
- [ ] Custom fields con fields()
- [ ] Entender span creation automático

---

### DAY 2 (Tuesday): Structured Logging (1h)

**Log Levels (1h)**
```rust
use tracing::{trace, debug, info, warn, error};

pub async fn process_order(order_id: Uuid) -> Result<()> {
    trace!("Entering process_order"); // Very detailed
    
    debug!(order_id = %order_id, "Processing order"); // Debug info
    
    info!("Order processing started: {}", order_id); // Normal info
    
    if let Err(e) = validate_order(order_id).await {
        warn!("Order validation warning: {}", e); // Warning
        return Err(e);
    }
    
    if payment_failed {
        error!(
            order_id = %order_id,
            reason = "insufficient_funds",
            "Payment failed"
        ); // Error
    }
    
    Ok(())
}
```

**Structured Fields**
```rust
use tracing::info;

pub async fn payment_completed(payment: &Payment) {
    info!(
        payment_id = %payment.id,
        amount = %payment.amount,
        currency = "USD",
        status = "completed",
        "Payment processed successfully"
    );
}

// Query in Loki: {status="completed"} | json | amount > 1000
```

**Recursos:**
- Structured Logging: https://docs.rs/tracing/latest/tracing/#recording-fields

**Checkpoint:**
- [ ] Usar niveles apropiados (trace/debug/info/warn/error)
- [ ] Log con structured fields
- [ ] Format fields: %display, ?debug
- [ ] Queryable logs en Loki

---

### DAY 3 (Wednesday): OpenTelemetry Instrumentation (1.5h)

**OTLP Setup (1h)**
```rust
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};
use tracing_opentelemetry::OpenTelemetryLayer;

pub fn init_tracing() -> Result<()> {
    // Jaeger exporter
    let tracer = opentelemetry_jaeger::new_agent_pipeline()
        .with_service_name("marketflow")
        .with_endpoint("localhost:6831")
        .install_simple()?;
    
    // Subscriber with multiple layers
    tracing_subscriber::registry()
        .with(tracing_subscriber::fmt::layer()) // Console logs
        .with(OpenTelemetryLayer::new(tracer))  // OTLP traces
        .init();
    
    Ok(())
}
```

**Context Propagation (0.5h)**
```rust
use tracing::Span;

pub async fn distributed_trace_example() {
    let span = tracing::info_span!("parent_operation");
    let _enter = span.enter();
    
    // This call will be a child span
    child_operation().await;
}

#[instrument]
async fn child_operation() {
    // Automatically linked to parent span
    info!("Child operation executing");
}
```

**Recursos:**
- OpenTelemetry: https://docs.rs/tracing-opentelemetry

**Checkpoint:**
- [ ] Setup Jaeger exporter
- [ ] Init tracing subscriber
- [ ] Multiple layers (console + OTLP)
- [ ] Verify traces en Jaeger UI

---

### DAY 4 (Thursday): Tracing Context (1h)

**Span Events (1h)**
```rust
use tracing::{info_span, Span};

pub async fn complex_operation() -> Result<()> {
    let span = info_span!("complex_operation");
    let _enter = span.enter();
    
    span.record("step", &"initialization");
    initialize().await?;
    
    span.record("step", &"processing");
    process().await?;
    
    span.record("step", &"finalization");
    finalize().await?;
    
    info!("Operation completed");
    Ok(())
}
```

**Async Span Management**
```rust
use tracing::Instrument;

pub async fn async_with_span() -> Result<()> {
    let span = info_span!("async_operation");
    
    // Attach span to future
    async_work()
        .instrument(span)
        .await
}
```

**Recursos:**
- Async Instrumentation: https://docs.rs/tracing/latest/tracing/trait.Instrument.html

**Checkpoint:**
- [ ] Create custom spans
- [ ] Record span events
- [ ] Use .instrument() for async
- [ ] Span hierarchy visible en Jaeger

---

### DAY 5 (Friday): Metrics Basics (0.5h)

**Prometheus Metrics (0.5h)**
```rust
use prometheus::{Counter, Histogram, Registry};
use once_cell::sync::Lazy;

static PAYMENTS_TOTAL: Lazy<Counter> = Lazy::new(|| {
    Counter::new("payments_total", "Total payments processed").unwrap()
});

static PAYMENT_DURATION: Lazy<Histogram> = Lazy::new(|| {
    Histogram::with_opts(
        prometheus::HistogramOpts::new("payment_duration_seconds", "Payment duration")
            .buckets(vec![0.01, 0.05, 0.1, 0.5, 1.0, 5.0])
    ).unwrap()
});

pub async fn process_payment() -> Result<()> {
    let timer = PAYMENT_DURATION.start_timer();
    
    // Process payment...
    
    PAYMENTS_TOTAL.inc();
    timer.observe_duration();
    
    Ok(())
}
```

**Recursos:**
- Prometheus: https://docs.rs/prometheus

**Checkpoint:**
- [ ] Definir Counter metrics
- [ ] Definir Histogram metrics
- [ ] Record metrics en handlers
- [ ] Metrics scrapeables en /metrics

---

## WEEK 11: Load Testing + Optimization

**Total estudio:** 4 horas

### DAY 1 (Monday): Load Testing Setup (1h)

**wrk Scripts (1h)**
```lua
-- tests/load/wrk_scripts/products.lua
wrk.method = "GET"
wrk.headers["Content-Type"] = "application/json"

request = function()
    local path = "/products?query=laptop&limit=10"
    return wrk.format("GET", path)
end

response = function(status, headers, body)
    if status ~= 200 then
        print("Error: " .. status .. " - " .. body)
    end
end
```

```lua
-- tests/load/wrk_scripts/payments.lua
wrk.method = "POST"
wrk.headers["Content-Type"] = "application/json"
wrk.headers["Authorization"] = "Bearer test_token"

request = function()
    local body = '{"amount":"100.00","currency":"USD"}'
    return wrk.format("POST", "/payments", nil, body)
end
```

**Run Scripts**
```bash
# Basic test
wrk -t4 -c100 -d60s http://localhost:3000/products

# With script
wrk -t4 -c100 -d60s -s tests/load/wrk_scripts/products.lua http://localhost:3000

# Options:
# -t4: 4 threads
# -c100: 100 connections
# -d60s: 60 seconds duration
```

**Recursos:**
- wrk: https://github.com/wg/wrk

**Checkpoint:**
- [ ] Install wrk
- [ ] Write basic lua script
- [ ] Run load test
- [ ] Interpret results (req/sec, latency)

---

### DAY 2 (Tuesday): Benchmarking con Criterion (1.5h)

**Criterion Setup (1h)**
```rust
// benches/payment_bench.rs
use criterion::{black_box, criterion_group, criterion_main, Criterion};
use marketflow::payment::calculate_fee;

fn fee_calculation_benchmark(c: &mut Criterion) {
    c.bench_function("calculate_fee", |b| {
        b.iter(|| {
            calculate_fee(black_box(dec!(100.00)))
        })
    });
}

criterion_group!(benches, fee_calculation_benchmark);
criterion_main!(benches);
```

**Async Benchmarks (0.5h)**
```rust
use criterion::{criterion_group, criterion_main, Criterion};
use tokio::runtime::Runtime;

fn async_benchmark(c: &mut Criterion) {
    let rt = Runtime::new().unwrap();
    
    c.bench_function("async_operation", |b| {
        b.to_async(&rt).iter(|| async {
            expensive_async_operation().await
        })
    });
}

criterion_group!(benches, async_benchmark);
criterion_main!(benches);
```

**Recursos:**
- Criterion: https://docs.rs/criterion

**Checkpoint:**
- [ ] Setup criterion in Cargo.toml
- [ ] Write sync benchmark
- [ ] Write async benchmark
- [ ] Run: cargo bench

---

### DAY 3 (Wednesday): Query Optimization (1h)

**EXPLAIN ANALYZE (1h)**
```sql
-- Check query plan
EXPLAIN ANALYZE
SELECT * FROM products
WHERE to_tsvector('english', name || ' ' || description) @@ 
      plainto_tsquery('english', 'laptop');

-- Add index if needed
CREATE INDEX idx_products_fts 
ON products 
USING GIN (to_tsvector('english', name || ' ' || description));

-- Check index usage
EXPLAIN ANALYZE
SELECT * FROM products
WHERE to_tsvector('english', name || ' ' || description) @@ 
      plainto_tsquery('english', 'laptop');
```

**N+1 Query Prevention**
```rust
// BAD: N+1 queries
pub async fn get_orders_with_items_slow(pool: &PgPool) -> Result<Vec<Order>> {
    let orders = sqlx::query_as::<_, Order>("SELECT * FROM orders")
        .fetch_all(pool)
        .await?;
    
    for order in &mut orders {
        // N queries!
        let items = sqlx::query_as::<_, OrderItem>(
            "SELECT * FROM order_items WHERE order_id = $1"
        )
        .bind(order.id)
        .fetch_all(pool)
        .await?;
        
        order.items = items;
    }
    
    Ok(orders)
}

// GOOD: Single JOIN query
pub async fn get_orders_with_items_fast(pool: &PgPool) -> Result<Vec<Order>> {
    let rows = sqlx::query!(
        "SELECT 
            o.id as order_id, o.total_amount, o.status,
            i.id as item_id, i.product_id, i.quantity, i.price
         FROM orders o
         LEFT JOIN order_items i ON i.order_id = o.id"
    )
    .fetch_all(pool)
    .await?;
    
    // Group items by order
    let mut orders = HashMap::new();
    for row in rows {
        orders.entry(row.order_id)
            .or_insert_with(|| Order::new(row.order_id))
            .items.push(OrderItem {
                id: row.item_id,
                product_id: row.product_id,
                quantity: row.quantity,
                price: row.price,
            });
    }
    
    Ok(orders.into_values().collect())
}
```

**Checkpoint:**
- [ ] Use EXPLAIN ANALYZE
- [ ] Identify missing indices
- [ ] Avoid N+1 queries
- [ ] Use JOINs when needed

---

### DAY 4 (Thursday): Performance Profiling (0.5h)

**Flamegraph (0.5h)**
```bash
# Install
cargo install flamegraph

# Profile application
cargo flamegraph --bin marketflow

# Open flamegraph.svg in browser
```

**Profile Specific Test**
```bash
# Profile specific function
cargo flamegraph --bin marketflow --test integration_test
```

**Recursos:**
- Flamegraph: https://github.com/flamegraph-rs/flamegraph

**Checkpoint:**
- [ ] Generate flamegraph
- [ ] Identify hot paths
- [ ] Focus optimization efforts
- [ ] Re-profile after changes

---

## WEEK 12: Documentation + Final

**Total estudio:** 2 horas

### DAY 1 (Monday): Doc Comments (1h)

**Rustdoc Basics (1h)**
```rust
/// Creates a new payment with the given amount.
///
/// # Arguments
///
/// * `pool` - Database connection pool
/// * `amount` - Payment amount in USD
///
/// # Returns
///
/// Returns the created `Payment` or an error if the amount is invalid.
///
/// # Errors
///
/// This function will return an error if:
/// * The amount is negative or zero
/// * The database connection fails
///
/// # Examples
///
/// ```
/// use marketflow::payment::create_payment;
/// use rust_decimal_macros::dec;
///
/// # async fn example() -> Result<(), Box<dyn std::error::Error>> {
/// let pool = create_pool().await?;
/// let payment = create_payment(&pool, dec!(100.00)).await?;
/// assert_eq!(payment.amount, dec!(100.00));
/// # Ok(())
/// # }
/// ```
#[instrument(skip(pool))]
pub async fn create_payment(pool: &PgPool, amount: Decimal) -> Result<Payment> {
    if amount <= Decimal::ZERO {
        return Err(Error::InvalidAmount);
    }
    
    // Implementation...
}
```

**Module Documentation**
```rust
//! # Payment Module
//!
//! This module provides payment processing functionality using Stripe.
//!
//! ## Features
//!
//! - Payment creation
//! - Webhook handling
//! - Idempotency enforcement
//!
//! ## Example
//!
//! ```no_run
//! use marketflow::payment::PaymentService;
//!
//! # async fn example() {
//! let service = PaymentService::new(pool, stripe_client);
//! let payment = service.create(amount).await.unwrap();
//! # }
//! ```

pub mod models;
pub mod service;
pub mod webhook;
```

**Recursos:**
- Rustdoc: https://doc.rust-lang.org/rustdoc/

**Checkpoint:**
- [ ] Doc comments con ///
- [ ] Sections: Arguments, Returns, Errors, Examples
- [ ] Module docs con //!
- [ ] cargo doc --open

---

### DAY 2 (Tuesday): OpenAPI with utoipa (1h)

**utoipa Setup (1h)**
```rust
use utoipa::{OpenApi, ToSchema};

#[derive(OpenApi)]
#[openapi(
    paths(
        create_payment,
        get_payment,
        register_user,
        login,
    ),
    components(
        schemas(
            Payment,
            User,
            CreatePaymentRequest,
            LoginRequest,
        )
    ),
    tags(
        (name = "payments", description = "Payment operations"),
        (name = "auth", description = "Authentication endpoints"),
    ),
    info(
        title = "MarketFlow API",
        version = "1.0.0",
        description = "E-commerce marketplace API",
    )
)]
struct ApiDoc;

#[derive(Serialize, Deserialize, ToSchema)]
pub struct Payment {
    /// Payment ID
    pub id: Uuid,
    /// Payment amount in USD
    pub amount: Decimal,
    /// Payment status
    pub status: PaymentStatus,
}

/// Create a new payment
#[utoipa::path(
    post,
    path = "/payments",
    request_body = CreatePaymentRequest,
    responses(
        (status = 201, description = "Payment created", body = Payment),
        (status = 400, description = "Invalid request"),
        (status = 401, description = "Unauthorized"),
    ),
    tag = "payments"
)]
pub async fn create_payment(
    State(state): State<AppState>,
    Json(req): Json<CreatePaymentRequest>,
) -> Result<Json<Payment>> {
    // Implementation...
}
```

**Swagger UI Integration**
```rust
use utoipa_swagger_ui::SwaggerUi;

let app = Router::new()
    .merge(SwaggerUi::new("/swagger-ui")
        .url("/api-docs/openapi.json", ApiDoc::openapi()))
    .route("/payments", post(create_payment));
```

**Recursos:**
- utoipa: https://docs.rs/utoipa

**Checkpoint:**
- [ ] Derive ToSchema for types
- [ ] Annotate handlers con #[utoipa::path]
- [ ] Generate OpenAPI spec
- [ ] Swagger UI en /swagger-ui

---

**FIN SPRINT 3**

**Resumen de Tópicos Estudiados:**
- Tracing macros (#[instrument])
- Structured logging
- OpenTelemetry instrumentation
- Context propagation
- Prometheus metrics
- Load testing con wrk
- Benchmarking con Criterion
- Query optimization (EXPLAIN ANALYZE)
- Performance profiling (flamegraph)
- Doc comments (rustdoc)
- OpenAPI generation (utoipa)

**Total horas estudio Sprint 3:** 11 horas (5h + 4h + 2h)

---

# 🦀 GUÍA DE ESTUDIO RUST - SPRINT 4: FINAL POLISH

---

## WEEK 13: Advanced Features

**Total estudio:** 6 horas

### DAY 1 (Monday): Query Builder Pattern (1.5h)

**Dynamic Query Construction (1h)**
```rust
use sqlx::QueryBuilder;

pub struct ProductFilters {
    pub category_id: Option<Uuid>,
    pub min_price: Option<Decimal>,
    pub max_price: Option<Decimal>,
    pub in_stock: Option<bool>,
    pub search_query: Option<String>,
}

pub async fn search_products_dynamic(
    pool: &PgPool,
    filters: ProductFilters,
    limit: i64,
) -> Result<Vec<Product>> {
    let mut builder = QueryBuilder::new("SELECT * FROM products WHERE 1=1");
    
    if let Some(cat) = filters.category_id {
        builder.push(" AND category_id = ");
        builder.push_bind(cat);
    }
    
    if let Some(min) = filters.min_price {
        builder.push(" AND price >= ");
        builder.push_bind(min);
    }
    
    if let Some(max) = filters.max_price {
        builder.push(" AND price <= ");
        builder.push_bind(max);
    }
    
    if filters.in_stock == Some(true) {
        builder.push(" AND stock > 0");
    }
    
    if let Some(query) = filters.search_query {
        builder.push(" AND to_tsvector('english', name || ' ' || description) @@ plainto_tsquery('english', ");
        builder.push_bind(query);
        builder.push(")");
    }
    
    builder.push(" ORDER BY created_at DESC LIMIT ");
    builder.push_bind(limit);
    
    builder.build_query_as::<Product>()
        .fetch_all(pool)
        .await
}
```

**Type-Safe Builder (0.5h)**
```rust
pub struct ProductQueryBuilder<'a> {
    query: QueryBuilder<'a, Postgres>,
    has_filters: bool,
}

impl<'a> ProductQueryBuilder<'a> {
    pub fn new() -> Self {
        Self {
            query: QueryBuilder::new("SELECT * FROM products WHERE 1=1"),
            has_filters: false,
        }
    }
    
    pub fn category(mut self, category_id: Uuid) -> Self {
        self.query.push(" AND category_id = ");
        self.query.push_bind(category_id);
        self.has_filters = true;
        self
    }
    
    pub fn price_range(mut self, min: Decimal, max: Decimal) -> Self {
        self.query.push(" AND price BETWEEN ");
        self.query.push_bind(min);
        self.query.push(" AND ");
        self.query.push_bind(max);
        self.has_filters = true;
        self
    }
    
    pub fn build(mut self) -> Query<'a, Postgres, PgArguments> {
        self.query.build()
    }
}

// Usage
let products = ProductQueryBuilder::new()
    .category(cat_id)
    .price_range(dec!(10), dec!(100))
    .build()
    .fetch_all(pool)
    .await?;
```

**Recursos:**
- SQLx QueryBuilder: https://docs.rs/sqlx/latest/sqlx/query_builder/

**Checkpoint:**
- [ ] Construir queries dinámicamente
- [ ] Evitar SQL injection con push_bind
- [ ] Chainable builder methods
- [ ] Type-safe query construction

---

### DAY 2 (Tuesday): Cursor Pagination (1.5h)

**Cursor-Based Pagination (1h)**
```rust
#[derive(Deserialize)]
pub struct PaginationParams {
    pub limit: Option<i64>,
    pub cursor: Option<Uuid>,
}

#[derive(Serialize)]
pub struct PaginatedResponse<T> {
    pub items: Vec<T>,
    pub next_cursor: Option<Uuid>,
    pub has_more: bool,
}

pub async fn paginate_products(
    pool: &PgPool,
    params: PaginationParams,
) -> Result<PaginatedResponse<Product>> {
    let limit = params.limit.unwrap_or(20).min(100);
    
    let mut query = QueryBuilder::new("SELECT * FROM products WHERE 1=1");
    
    if let Some(cursor) = params.cursor {
        query.push(" AND id > ");
        query.push_bind(cursor);
    }
    
    query.push(" ORDER BY id ASC LIMIT ");
    query.push_bind(limit + 1); // Fetch one extra to check has_more
    
    let mut items = query.build_query_as::<Product>()
        .fetch_all(pool)
        .await?;
    
    let has_more = items.len() > limit as usize;
    if has_more {
        items.pop(); // Remove extra item
    }
    
    let next_cursor = items.last().map(|p| p.id);
    
    Ok(PaginatedResponse {
        items,
        next_cursor,
        has_more,
    })
}
```

**Why Cursor > Offset (0.5h)**
```rust
// BAD: Offset pagination (slow for large offsets)
pub async fn paginate_with_offset(
    pool: &PgPool,
    page: i64,
    limit: i64,
) -> Result<Vec<Product>> {
    let offset = (page - 1) * limit;
    
    // Slow: DB must scan and skip 'offset' rows
    sqlx::query_as::<_, Product>(
        "SELECT * FROM products ORDER BY id LIMIT $1 OFFSET $2"
    )
    .bind(limit)
    .bind(offset) // Problem: O(n) for large offsets
    .fetch_all(pool)
    .await
}

// GOOD: Cursor pagination (O(1) with index)
pub async fn paginate_with_cursor(
    pool: &PgPool,
    cursor: Option<Uuid>,
    limit: i64,
) -> Result<Vec<Product>> {
    let mut query = QueryBuilder::new("SELECT * FROM products WHERE 1=1");
    
    if let Some(c) = cursor {
        query.push(" AND id > "); // Uses index directly
        query.push_bind(c);
    }
    
    query.push(" ORDER BY id LIMIT ");
    query.push_bind(limit);
    
    query.build_query_as::<Product>()
        .fetch_all(pool)
        .await
}

/*
Performance comparison:
- Offset 10000: ~500ms (scans 10000 rows)
- Cursor: ~5ms (direct index lookup)
*/
```

**Recursos:**
- Cursor Pagination: https://use-the-index-luke.com/no-offset

**Checkpoint:**
- [ ] Implementar cursor-based pagination
- [ ] Entender ventaja sobre OFFSET
- [ ] next_cursor en response
- [ ] has_more flag

---

### DAY 3 (Wednesday): Background Jobs (1.5h)

**Job Queue Pattern (1h)**
```rust
use tokio::sync::mpsc;
use std::sync::Arc;

#[derive(Debug, Clone)]
pub enum Job {
    WarmCache { product_ids: Vec<Uuid> },
    SendEmail { user_id: Uuid, template: String },
    ProcessWebhook { webhook_id: Uuid },
}

pub struct JobQueue {
    sender: mpsc::Sender<Job>,
}

impl JobQueue {
    pub fn new(worker_count: usize) -> Self {
        let (sender, receiver) = mpsc::channel(100);
        
        // Spawn workers
        for _ in 0..worker_count {
            let mut rx = receiver.clone();
            tokio::spawn(async move {
                while let Some(job) = rx.recv().await {
                    if let Err(e) = process_job(job).await {
                        eprintln!("Job failed: {}", e);
                    }
                }
            });
        }
        
        Self { sender }
    }
    
    pub async fn enqueue(&self, job: Job) -> Result<()> {
        self.sender.send(job).await
            .map_err(|_| Error::QueueClosed)?;
        Ok(())
    }
}

async fn process_job(job: Job) -> Result<()> {
    match job {
        Job::WarmCache { product_ids } => {
            for id in product_ids {
                warm_product_cache(id).await?;
            }
        }
        Job::SendEmail { user_id, template } => {
            send_email_to_user(user_id, &template).await?;
        }
        Job::ProcessWebhook { webhook_id } => {
            process_webhook(webhook_id).await?;
        }
    }
    Ok(())
}
```

**Scheduled Jobs (0.5h)**
```rust
use tokio::time::{interval, Duration};

pub fn spawn_cache_warmer(redis: Arc<redis::Connection>, pool: Arc<PgPool>) {
    tokio::spawn(async move {
        let mut ticker = interval(Duration::from_secs(3600)); // Every hour
        
        loop {
            ticker.tick().await;
            
            info!("Starting cache warming...");
            match warm_top_products(&redis, &pool).await {
                Ok(count) => info!("Warmed {} products", count),
                Err(e) => error!("Cache warming failed: {}", e),
            }
        }
    });
}

pub async fn warm_top_products(
    redis: &redis::Connection,
    pool: &PgPool,
) -> Result<usize> {
    let products = sqlx::query_as::<_, Product>(
        "SELECT * FROM products 
         ORDER BY stock DESC 
         LIMIT 100"
    )
    .fetch_all(pool)
    .await?;
    
    for product in &products {
        let key = format!("product:{}", product.id);
        let json = serde_json::to_string(product)?;
        redis.set_ex(&key, json, 3600)?;
    }
    
    Ok(products.len())
}
```

**Recursos:**
- Tokio Time: https://docs.rs/tokio/latest/tokio/time/

**Checkpoint:**
- [ ] Job queue con mpsc channel
- [ ] Multiple workers
- [ ] Scheduled jobs con interval
- [ ] Background processing

---

### DAY 4 (Thursday): Advanced Error Handling (1h)

**Context with anyhow (1h)**
```rust
use anyhow::{Context, Result};

pub async fn create_order_with_context(
    pool: &PgPool,
    user_id: Uuid,
    items: Vec<OrderItem>,
) -> Result<Order> {
    let user = sqlx::query_as::<_, User>("SELECT * FROM users WHERE id = $1")
        .bind(user_id)
        .fetch_one(pool)
        .await
        .context("Failed to fetch user")?;
    
    let total = calculate_total(&items)
        .context("Failed to calculate order total")?;
    
    if total > user.balance {
        anyhow::bail!("Insufficient balance: need {}, have {}", total, user.balance);
    }
    
    let order = sqlx::query_as::<_, Order>(
        "INSERT INTO orders (id, user_id, total_amount, status) 
         VALUES ($1, $2, $3, 'pending') RETURNING *"
    )
    .bind(Uuid::new_v4())
    .bind(user_id)
    .bind(total)
    .fetch_one(pool)
    .await
    .with_context(|| format!("Failed to create order for user {}", user_id))?;
    
    Ok(order)
}

/*
Error output with context:
Failed to create order for user 550e8400-e29b-41d4-a716-446655440000

Caused by:
    0: Failed to fetch user
    1: connection closed
*/
```

**Recursos:**
- anyhow: https://docs.rs/anyhow

**Checkpoint:**
- [ ] Add context to errors
- [ ] Use .context() for static messages
- [ ] Use .with_context() for dynamic messages
- [ ] anyhow::bail! for quick errors

---

### DAY 5 (Friday): Performance Patterns (0.5h)

**Lazy Static (0.5h)**
```rust
use once_cell::sync::Lazy;
use regex::Regex;

// Compiled once, reused forever
static EMAIL_REGEX: Lazy<Regex> = Lazy::new(|| {
    Regex::new(r"^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$").unwrap()
});

pub fn validate_email(email: &str) -> bool {
    EMAIL_REGEX.is_match(email)
}

// Connection pool (initialized once)
static DB_POOL: Lazy<PgPool> = Lazy::new(|| {
    PgPool::connect_lazy("postgres://...")
        .expect("Failed to create pool")
});
```

**Recursos:**
- once_cell: https://docs.rs/once_cell

**Checkpoint:**
- [ ] Use Lazy for expensive initialization
- [ ] Compile regex once
- [ ] Lazy connection pools
- [ ] Thread-safe static initialization

---

## WEEK 14: Security Audit + Hardening

**Total estudio:** 4 horas

### DAY 1 (Monday): cargo audit (1h)

**Dependency Auditing (1h)**
```bash
# Install cargo-audit
cargo install cargo-audit

# Audit dependencies
cargo audit

# Example output:
# Crate:     time
# Version:   0.1.43
# Warning:   time 0.1 has known vulnerabilities
# Solution:  Upgrade to time 0.3

# Fix in Cargo.toml
[dependencies]
time = "0.3"

# Re-audit
cargo audit

# Continuous auditing in CI
# .github/workflows/security.yml
- name: Security audit
  run: cargo audit
```

**Deny.toml Configuration**
```toml
# deny.toml
[advisories]
vulnerability = "deny"
unmaintained = "warn"
unsound = "warn"
yanked = "deny"

[licenses]
unlicensed = "deny"
allow = [
    "MIT",
    "Apache-2.0",
]

[bans]
multiple-versions = "warn"
deny = [
    { name = "openssl", reason = "use rustls instead" },
]
```

**Recursos:**
- cargo-audit: https://github.com/rustsec/rustsec/tree/main/cargo-audit
- cargo-deny: https://github.com/EmbarkStudios/cargo-deny

**Checkpoint:**
- [ ] Run cargo audit
- [ ] Fix vulnerable dependencies
- [ ] Configure deny.toml
- [ ] Add to CI pipeline

---

### DAY 2 (Tuesday): SQL Injection Prevention (1h)

**Safe Query Patterns (1h)**
```rust
// ❌ NEVER: String concatenation
pub async fn search_users_unsafe(pool: &PgPool, query: &str) -> Result<Vec<User>> {
    let sql = format!("SELECT * FROM users WHERE name LIKE '%{}%'", query);
    // SQL injection vulnerability!
    sqlx::query_as(&sql).fetch_all(pool).await
}

// ✅ ALWAYS: Parameterized queries
pub async fn search_users_safe(pool: &PgPool, query: &str) -> Result<Vec<User>> {
    sqlx::query_as::<_, User>(
        "SELECT * FROM users WHERE name ILIKE $1"
    )
    .bind(format!("%{}%", query)) // Safe: parameter binding
    .fetch_all(pool)
    .await
}

// ✅ QueryBuilder (also safe)
pub async fn search_users_builder(pool: &PgPool, query: &str) -> Result<Vec<User>> {
    let mut builder = QueryBuilder::new("SELECT * FROM users WHERE name ILIKE ");
    builder.push_bind(format!("%{}%", query)); // Safe: push_bind
    
    builder.build_query_as::<Product>()
        .fetch_all(pool)
        .await
}

// ❌ NEVER: format! with SQL
let bad = format!("DELETE FROM users WHERE id = {}", user_input);

// ✅ ALWAYS: bind parameters
sqlx::query("DELETE FROM users WHERE id = $1")
    .bind(user_input)
    .execute(pool)
    .await?;
```

**Audit Script**
```bash
#!/bin/bash
# scripts/audit_sql.sh

echo "Checking for SQL injection risks..."

# Look for format! with SQL keywords
grep -rn "format!\|concat!\|push_str" src/ | grep -i "select\|insert\|update\|delete" | grep -v "push_bind"

if [ $? -eq 0 ]; then
    echo "⚠️  Found potential SQL injection risks!"
    exit 1
else
    echo "✅ No SQL injection risks found"
fi
```

**Checkpoint:**
- [ ] Never use format! with SQL
- [ ] Always use $1, $2 parameters
- [ ] Use push_bind with QueryBuilder
- [ ] Audit script in CI

---

### DAY 3 (Wednesday): OWASP Security Headers (1h)

**Complete Security Headers (1h)**
```rust
use axum::{
    http::{header, HeaderMap, HeaderValue, StatusCode},
    middleware::Next,
    response::Response,
    Request,
};

pub async fn security_headers<B>(
    req: Request<B>,
    next: Next<B>,
) -> Response {
    let mut response = next.run(req).await;
    let headers = response.headers_mut();
    
    // Prevent MIME sniffing
    headers.insert(
        "X-Content-Type-Options",
        HeaderValue::from_static("nosniff"),
    );
    
    // Prevent clickjacking
    headers.insert(
        "X-Frame-Options",
        HeaderValue::from_static("DENY"),
    );
    
    // XSS protection (legacy but still useful)
    headers.insert(
        "X-XSS-Protection",
        HeaderValue::from_static("1; mode=block"),
    );
    
    // HSTS (force HTTPS)
    headers.insert(
        "Strict-Transport-Security",
        HeaderValue::from_static("max-age=31536000; includeSubDomains; preload"),
    );
    
    // Content Security Policy
    headers.insert(
        "Content-Security-Policy",
        HeaderValue::from_static(
            "default-src 'self'; \
             script-src 'self' 'unsafe-inline'; \
             style-src 'self' 'unsafe-inline'; \
             img-src 'self' data: https:; \
             font-src 'self' data:;"
        ),
    );
    
    // Referrer policy
    headers.insert(
        "Referrer-Policy",
        HeaderValue::from_static("strict-origin-when-cross-origin"),
    );
    
    // Permissions policy
    headers.insert(
        "Permissions-Policy",
        HeaderValue::from_static("geolocation=(), microphone=(), camera=()"),
    );
    
    response
}
```

**Recursos:**
- OWASP Secure Headers: https://owasp.org/www-project-secure-headers/

**Checkpoint:**
- [ ] All OWASP headers configured
- [ ] CSP policy defined
- [ ] HSTS enabled
- [ ] Test headers con securityheaders.com

---

### DAY 4 (Thursday): Secrets Rotation (1h)

**Secrets Management (1h)**
```rust
use std::sync::Arc;
use tokio::sync::RwLock;

pub struct RotatingSecret {
    current: Arc<RwLock<String>>,
}

impl RotatingSecret {
    pub fn new(initial: String) -> Self {
        Self {
            current: Arc::new(RwLock::new(initial)),
        }
    }
    
    pub async fn get(&self) -> String {
        self.current.read().await.clone()
    }
    
    pub async fn rotate(&self, new_secret: String) {
        let mut current = self.current.write().await;
        *current = new_secret;
    }
}

// JWT with rotating secret
pub struct JwtService {
    secret: RotatingSecret,
}

impl JwtService {
    pub async fn generate_token(&self, user_id: &str) -> Result<String> {
        let secret = self.secret.get().await;
        
        let claims = Claims {
            sub: user_id.to_string(),
            exp: (Utc::now() + Duration::hours(24)).timestamp(),
        };
        
        encode(
            &Header::default(),
            &claims,
            &EncodingKey::from_secret(secret.as_bytes()),
        )
        .map_err(Into::into)
    }
}

// Rotation endpoint (admin only)
pub async fn rotate_jwt_secret(
    State(jwt_service): State<Arc<JwtService>>,
    AuthUser { user_id }: AuthUser,
) -> Result<StatusCode> {
    // Verify admin
    if !is_admin(user_id).await? {
        return Err(Error::Unauthorized);
    }
    
    // Generate new secret
    let new_secret = generate_random_string(64);
    
    // Rotate
    jwt_service.secret.rotate(new_secret).await;
    
    info!("JWT secret rotated by admin: {}", user_id);
    
    Ok(StatusCode::OK)
}
```

**Checkpoint:**
- [ ] Secrets in RwLock for rotation
- [ ] Rotation endpoint (admin only)
- [ ] Generate random secrets
- [ ] Log rotation events

---

## WEEK 15: Final Integration + Portfolio

**Total estudio:** 2 horas

### DAY 1 (Monday): GitHub Best Practices (1h)

**Commit Messages (0.5h)**
```bash
# Good commit messages
git commit -m "feat(payment): Add idempotency key validation"
git commit -m "fix(auth): Resolve JWT expiration bug"
git commit -m "refactor(ledger): Extract balance calculation logic"
git commit -m "test(order): Add state transition tests"
git commit -m "docs(api): Update Swagger specs"
git commit -m "perf(product): Optimize full-text search query"

# Conventional Commits format:
# <type>(<scope>): <description>
#
# Types:
# - feat: New feature
# - fix: Bug fix
# - refactor: Code change (no behavior change)
# - test: Add/update tests
# - docs: Documentation
# - perf: Performance improvement
# - chore: Maintenance tasks
```

**Git Workflow**
```bash
# Clean history
git log --oneline --graph

# Squash commits before merge
git rebase -i HEAD~5

# Good branch names
git checkout -b feat/payment-idempotency
git checkout -b fix/auth-jwt-expiration
git checkout -b refactor/ledger-balance

# Tag releases
git tag -a v1.0.0 -m "Release v1.0.0"
git push origin v1.0.0
```

**Recursos:**
- Conventional Commits: https://www.conventionalcommits.org/

**Checkpoint:**
- [ ] Clean commit messages
- [ ] Logical commit grouping
- [ ] Squash before merge
- [ ] Tag releases

---

### DAY 2 (Tuesday): Code Quality Tools (1h)

**clippy Lints (0.5h)**
```bash
# Run clippy
cargo clippy -- -D warnings

# Common lints to fix:
# - Unnecessary clones
# - Unused variables
# - Inefficient patterns
# - Missing documentation

# Clippy configuration
# .cargo/config.toml
[target.'cfg(all())']
rustflags = [
    "-W", "clippy::all",
    "-W", "clippy::pedantic",
    "-W", "clippy::nursery",
]

# Allow specific lints
#![allow(clippy::missing_errors_doc)]
```

**rustfmt Configuration (0.5h)**
```toml
# rustfmt.toml
max_width = 100
tab_spaces = 4
edition = "2021"
use_field_init_shorthand = true
use_try_shorthand = true
imports_granularity = "Crate"

# Format code
cargo fmt

# Check formatting
cargo fmt -- --check
```

**Recursos:**
- clippy: https://github.com/rust-lang/rust-clippy
- rustfmt: https://github.com/rust-lang/rustfmt

**Checkpoint:**
- [ ] cargo clippy passes
- [ ] cargo fmt applied
- [ ] Zero warnings
- [ ] Consistent style

---

**FIN SPRINT 4**

---

## RESUMEN COMPLETO - 15 SEMANAS DE ESTUDIO

### Total Horas de Estudio: 51 horas

**Sprint 0 (Weeks 1-2):** 20 horas
- Week 1: 12h (Async, Traits, Errors, Axum, SQLx)
- Week 2: 8h (Enums, Collections, Iterators, Testing)

**Sprint 1 (Weeks 3-5):** 20 horas
- Week 3: 8h (Generics, Password Hashing, JWT, Extractors)
- Week 4: 6h (Full-text Search, Redis, Caching)
- Week 5: 6h (Serde, TTL, Transactions)

**Sprint 2 (Weeks 6-9):** 28 horas
- Week 6: 8h (State Machines, Event Sourcing, Builder Pattern)
- Week 7: 6h (Exponential Backoff, Retry Logic, DLQ)
- Week 8: 6h (Rate Limiting, Validation, CORS, Secrets)
- Week 9: 8h (Circuit Breaker, Timeouts, Arc+Mutex)

**Sprint 3 (Weeks 10-12):** 11 horas
- Week 10: 5h (Tracing, OpenTelemetry, Metrics)
- Week 11: 4h (Load Testing, Benchmarking, Profiling)
- Week 12: 2h (Rustdoc, OpenAPI)

**Sprint 4 (Weeks 13-15):** 12 horas
- Week 13: 6h (Query Builder, Pagination, Background Jobs)
- Week 14: 4h (cargo audit, SQL Injection, OWASP, Secrets)
- Week 15: 2h (Git Best Practices, Code Quality)

---

### Rust Topics Mastered (Total: 60+ topics)

**Core Language:**
- Ownership, Borrowing, Lifetimes
- Traits, Generics, Trait Bounds
- Enums, Pattern Matching
- Error Handling (Result, ?, Custom Errors)
- Collections (Vec, HashMap)
- Iterators (map, filter, fold)
- String types (String vs &str)

**Async Programming:**
- Tokio runtime
- async/await
- Futures
- Channels (mpsc)
- Task spawning
- Concurrent execution

**Web Development:**
- Axum (routing, handlers, extractors, middleware)
- HTTP methods
- JSON serialization (Serde)
- State management
- Error responses

**Database:**
- SQLx (queries, transactions, connection pooling)
- PostgreSQL (full-text search, indices)
- Query optimization
- N+1 prevention

**Caching & Performance:**
- Redis (cache-aside pattern)
- TTL management
- Cache invalidation
- Query builder pattern
- Cursor pagination
- Benchmarking (Criterion)
- Profiling (Flamegraph)

**Security:**
- Password hashing (Argon2)
- JWT tokens
- Rate limiting (Governor)
- Input validation (validator)
- CORS
- Security headers
- Secrets management
- SQL injection prevention
- cargo audit

**Resilience:**
- Circuit breaker pattern
- Exponential backoff
- Retry logic
- Timeouts
- Dead Letter Queue
- Graceful degradation

**Observability:**
- Tracing (tracing crate)
- OpenTelemetry
- Structured logging
- Prometheus metrics
- Span instrumentation

**Testing:**
- Unit tests
- Integration tests
- Async testing
- Test databases

**Documentation:**
- Rustdoc comments
- OpenAPI (utoipa)
- Module documentation

**Code Quality:**
- clippy
- rustfmt
- Conventional commits
- Git workflow

---

### Key Resources Referenced

**Official Docs:**
- Rust Book: https://doc.rust-lang.org/book/
- Tokio: https://tokio.rs
- Axum: https://docs.rs/axum
- SQLx: https://github.com/launchbadge/sqlx

**Crates:**
- async-trait, serde, uuid, chrono
- rust_decimal, argon2, jsonwebtoken
- redis, governor, validator
- tracing, tracing-opentelemetry
- prometheus, criterion, anyhow

**Patterns:**
- Repository pattern
- Builder pattern
- State machines
- Circuit breaker
- Event sourcing
- Cache-aside

---

**Portfolio-Ready Skills:**
✅ Production Rust code (7-9K LOC)
✅ Async/concurrent programming
✅ Database design & optimization
✅ Security best practices
✅ Observability & monitoring
✅ Performance optimization
✅ Clean architecture
✅ Comprehensive testing
✅ Professional documentation

**Resultado:** Rust Developer con skills production-ready para $180K+ salary 🎯
