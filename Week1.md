# MARKETFLOW - ROADMAP OPTIMIZADO (9-10 SEMANAS)

---

# SPRINT 0: FOUNDATION

---

## WEEK 1: Docker + Stripe + Payments

### 1. OBJETIVO
Docker funcionando + Stripe integrado + Sistema de pagos con idempotency + Observabilidad básica

### 2. DELIVERABLES
1. Docker Compose con PostgreSQL, Redis, Jaeger, Prometheus
2. Axum server corriendo (puerto 3000)
3. Stripe webhook handler con signature verification
4. Payment table + repository pattern
5. Idempotency implementada (evitar cobros duplicados)
6. OpenTelemetry traces básicas en Jaeger
7. Health check endpoint
8. Tests: 8 (ALTA prioridad - payment critical path)

### 3. ESTRUCTURA DE ARCHIVOS
```
marketflow/
├── docker-compose.yml
├── .env.example
├── Cargo.toml
├── src/
│   ├── main.rs                    # Axum server + routes
│   ├── config.rs                  # Environment configuration
│   ├── error.rs                   # Error types + Result alias
│   ├── payment/
│   │   ├── mod.rs                 # Module exports
│   │   ├── models.rs              # Payment, PaymentStatus enums
│   │   ├── repository.rs          # PaymentRepository trait + impl
│   │   ├── service.rs             # PaymentService (business logic)
│   │   └── handlers.rs            # REST endpoints
│   ├── stripe/
│   │   ├── mod.rs
│   │   ├── webhook.rs             # Webhook handler + verification
│   │   └── client.rs              # Stripe API client
│   └── observability/
│       ├── mod.rs
│       └── tracing.rs             # OpenTelemetry setup
├── migrations/
│   └── 001_create_payments.sql
└── tests/
    ├── common/
    │   └── mod.rs                 # Test fixtures
    └── payment_test.rs            # Integration tests
```

### 4. DATABASE SCHEMA
```sql
-- migrations/001_create_payments.sql
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

CREATE TABLE payments (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    stripe_payment_intent_id VARCHAR(255) UNIQUE NOT NULL,
    idempotency_key VARCHAR(255) UNIQUE,
    amount DECIMAL(19, 4) NOT NULL CHECK (amount > 0),
    currency VARCHAR(3) NOT NULL DEFAULT 'USD',
    status VARCHAR(50) NOT NULL CHECK (status IN ('pending', 'succeeded', 'failed', 'cancelled')),
    metadata JSONB,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_payments_idempotency ON payments(idempotency_key) WHERE idempotency_key IS NOT NULL;
CREATE INDEX idx_payments_stripe_id ON payments(stripe_payment_intent_id);
CREATE INDEX idx_payments_status ON payments(status);
CREATE INDEX idx_payments_created_at ON payments(created_at DESC);
```

### 5. API ENDPOINTS
```rust
// POST /payments
// Request
{
  "amount": 1000.00,
  "currency": "USD",
  "idempotency_key": "order_123_attempt_1"
}

// Response 201 Created
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "stripe_payment_intent_id": "pi_3AbCdEfGhIjK",
  "amount": 1000.00,
  "status": "pending",
  "created_at": "2024-01-15T10:30:00Z"
}

// POST /webhooks/stripe
// Stripe signature in header: Stripe-Signature
// Body: Stripe event payload
// Response: 200 OK

// GET /payments/:id
// Response 200 OK
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "amount": 1000.00,
  "status": "succeeded"
}

// GET /health
// Response 200 OK
{
  "status": "healthy",
  "database": "connected",
  "redis": "connected"
}
```

### 6. CORE LOGIC
```rust
// src/payment/models.rs
use rust_decimal::Decimal;
use serde::{Deserialize, Serialize};
use sqlx::FromRow;
use uuid::Uuid;

#[derive(Debug, Clone, Copy, Serialize, Deserialize, sqlx::Type, PartialEq)]
#[sqlx(type_name = "varchar")]
pub enum PaymentStatus {
    Pending,
    Succeeded,
    Failed,
    Cancelled,
}

#[derive(Debug, Clone, Serialize, Deserialize, FromRow)]
pub struct Payment {
    pub id: Uuid,
    pub stripe_payment_intent_id: String,
    pub idempotency_key: Option<String>,
    pub amount: Decimal,
    pub currency: String,
    pub status: PaymentStatus,
    pub metadata: Option<serde_json::Value>,
    pub created_at: chrono::DateTime<chrono::Utc>,
    pub updated_at: chrono::DateTime<chrono::Utc>,
}

#[derive(Debug, Deserialize, Validate)]
pub struct CreatePaymentRequest {
    #[validate(range(min = 0.01))]
    pub amount: Decimal,
    #[validate(length(equal = 3))]
    pub currency: String,
    pub idempotency_key: Option<String>,
}

// src/payment/repository.rs
use async_trait::async_trait;
use sqlx::PgPool;
use uuid::Uuid;

#[async_trait]
pub trait PaymentRepository: Send + Sync {
    async fn create(&self, payment: &Payment) -> Result<Payment>;
    async fn find_by_id(&self, id: Uuid) -> Result<Option<Payment>>;
    async fn find_by_idempotency_key(&self, key: &str) -> Result<Option<Payment>>;
    async fn update_status(&self, id: Uuid, status: PaymentStatus) -> Result<()>;
}

pub struct PostgresPaymentRepository {
    pool: PgPool,
}

#[async_trait]
impl PaymentRepository for PostgresPaymentRepository {
    async fn create(&self, payment: &Payment) -> Result<Payment> {
        let result = sqlx::query_as::<_, Payment>(
            r#"
            INSERT INTO payments (id, stripe_payment_intent_id, idempotency_key, amount, currency, status, metadata)
            VALUES ($1, $2, $3, $4, $5, $6, $7)
            RETURNING *
            "#
        )
        .bind(payment.id)
        .bind(&payment.stripe_payment_intent_id)
        .bind(&payment.idempotency_key)
        .bind(payment.amount)
        .bind(&payment.currency)
        .bind(payment.status)
        .bind(&payment.metadata)
        .fetch_one(&self.pool)
        .await?;
        
        Ok(result)
    }
    
    async fn find_by_idempotency_key(&self, key: &str) -> Result<Option<Payment>> {
        let result = sqlx::query_as::<_, Payment>(
            "SELECT * FROM payments WHERE idempotency_key = $1"
        )
        .bind(key)
        .fetch_optional(&self.pool)
        .await?;
        
        Ok(result)
    }
}

// src/payment/service.rs
use tracing::instrument;

pub struct PaymentService {
    repository: Arc<dyn PaymentRepository>,
    stripe_client: Arc<StripeClient>,
}

impl PaymentService {
    #[instrument(skip(self), fields(amount = %request.amount))]
    pub async fn create_payment(&self, request: CreatePaymentRequest) -> Result<Payment> {
        // Idempotency check
        if let Some(key) = &request.idempotency_key {
            if let Some(existing) = self.repository.find_by_idempotency_key(key).await? {
                info!("Payment already exists for idempotency key: {}", key);
                return Ok(existing);
            }
        }
        
        // Create Stripe PaymentIntent
        let intent = self.stripe_client.create_payment_intent(
            request.amount,
            &request.currency,
        ).await?;
        
        let payment = Payment {
            id: Uuid::new_v4(),
            stripe_payment_intent_id: intent.id,
            idempotency_key: request.idempotency_key,
            amount: request.amount,
            currency: request.currency,
            status: PaymentStatus::Pending,
            metadata: None,
            created_at: Utc::now(),
            updated_at: Utc::now(),
        };
        
        self.repository.create(&payment).await
    }
}

// src/stripe/webhook.rs
use hmac::{Hmac, Mac};
use sha2::Sha256;

type HmacSha256 = Hmac<Sha256>;

pub fn verify_stripe_signature(
    payload: &[u8],
    signature: &str,
    secret: &str,
) -> Result<()> {
    let parts: Vec<&str> = signature.split(',').collect();
    let mut timestamp = None;
    let mut signatures = Vec::new();
    
    for part in parts {
        if let Some(t) = part.strip_prefix("t=") {
            timestamp = Some(t);
        } else if let Some(v1) = part.strip_prefix("v1=") {
            signatures.push(v1);
        }
    }
    
    let timestamp = timestamp.ok_or(Error::InvalidSignature)?;
    let signed_payload = format!("{}.{}", timestamp, String::from_utf8_lossy(payload));
    
    let mut mac = HmacSha256::new_from_slice(secret.as_bytes())
        .map_err(|_| Error::InvalidSignature)?;
    mac.update(signed_payload.as_bytes());
    
    let expected_signature = hex::encode(mac.finalize().into_bytes());
    
    if !signatures.iter().any(|sig| sig == &expected_signature) {
        return Err(Error::InvalidSignature);
    }
    
    Ok(())
}

#[instrument(skip(payload, service))]
pub async fn handle_stripe_webhook(
    signature: String,
    payload: Vec<u8>,
    service: Arc<PaymentService>,
) -> Result<()> {
    verify_stripe_signature(&payload, &signature, &CONFIG.stripe_webhook_secret)?;
    
    let event: StripeEvent = serde_json::from_slice(&payload)?;
    
    match event.event_type.as_str() {
        "payment_intent.succeeded" => {
            let intent_id = event.data.object.id;
            service.handle_payment_succeeded(&intent_id).await?;
        }
        "payment_intent.payment_failed" => {
            let intent_id = event.data.object.id;
            service.handle_payment_failed(&intent_id).await?;
        }
        _ => {
            info!("Unhandled webhook event: {}", event.event_type);
        }
    }
    
    Ok(())
}
```

### 7. TESTS REQUERIDOS (8 tests - ALTA prioridad)
```rust
// tests/payment_test.rs

// ALTA priority - Critical payment flow
- [ ] test_create_payment_success
      → Create payment → verify in DB → verify Stripe call

- [ ] test_idempotency_same_key_returns_same_payment
      → Create payment with key_1 → create again with key_1 → same payment returned

- [ ] test_idempotency_different_keys_create_different_payments
      → Create with key_1 → create with key_2 → 2 different payments

- [ ] test_webhook_signature_valid
      → Valid signature → webhook processed

- [ ] test_webhook_signature_invalid
      → Invalid signature → 401 error

- [ ] test_webhook_payment_succeeded_updates_status
      → payment_intent.succeeded event → status = Succeeded

- [ ] test_get_payment_by_id
      → Create payment → GET /payments/:id → payment returned

- [ ] test_health_check
      → GET /health → 200 OK
```

### 8. COMMITS ESPERADOS
```
1. feat(infra): Docker Compose + PostgreSQL + Redis setup
2. feat(payment): Payment domain model + repository trait
3. feat(payment): PaymentService + idempotency logic
4. feat(stripe): Webhook signature verification
5. feat(stripe): Webhook event handlers
6. feat(observability): OpenTelemetry + Jaeger tracing
7. test(payment): Integration tests for payment flow
```

### 9. MÉTRICAS DE ÉXITO
```
✅ docker-compose up -d → todos los servicios running
✅ cargo run → server on port 3000
✅ curl localhost:3000/health → 200 OK
✅ Webhook signature verification working
✅ Idempotency: mismo key = mismo payment (no duplicate charges)
✅ Jaeger UI (localhost:16686) → traces visibles
✅ cargo test → 8/8 tests passing
✅ Zero compiler warnings
```

### 10. DISTRIBUCIÓN DE HORAS (40h)
```
Lunes (8h):
  - Docker Compose setup (2h)
  - PostgreSQL + migrations (2h)
  - Axum server skeleton (2h)
  - Payment models + repository trait (2h)

Martes (8h):
  - PaymentRepository SQLx implementation (3h)
  - PaymentService business logic (3h)
  - Health check endpoint (1h)
  - Error handling (1h)

Miércoles (8h):
  - Stripe webhook signature verification (3h)
  - Webhook event handlers (3h)
  - Idempotency logic (2h)

Jueves (8h):
  - REST endpoints (/payments, /webhooks/stripe) (3h)
  - OpenTelemetry + Jaeger setup (3h)
  - Tracing instrumentation (#[instrument]) (2h)

Viernes (8h):
  - Integration tests (8 tests) (6h)
  - Testing idempotency scenarios (2h)
```

---
