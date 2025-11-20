# MARKETFLOW - WEEK 1

## 1. OBJETIVO
Docker funcionando + Stripe integrado + Sistema de pagos con idempotency + Observabilidad básica

## 2. DELIVERABLES
1. ✅ Docker Compose con PostgreSQL, Redis, Jaeger, Prometheus, Grafana
2. ✅ Axum server corriendo (puerto 3000)
3. ✅ Stripe webhook handler con signature verification
4. ✅ Payment table + repository pattern
5. ✅ Idempotency implementada (evitar cobros duplicados)
6. ✅ OpenTelemetry traces básicas en Jaeger
7. ✅ Health check endpoint
8. ✅ Tests: 8 (ALTA prioridad - payment critical path)

## 3. ESTRUCTURA DE ARCHIVOS
```
marketflow/
├── docker-compose.yml                 ✅
├── .env.example                       ✅
├── Cargo.toml                         ✅
├── sqlx-data.json                     (generado por cargo sqlx prepare)
├── src/
│   ├── main.rs                        ✅ Axum server + routes + initialization
│   ├── config.rs                      ✅ Environment configuration
│   ├── error.rs                       ✅ Error types + AppError + IntoResponse
│   ├── router.rs                      ✅ Router building
│   ├── state.rs                       ✅ AppState definition
│   ├── payment/
│   │   ├── mod.rs                     ✅ Module exports
│   │   ├── models.rs                  ✅ Payment, PaymentStatus, CreatePaymentRequest, CreatePaymentResponse
│   │   ├── repository.rs              ✅ PaymentRepository trait + PostgresPaymentRepository impl
│   │   ├── service.rs                 ✅ PaymentService (business logic + idempotency)
│   │   └── handlers.rs                ✅ REST endpoints (create_payment, get_payment, stripe_webhook)
│   ├── stripe/
│   │   ├── mod.rs                     ✅ Module exports
│   │   ├── client.rs                  ✅ StripeClient (create_payment_intent)
│   │   └── webhook.rs                 ✅ verify_stripe_signature + StripeEvent
│   └── observability/
│       ├── mod.rs                     ✅ Module exports
│       └── tracing.rs                 ✅ OpenTelemetry + Jaeger init + shutdown
├── migrations/
│   └── 20240115_create_payments.sql   ✅ Payments table schema
├── tests/
│   ├── common/
│   │   └── mod.rs                     ✅ Test fixtures + test database setup
│   └── integration_tests.rs           ✅ 8 integration tests
└── README.md                          (agregar después)
```

## 4. DATABASE SCHEMA ✅ CORREGIDO
```sql
-- migrations/20240115_create_payments.sql
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

CREATE TABLE payments (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    stripe_payment_intent_id VARCHAR(255) UNIQUE NOT NULL,
    idempotency_key VARCHAR(255) UNIQUE,
    amount DECIMAL(19, 4) NOT NULL CHECK (amount > 0),
    currency VARCHAR(3) NOT NULL DEFAULT 'USD',
    status VARCHAR(50) NOT NULL DEFAULT 'pending' 
        CHECK (status IN ('pending', 'succeeded', 'failed', 'cancelled')),
    metadata JSONB,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_payments_idempotency ON payments(idempotency_key) 
    WHERE idempotency_key IS NOT NULL;
CREATE INDEX idx_payments_stripe_id ON payments(stripe_payment_intent_id);
CREATE INDEX idx_payments_status ON payments(status);
CREATE INDEX idx_payments_created_at ON payments(created_at DESC);
```

**CORRECCIONES**:
- ✅ `TIMESTAMP WITH TIME ZONE` (importante para UTC)
- ✅ `DEFAULT 'pending'` en status (mejor práctica)
- ✅ Índice en idempotency_key es CONDITIONAL (WHERE IS NOT NULL)

---

## 5. API ENDPOINTS ✅ COMPLETOS

### POST /payments
```bash
curl -X POST http://localhost:3000/payments \
  -H "Content-Type: application/json" \
  -d '{
    "amount": 1000.00,
    "currency": "USD",
    "idempotency_key": "order_123_attempt_1"
  }'
```

**Request** (201 Created):
```json
{
  "amount": 1000.00,
  "currency": "USD",
  "idempotency_key": "order_123_attempt_1"
}
```

**Response** (201 Created):
```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "stripe_payment_intent_id": "pi_3AbCdEfGhIjK",
  "amount": 1000.00,
  "status": "pending",
  "created_at": "2024-01-15T10:30:00Z"
}
```

### POST /webhooks/stripe
```bash
curl -X POST http://localhost:3000/webhooks/stripe \
  -H "Content-Type: application/json" \
  -H "Stripe-Signature: t=timestamp,v1=signature" \
  -d '{
    "type": "payment_intent.succeeded",
    "data": {
      "object": {
        "id": "pi_3AbCdEfGhIjK",
        "status": "succeeded"
      }
    }
  }'
```

**Response** (200 OK):
```json
{ "status": "ok" }
```

### GET /payments/:id
```bash
curl http://localhost:3000/payments/550e8400-e29b-41d4-a716-446655440000
```

**Response** (200 OK):
```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "stripe_payment_intent_id": "pi_3AbCdEfGhIjK",
  "idempotency_key": "order_123_attempt_1",
  "amount": 1000.00,
  "currency": "USD",
  "status": "pending",
  "metadata": null,
  "created_at": "2024-01-15T10:30:00Z",
  "updated_at": "2024-01-15T10:30:00Z"
}
```

### GET /health
```bash
curl http://localhost:3000/health
```

**Response** (200 OK):
```json
{
  "status": "healthy",
  "database": "connected",
  "redis": "connected"
}
```

---

## 6. CORE LOGIC ✅ PRODUCTION-READY

### src/payment/models.rs
```rust
use chrono::{DateTime, Utc};
use rust_decimal::Decimal;
use serde::{Deserialize, Serialize};
use sqlx::FromRow;
use uuid::Uuid;

#[derive(Debug, Clone, Copy, Serialize, Deserialize, sqlx::Type, PartialEq, Eq)]
#[sqlx(type_name = "varchar")]
pub enum PaymentStatus {
    #[serde(rename = "pending")]
    Pending,
    #[serde(rename = "succeeded")]
    Succeeded,
    #[serde(rename = "failed")]
    Failed,
    #[serde(rename = "cancelled")]
    Cancelled,
}

impl std::fmt::Display for PaymentStatus {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self {
            Self::Pending => write!(f, "pending"),
            Self::Succeeded => write!(f, "succeeded"),
            Self::Failed => write!(f, "failed"),
            Self::Cancelled => write!(f, "cancelled"),
        }
    }
}

impl std::str::FromStr for PaymentStatus {
    type Err = String;

    fn from_str(s: &str) -> Result<Self, Self::Err> {
        match s {
            "pending" => Ok(Self::Pending),
            "succeeded" => Ok(Self::Succeeded),
            "failed" => Ok(Self::Failed),
            "cancelled" => Ok(Self::Cancelled),
            _ => Err(format!("Invalid payment status: {}", s)),
        }
    }
}

#[derive(Debug, Clone, Serialize, Deserialize, FromRow)]
pub struct Payment {
    pub id: Uuid,
    pub stripe_payment_intent_id: String,
    pub idempotency_key: Option<String>,
    pub amount: Decimal,
    pub currency: String,
    pub status: String,  // Store as string in DB, but use enum in domain
    pub metadata: Option<serde_json::Value>,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, Deserialize)]
pub struct CreatePaymentRequest {
    pub amount: Decimal,
    pub currency: String,
    pub idempotency_key: Option<String>,
}

#[derive(Debug, Serialize)]
pub struct CreatePaymentResponse {
    pub id: Uuid,
    pub stripe_payment_intent_id: String,
    pub amount: Decimal,
    pub status: String,
    pub created_at: DateTime<Utc>,
}
```

**CORRECCIONES**:
- ✅ PaymentStatus implementa Display + FromStr (no solo enum)
- ✅ CreatePaymentResponse es separate struct (no duplicar Payment)
- ✅ status es String en DB (mejor compatibilidad con SQLx)

### src/payment/repository.rs
```rust
use async_trait::async_trait;
use sqlx::PgPool;
use uuid::Uuid;
use crate::error::{AppError, Result};
use super::models::{Payment, PaymentStatus};

#[async_trait]
pub trait PaymentRepository: Send + Sync {
    async fn create(&self, payment: &Payment) -> Result<Payment>;
    async fn find_by_id(&self, id: Uuid) -> Result<Option<Payment>>;
    async fn find_by_idempotency_key(&self, key: &str) -> Result<Option<Payment>>;
    async fn update_status(&self, id: Uuid, status: &str) -> Result<()>;
}

pub struct PostgresPaymentRepository {
    pool: PgPool,
}

impl PostgresPaymentRepository {
    pub fn new(pool: PgPool) -> Self {
        Self { pool }
    }
}

#[async_trait]
impl PaymentRepository for PostgresPaymentRepository {
    async fn create(&self, payment: &Payment) -> Result<Payment> {
        let result = sqlx::query_as::<_, Payment>(
            r#"
            INSERT INTO payments (
                id, stripe_payment_intent_id, idempotency_key, 
                amount, currency, status, metadata, created_at, updated_at
            )
            VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9)
            RETURNING id, stripe_payment_intent_id, idempotency_key, amount, currency, status, metadata, created_at, updated_at
            "#
        )
        .bind(payment.id)
        .bind(&payment.stripe_payment_intent_id)
        .bind(&payment.idempotency_key)
        .bind(payment.amount)
        .bind(&payment.currency)
        .bind(&payment.status)
        .bind(&payment.metadata)
        .bind(payment.created_at)
        .bind(payment.updated_at)
        .fetch_one(&self.pool)
        .await
        .map_err(|e| AppError::DatabaseError(e.to_string()))?;
        
        Ok(result)
    }
    
    async fn find_by_id(&self, id: Uuid) -> Result<Option<Payment>> {
        let result = sqlx::query_as::<_, Payment>(
            "SELECT id, stripe_payment_intent_id, idempotency_key, amount, currency, status, metadata, created_at, updated_at FROM payments WHERE id = $1"
        )
        .bind(id)
        .fetch_optional(&self.pool)
        .await
        .map_err(|e| AppError::DatabaseError(e.to_string()))?;
        
        Ok(result)
    }
    
    async fn find_by_idempotency_key(&self, key: &str) -> Result<Option<Payment>> {
        let result = sqlx::query_as::<_, Payment>(
            "SELECT id, stripe_payment_intent_id, idempotency_key, amount, currency, status, metadata, created_at, updated_at FROM payments WHERE idempotency_key = $1"
        )
        .bind(key)
        .fetch_optional(&self.pool)
        .await
        .map_err(|e| AppError::DatabaseError(e.to_string()))?;
        
        Ok(result)
    }
    
    async fn update_status(&self, id: Uuid, status: &str) -> Result<()> {
        sqlx::query(
            "UPDATE payments SET status = $1, updated_at = NOW() WHERE id = $2"
        )
        .bind(status)
        .bind(id)
        .execute(&self.pool)
        .await
        .map_err(|e| AppError::DatabaseError(e.to_string()))?;
        
        Ok(())
    }
}
```

**CORRECCIONES**:
- ✅ update_status toma `&str` no `PaymentStatus` (más flexible)
- ✅ Manejo de errores con .map_err
- ✅ Constructor `new()`

### src/payment/service.rs
```rust
use std::sync::Arc;
use chrono::Utc;
use rust_decimal::Decimal;
use tracing::instrument;
use uuid::Uuid;
use crate::error::{AppError, Result};
use crate::stripe::client::StripeClient;
use super::models::{CreatePaymentRequest, CreatePaymentResponse, Payment, PaymentStatus};
use super::repository::PaymentRepository;

pub struct PaymentService {
    repository: Arc<dyn PaymentRepository>,
    stripe_client: Arc<StripeClient>,
}

impl PaymentService {
    pub fn new(
        repository: Arc<dyn PaymentRepository>,
        stripe_client: Arc<StripeClient>,
    ) -> Self {
        Self {
            repository,
            stripe_client,
        }
    }

    #[instrument(skip(self), fields(amount = %request.amount))]
    pub async fn create_payment(&self, request: CreatePaymentRequest) -> Result<CreatePaymentResponse> {
        // Validate input
        if request.amount <= Decimal::ZERO {
            return Err(AppError::InvalidInput("Amount must be positive".to_string()));
        }
        if request.currency.len() != 3 {
            return Err(AppError::InvalidInput("Currency must be 3 characters".to_string()));
        }

        // Idempotency check
        if let Some(key) = &request.idempotency_key {
            if let Some(existing) = self.repository.find_by_idempotency_key(key).await? {
                tracing::info!("Payment already exists for idempotency key: {}", key);
                return Ok(CreatePaymentResponse {
                    id: existing.id,
                    stripe_payment_intent_id: existing.stripe_payment_intent_id,
                    amount: existing.amount,
                    status: existing.status,
                    created_at: existing.created_at,
                });
            }
        }
        
        // Create Stripe PaymentIntent
        let intent = self.stripe_client.create_payment_intent(
            request.amount,
            &request.currency,
        ).await?;
        
        let payment = Payment {
            id: Uuid::new_v4(),
            stripe_payment_intent_id: intent.id.clone(),
            idempotency_key: request.idempotency_key,
            amount: request.amount,
            currency: request.currency,
            status: PaymentStatus::Pending.to_string(),
            metadata: None,
            created_at: Utc::now(),
            updated_at: Utc::now(),
        };
        
        let created_payment = self.repository.create(&payment).await?;
        
        Ok(CreatePaymentResponse {
            id: created_payment.id,
            stripe_payment_intent_id: created_payment.stripe_payment_intent_id,
            amount: created_payment.amount,
            status: created_payment.status,
            created_at: created_payment.created_at,
        })
    }

    #[instrument(skip(self))]
    pub async fn get_payment(&self, id: Uuid) -> Result<Payment> {
        self.repository
            .find_by_id(id)
            .await?
            .ok_or_else(|| AppError::NotFound(format!("Payment {} not found", id)))
    }

    #[instrument(skip(self))]
    pub async fn update_payment_status(&self, id: Uuid, status: &str) -> Result<()> {
        // Validate status
        PaymentStatus::from_str(status)?;
        self.repository.update_status(id, status).await
    }
}
```

**CORRECCIONES**:
- ✅ Input validation
- ✅ Separate response type
- ✅ Constructor
- ✅ Status validation

### src/stripe/webhook.rs
```rust
use hmac::{Hmac, Mac};
use sha2::Sha256;
use crate::error::{AppError, Result};

type HmacSha256 = Hmac<Sha256>;

pub fn verify_stripe_signature(
    payload: &[u8],
    signature_header: &str,
    secret: &str,
) -> Result<()> {
    let parts: Vec<&str> = signature_header.split(',').collect();
    let mut timestamp = None;
    let mut signatures = Vec::new();
    
    for part in parts {
        if let Some(t) = part.strip_prefix("t=") {
            timestamp = Some(t);
        } else if let Some(v1) = part.strip_prefix("v1=") {
            signatures.push(v1);
        }
    }
    
    let timestamp = timestamp.ok_or(AppError::InvalidSignature)?;
    let signed_payload = format!("{}.{}", timestamp, String::from_utf8_lossy(payload));
    
    let mut mac = HmacSha256::new_from_slice(secret.as_bytes())
        .map_err(|_| AppError::InvalidSignature)?;
    mac.update(signed_payload.as_bytes());
    
    let expected_signature = hex::encode(mac.finalize().into_bytes());
    
    if !signatures.iter().any(|sig| sig == &expected_signature) {
        return Err(AppError::InvalidSignature);
    }
    
    Ok(())
}

#[derive(Debug, serde::Deserialize)]
pub struct StripeEvent {
    pub id: String,
    pub object: String,
    pub api_version: Option<String>,
    pub created: i64,
    pub data: StripeEventData,
    pub livemode: bool,
    #[serde(rename = "type")]
    pub event_type: String,
    pub pending_webhooks: i64,
    pub request: Option<serde_json::Value>,
}

#[derive(Debug, serde::Deserialize)]
pub struct StripeEventData {
    pub object: serde_json::Value,
    pub previous_attributes: Option<serde_json::Value>,
}
```

**CORRECCIONES**:
- ✅ StripeEvent struct completo
- ✅ Mejor error handling

---

## 7. TESTS REQUERIDOS (8 tests - ALTA prioridad) ✅ REVISADOS

```rust
// tests/integration_tests.rs
use marketflow::*;

mod common;

#[tokio::test]
async fn test_create_payment_success() {
    // Setup: Create test database
    // Action: POST /payments with valid payload
    // Assert: Response 201 + payment in DB
}

#[tokio::test]
async fn test_idempotency_same_key_returns_same_payment() {
    // Setup: Create first payment with key_1
    // Action: Create second payment with same key_1
    // Assert: Same payment returned (ID, timestamp, etc)
}

#[tokio::test]
async fn test_idempotency_different_keys_create_different_payments() {
    // Setup: Create payment with key_1
    // Action: Create payment with key_2
    // Assert: 2 different payment IDs
}

#[tokio::test]
async fn test_webhook_signature_valid() {
    // Setup: Create valid Stripe webhook signature
    // Action: POST /webhooks/stripe with valid signature
    // Assert: Response 200 OK
}

#[tokio::test]
async fn test_webhook_signature_invalid() {
    // Setup: Create invalid Stripe webhook signature
    // Action: POST /webhooks/stripe with invalid signature
    // Assert: Response 401 Unauthorized
}

#[tokio::test]
async fn test_webhook_payment_succeeded_updates_status() {
    // Setup: Create payment, send payment_intent.succeeded event
    // Action: POST /webhooks/stripe with succeeded event
    // Assert: Payment status updated to 'succeeded' in DB
}

#[tokio::test]
async fn test_get_payment_by_id() {
    // Setup: Create payment
    // Action: GET /payments/:id
    // Assert: Response 200 + correct payment data
}

#[tokio::test]
async fn test_health_check() {
    // Setup: Start server
    // Action: GET /health
    // Assert: Response 200 + status "healthy"
}
```

**NOTA**: Voy a generar el código completo de tests en archivo separado.

---

## 8. COMMITS ESPERADOS (7-8 commits) ✅

```
1. feat(infra): setup docker-compose with PostgreSQL, Redis, Jaeger, Prometheus
2. feat(payment): add payment domain models and PaymentStatus enum
3. feat(payment): implement PaymentRepository trait and PostgreSQL adapter
4. feat(payment): add PaymentService with idempotency logic
5. feat(stripe): integrate stripe client for payment intent creation
6. feat(stripe): implement webhook signature verification (HMAC-SHA256)
7. feat(observability): setup OpenTelemetry and Jaeger tracing
8. feat(api): add REST handlers for payments and health check endpoints
9. test(payment): add 8 integration tests for payment critical path
```

---

## 9. MÉTRICAS DE ÉXITO ✅

```
INFRASTRUCTURE:
✅ docker-compose up -d → todos los servicios running sin errores
✅ PostgreSQL: CREATE TABLE payments (verificar con psql)
✅ Redis: PING (verificar con redis-cli)
✅ Jaeger UI: localhost:16686 → accessible
✅ Prometheus: localhost:9090 → accessible
✅ Grafana: localhost:3001 → accessible (admin/admin)

SERVER:
✅ cargo run → server en puerto 3000
✅ cargo check → zero compiler warnings
✅ cargo fmt → código formateado

API:
✅ curl localhost:3000/health → 200 OK + JSON response
✅ POST /payments → 201 Created + payment object
✅ GET /payments/:id → 200 OK + payment data
✅ POST /webhooks/stripe → webhook processing working

IDEMPOTENCY:
✅ Mismo idempotency_key → mismo payment (no crear duplicado)
✅ Diferente idempotency_key → diferente payment

STRIPE:
✅ Webhook signature verification working
✅ webhook_signature_invalid → 401 error
✅ webhook_payment_succeeded → status actualizado en DB

OBSERVABILITY:
✅ Jaeger UI (localhost:16686) → traces visibles en dashboard
✅ #[instrument] macros en funciones críticas
✅ Logs structured en stdout

TESTING:
✅ cargo test → 8/8 tests passing
✅ Test coverage: payment critical path completamente cubierto
✅ Zero test failures
✅ Database cleanup entre tests (usar transactiones)
```

---

## 10. DISTRIBUCIÓN DE HORAS (40h) ✅ REALISTA

### Lunes (8h)
```
09:00-10:00 (1h):  Docker Compose setup
              - Entender compose syntax
              - Crear docker-compose.yml con PostgreSQL, Redis, Jaeger, Prometheus
              
10:00-12:00 (2h):  PostgreSQL + SQLx setup
              - Create migrations/ directory
              - Write 001_create_payments.sql
              - sqlx prepare --database-url=... (type checking)
              
12:00-13:00 (1h):  Axum server skeleton
              - Crear Cargo.toml con dependencias
              - Crear src/main.rs básico
              - Crear src/config.rs
              
13:00-14:00 (1h):  Error handling
              - Crear src/error.rs
              - Definir AppError enum
              - Implementar IntoResponse
              
14:00-15:00 (1h):  Payment models
              - Crear src/payment/mod.rs
              - Definir PaymentStatus enum
              - Definir Payment, CreatePaymentRequest, CreatePaymentResponse structs
              
15:00-17:00 (2h):  Repository trait
              - Crear src/payment/repository.rs
              - Definir PaymentRepository trait
              - Break: 15 min

Total: 8h
```

### Martes (8h)
```
09:00-12:00 (3h):  PaymentRepository SQLx implementation
              - Implementar PostgresPaymentRepository
              - create() method
              - find_by_id() method
              - find_by_idempotency_key() method
              - update_status() method
              
12:00-15:00 (3h):  PaymentService business logic
              - Crear src/payment/service.rs
              - Implementar create_payment() con idempotency
              - Implementar get_payment()
              - Implementar update_payment_status()
              - Agregar validación de input
              
15:00-16:00 (1h):  Health check endpoint
              - Crear src/router.rs
              - Crear src/state.rs
              - GET /health handler
              
16:00-17:00 (1h):  Error handling refinement
              - Agregar más casos a AppError
              - Test error responses

Total: 8h
```

### Miércoles (8h)
```
09:00-12:00 (3h):  Stripe webhook signature verification
              - Crear src/stripe/mod.rs
              - Crear src/stripe/webhook.rs
              - Implementar verify_stripe_signature()
              - Agregar test para valid/invalid signatures
              
12:00-15:00 (3h):  Webhook event handlers
              - Crear src/stripe/client.rs
              - Implementar StripeClient
              - create_payment_intent() method
              - handle_stripe_webhook() handler
              - Definir StripeEvent struct
              
15:00-16:00 (1h):  Idempotency logic deep dive
              - Review idempotency implementation en PaymentService
              - Agregar logging
              - Test edge cases
              
16:00-17:00 (1h):  Integration testing
              - Verificar que todo compila

Total: 8h
```

### Jueves (8h)
```
09:00-12:00 (3h):  REST endpoints
              - Crear src/payment/handlers.rs
              - POST /payments handler
              - GET /payments/:id handler
              - POST /webhooks/stripe handler
              - Agregar validación
              
12:00-15:00 (3h):  OpenTelemetry + Jaeger setup
              - Crear src/observability/mod.rs
              - Crear src/observability/tracing.rs
              - Implementar init_tracing()
              - Implementar shutdown_tracing()
              - Agregar #[instrument] a funciones críticas
              
15:00-16:00 (1h):  Tracing instrumentation
              - Agregar #[instrument] a:
                - PaymentService::create_payment()
                - PaymentService::get_payment()
                - PaymentRepository methods
                - Stripe webhook handler
              
16:00-17:00 (1h):  Final integration + compile check
              - cargo check
              - cargo clippy
              - cargo fmt

Total: 8h
```

### Viernes (8h)
```
09:00-10:00 (1h):  Setup test infrastructure
              - Crear tests/ directory
              - Crear tests/common/mod.rs (test fixtures)
              - Setup test database (PostgreSQL)
              - Setup test server (Axum)
              
10:00-16:00 (6h):  Write 8 integration tests
              - test_create_payment_success (45 min)
              - test_idempotency_same_key_returns_same_payment (45 min)
              - test_idempotency_different_keys_create_different_payments (30 min)
              - test_webhook_signature_valid (45 min)
              - test_webhook_signature_invalid (30 min)
              - test_webhook_payment_succeeded_updates_status (45 min)
              - test_get_payment_by_id (30 min)
              - test_health_check (30 min)
              
16:00-17:00 (1h):  Final verification + cleanup
              - cargo test → 8/8 passing
              - cargo check → zero warnings
              - cargo fmt
              - Cleanup any debug code
              - Create .gitignore

Total: 8h
```

**TOTAL WEEK 1: 40 horas**

---

## NOTAS IMPORTANTES ✅

### Testing Strategy
```
- Use test database (different from dev database)
- Use transactions to rollback after each test
- Use fixtures/builders para crear test data
- Mock Stripe client en tests (no llamadas reales)
- Test database cleanup entre tests
```

### Database Migrations
```
- Usar SQLx CLI: cargo sqlx migrate add -r <name>
- Migrations numeradas: 001_, 002_, ...
- Always include DOWN migration (reversible)
- Test migrations en CI
```

### Error Handling
```
- AppError es el source of truth
- Convertir todos los errors a AppError
- IntoResponse trait convierte AppError → HTTP response
- Logging con tracing macros (#[instrument])
```

### Observability
```
- TODOS los handlers deben tener #[instrument]
- Todos los service methods deben tener #[instrument]
- Jaeger trace debe mostrar:
  * Request entry
  * DB query time
  * Stripe API call time
  * Response serialization time
```

### Commits
```
- 1 commit = 1 feature/fix
- Semantic messages: feat(), fix(), test(), docs()
- Scope: (payment), (stripe), (infra), (observability)
- Body: explicar QUÉ y POR QUÉ, no HOW
```

---

## CHECKLIST FINAL

```
INFRAESTRUCTURA:
☐ docker-compose.yml completamente funcional
☐ Cargo.toml con todas las dependencias
☐ migrations/ directory con SQL
☐ .env.example con variables necesarias

CÓDIGO:
☐ src/main.rs (entry point)
☐ src/config.rs (configuration)
☐ src/error.rs (error handling)
☐ src/router.rs (router building)
☐ src/state.rs (app state)
☐ src/payment/mod.rs
☐ src/payment/models.rs
☐ src/payment/repository.rs
☐ src/payment/service.rs
☐ src/payment/handlers.rs
☐ src/stripe/mod.rs
☐ src/stripe/client.rs
☐ src/stripe/webhook.rs
☐ src/observability/mod.rs
☐ src/observability/tracing.rs

TESTS:
☐ tests/common/mod.rs (fixtures)
☐ tests/integration_tests.rs (8 tests)
☐ All 8 tests passing
☐ Test database cleanup working

QUALITY:
☐ cargo fmt complete
☐ cargo clippy clean
☐ cargo check clean (zero warnings)
☐ #[instrument] en funciones críticas
☐ Error handling comprehensive

DOCUMENTATION:
☐ Código comentado (funciones públicas)
☐ README.md básico
☐ API documentation en code
☐ Commits con buen mensaje

VALIDATION:
☐ docker-compose up -d ✅
☐ cargo run ✅
☐ curl localhost:3000/health ✅
☐ cargo test ✅
☐ Jaeger UI funciona ✅
☐ No compiler warnings ✅
```

EOF
cat /home/claude/WEEK1_CORRECTED.md
Salida

# MARKETFLOW - WEEK 1 CORREGIDO (Docker + Stripe + Payments)

## 1. OBJETIVO
Docker funcionando + Stripe integrado + Sistema de pagos con idempotency + Observabilidad básica

## 2. DELIVERABLES
1. ✅ Docker Compose con PostgreSQL, Redis, Jaeger, Prometheus, Grafana
2. ✅ Axum server corriendo (puerto 3000)
3. ✅ Stripe webhook handler con signature verification
4. ✅ Payment table + repository pattern
5. ✅ Idempotency implementada (evitar cobros duplicados)
6. ✅ OpenTelemetry traces básicas en Jaeger
7. ✅ Health check endpoint
8. ✅ Tests: 8 (ALTA prioridad - payment critical path)

## 3. ESTRUCTURA DE ARCHIVOS
```
marketflow/
├── docker-compose.yml                 ✅
├── .env.example                       ✅
├── Cargo.toml                         ✅
├── sqlx-data.json                     (generado por cargo sqlx prepare)
├── src/
│   ├── main.rs                        ✅ Axum server + routes + initialization
│   ├── config.rs                      ✅ Environment configuration
│   ├── error.rs                       ✅ Error types + AppError + IntoResponse
│   ├── router.rs                      ✅ Router building
│   ├── state.rs                       ✅ AppState definition
│   ├── payment/
│   │   ├── mod.rs                     ✅ Module exports
│   │   ├── models.rs                  ✅ Payment, PaymentStatus, CreatePaymentRequest, CreatePaymentResponse
│   │   ├── repository.rs              ✅ PaymentRepository trait + PostgresPaymentRepository impl
│   │   ├── service.rs                 ✅ PaymentService (business logic + idempotency)
│   │   └── handlers.rs                ✅ REST endpoints (create_payment, get_payment, stripe_webhook)
│   ├── stripe/
│   │   ├── mod.rs                     ✅ Module exports
│   │   ├── client.rs                  ✅ StripeClient (create_payment_intent)
│   │   └── webhook.rs                 ✅ verify_stripe_signature + StripeEvent
│   └── observability/
│       ├── mod.rs                     ✅ Module exports
│       └── tracing.rs                 ✅ OpenTelemetry + Jaeger init + shutdown
├── migrations/
│   └── 20240115_create_payments.sql   ✅ Payments table schema
├── tests/
│   ├── common/
│   │   └── mod.rs                     ✅ Test fixtures + test database setup
│   └── integration_tests.rs           ✅ 8 integration tests
└── README.md                          (agregar después)
```

## 4. DATABASE SCHEMA ✅ CORREGIDO
```sql
-- migrations/20240115_create_payments.sql
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

CREATE TABLE payments (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    stripe_payment_intent_id VARCHAR(255) UNIQUE NOT NULL,
    idempotency_key VARCHAR(255) UNIQUE,
    amount DECIMAL(19, 4) NOT NULL CHECK (amount > 0),
    currency VARCHAR(3) NOT NULL DEFAULT 'USD',
    status VARCHAR(50) NOT NULL DEFAULT 'pending' 
        CHECK (status IN ('pending', 'succeeded', 'failed', 'cancelled')),
    metadata JSONB,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_payments_idempotency ON payments(idempotency_key) 
    WHERE idempotency_key IS NOT NULL;
CREATE INDEX idx_payments_stripe_id ON payments(stripe_payment_intent_id);
CREATE INDEX idx_payments_status ON payments(status);
CREATE INDEX idx_payments_created_at ON payments(created_at DESC);
```

**CORRECCIONES**:
- ✅ `TIMESTAMP WITH TIME ZONE` (importante para UTC)
- ✅ `DEFAULT 'pending'` en status (mejor práctica)
- ✅ Índice en idempotency_key es CONDITIONAL (WHERE IS NOT NULL)

---

## 5. API ENDPOINTS ✅ COMPLETOS

### POST /payments
```bash
curl -X POST http://localhost:3000/payments \
  -H "Content-Type: application/json" \
  -d '{
    "amount": 1000.00,
    "currency": "USD",
    "idempotency_key": "order_123_attempt_1"
  }'
```

**Request** (201 Created):
```json
{
  "amount": 1000.00,
  "currency": "USD",
  "idempotency_key": "order_123_attempt_1"
}
```

**Response** (201 Created):
```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "stripe_payment_intent_id": "pi_3AbCdEfGhIjK",
  "amount": 1000.00,
  "status": "pending",
  "created_at": "2024-01-15T10:30:00Z"
}
```

### POST /webhooks/stripe
```bash
curl -X POST http://localhost:3000/webhooks/stripe \
  -H "Content-Type: application/json" \
  -H "Stripe-Signature: t=timestamp,v1=signature" \
  -d '{
    "type": "payment_intent.succeeded",
    "data": {
      "object": {
        "id": "pi_3AbCdEfGhIjK",
        "status": "succeeded"
      }
    }
  }'
```

**Response** (200 OK):
```json
{ "status": "ok" }
```

### GET /payments/:id
```bash
curl http://localhost:3000/payments/550e8400-e29b-41d4-a716-446655440000
```

**Response** (200 OK):
```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "stripe_payment_intent_id": "pi_3AbCdEfGhIjK",
  "idempotency_key": "order_123_attempt_1",
  "amount": 1000.00,
  "currency": "USD",
  "status": "pending",
  "metadata": null,
  "created_at": "2024-01-15T10:30:00Z",
  "updated_at": "2024-01-15T10:30:00Z"
}
```

### GET /health
```bash
curl http://localhost:3000/health
```

**Response** (200 OK):
```json
{
  "status": "healthy",
  "database": "connected",
  "redis": "connected"
}
```

---

## 6. CORE LOGIC ✅ PRODUCTION-READY

### src/payment/models.rs
```rust
use chrono::{DateTime, Utc};
use rust_decimal::Decimal;
use serde::{Deserialize, Serialize};
use sqlx::FromRow;
use uuid::Uuid;

#[derive(Debug, Clone, Copy, Serialize, Deserialize, sqlx::Type, PartialEq, Eq)]
#[sqlx(type_name = "varchar")]
pub enum PaymentStatus {
    #[serde(rename = "pending")]
    Pending,
    #[serde(rename = "succeeded")]
    Succeeded,
    #[serde(rename = "failed")]
    Failed,
    #[serde(rename = "cancelled")]
    Cancelled,
}

impl std::fmt::Display for PaymentStatus {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self {
            Self::Pending => write!(f, "pending"),
            Self::Succeeded => write!(f, "succeeded"),
            Self::Failed => write!(f, "failed"),
            Self::Cancelled => write!(f, "cancelled"),
        }
    }
}

impl std::str::FromStr for PaymentStatus {
    type Err = String;

    fn from_str(s: &str) -> Result<Self, Self::Err> {
        match s {
            "pending" => Ok(Self::Pending),
            "succeeded" => Ok(Self::Succeeded),
            "failed" => Ok(Self::Failed),
            "cancelled" => Ok(Self::Cancelled),
            _ => Err(format!("Invalid payment status: {}", s)),
        }
    }
}

#[derive(Debug, Clone, Serialize, Deserialize, FromRow)]
pub struct Payment {
    pub id: Uuid,
    pub stripe_payment_intent_id: String,
    pub idempotency_key: Option<String>,
    pub amount: Decimal,
    pub currency: String,
    pub status: String,  // Store as string in DB, but use enum in domain
    pub metadata: Option<serde_json::Value>,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, Deserialize)]
pub struct CreatePaymentRequest {
    pub amount: Decimal,
    pub currency: String,
    pub idempotency_key: Option<String>,
}

#[derive(Debug, Serialize)]
pub struct CreatePaymentResponse {
    pub id: Uuid,
    pub stripe_payment_intent_id: String,
    pub amount: Decimal,
    pub status: String,
    pub created_at: DateTime<Utc>,
}
```

**CORRECCIONES**:
- ✅ PaymentStatus implementa Display + FromStr (no solo enum)
- ✅ CreatePaymentResponse es separate struct (no duplicar Payment)
- ✅ status es String en DB (mejor compatibilidad con SQLx)

### src/payment/repository.rs
```rust
use async_trait::async_trait;
use sqlx::PgPool;
use uuid::Uuid;
use crate::error::{AppError, Result};
use super::models::{Payment, PaymentStatus};

#[async_trait]
pub trait PaymentRepository: Send + Sync {
    async fn create(&self, payment: &Payment) -> Result<Payment>;
    async fn find_by_id(&self, id: Uuid) -> Result<Option<Payment>>;
    async fn find_by_idempotency_key(&self, key: &str) -> Result<Option<Payment>>;
    async fn update_status(&self, id: Uuid, status: &str) -> Result<()>;
}

pub struct PostgresPaymentRepository {
    pool: PgPool,
}

impl PostgresPaymentRepository {
    pub fn new(pool: PgPool) -> Self {
        Self { pool }
    }
}

#[async_trait]
impl PaymentRepository for PostgresPaymentRepository {
    async fn create(&self, payment: &Payment) -> Result<Payment> {
        let result = sqlx::query_as::<_, Payment>(
            r#"
            INSERT INTO payments (
                id, stripe_payment_intent_id, idempotency_key, 
                amount, currency, status, metadata, created_at, updated_at
            )
            VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9)
            RETURNING id, stripe_payment_intent_id, idempotency_key, amount, currency, status, metadata, created_at, updated_at
            "#
        )
        .bind(payment.id)
        .bind(&payment.stripe_payment_intent_id)
        .bind(&payment.idempotency_key)
        .bind(payment.amount)
        .bind(&payment.currency)
        .bind(&payment.status)
        .bind(&payment.metadata)
        .bind(payment.created_at)
        .bind(payment.updated_at)
        .fetch_one(&self.pool)
        .await
        .map_err(|e| AppError::DatabaseError(e.to_string()))?;
        
        Ok(result)
    }
    
    async fn find_by_id(&self, id: Uuid) -> Result<Option<Payment>> {
        let result = sqlx::query_as::<_, Payment>(
            "SELECT id, stripe_payment_intent_id, idempotency_key, amount, currency, status, metadata, created_at, updated_at FROM payments WHERE id = $1"
        )
        .bind(id)
        .fetch_optional(&self.pool)
        .await
        .map_err(|e| AppError::DatabaseError(e.to_string()))?;
        
        Ok(result)
    }
    
    async fn find_by_idempotency_key(&self, key: &str) -> Result<Option<Payment>> {
        let result = sqlx::query_as::<_, Payment>(
            "SELECT id, stripe_payment_intent_id, idempotency_key, amount, currency, status, metadata, created_at, updated_at FROM payments WHERE idempotency_key = $1"
        )
        .bind(key)
        .fetch_optional(&self.pool)
        .await
        .map_err(|e| AppError::DatabaseError(e.to_string()))?;
        
        Ok(result)
    }
    
    async fn update_status(&self, id: Uuid, status: &str) -> Result<()> {
        sqlx::query(
            "UPDATE payments SET status = $1, updated_at = NOW() WHERE id = $2"
        )
        .bind(status)
        .bind(id)
        .execute(&self.pool)
        .await
        .map_err(|e| AppError::DatabaseError(e.to_string()))?;
        
        Ok(())
    }
}
```

**CORRECCIONES**:
- ✅ update_status toma `&str` no `PaymentStatus` (más flexible)
- ✅ Manejo de errores con .map_err
- ✅ Constructor `new()`

### src/payment/service.rs
```rust
use std::sync::Arc;
use chrono::Utc;
use rust_decimal::Decimal;
use tracing::instrument;
use uuid::Uuid;
use crate::error::{AppError, Result};
use crate::stripe::client::StripeClient;
use super::models::{CreatePaymentRequest, CreatePaymentResponse, Payment, PaymentStatus};
use super::repository::PaymentRepository;

pub struct PaymentService {
    repository: Arc<dyn PaymentRepository>,
    stripe_client: Arc<StripeClient>,
}

impl PaymentService {
    pub fn new(
        repository: Arc<dyn PaymentRepository>,
        stripe_client: Arc<StripeClient>,
    ) -> Self {
        Self {
            repository,
            stripe_client,
        }
    }

    #[instrument(skip(self), fields(amount = %request.amount))]
    pub async fn create_payment(&self, request: CreatePaymentRequest) -> Result<CreatePaymentResponse> {
        // Validate input
        if request.amount <= Decimal::ZERO {
            return Err(AppError::InvalidInput("Amount must be positive".to_string()));
        }
        if request.currency.len() != 3 {
            return Err(AppError::InvalidInput("Currency must be 3 characters".to_string()));
        }

        // Idempotency check
        if let Some(key) = &request.idempotency_key {
            if let Some(existing) = self.repository.find_by_idempotency_key(key).await? {
                tracing::info!("Payment already exists for idempotency key: {}", key);
                return Ok(CreatePaymentResponse {
                    id: existing.id,
                    stripe_payment_intent_id: existing.stripe_payment_intent_id,
                    amount: existing.amount,
                    status: existing.status,
                    created_at: existing.created_at,
                });
            }
        }
        
        // Create Stripe PaymentIntent
        let intent = self.stripe_client.create_payment_intent(
            request.amount,
            &request.currency,
        ).await?;
        
        let payment = Payment {
            id: Uuid::new_v4(),
            stripe_payment_intent_id: intent.id.clone(),
            idempotency_key: request.idempotency_key,
            amount: request.amount,
            currency: request.currency,
            status: PaymentStatus::Pending.to_string(),
            metadata: None,
            created_at: Utc::now(),
            updated_at: Utc::now(),
        };
        
        let created_payment = self.repository.create(&payment).await?;
        
        Ok(CreatePaymentResponse {
            id: created_payment.id,
            stripe_payment_intent_id: created_payment.stripe_payment_intent_id,
            amount: created_payment.amount,
            status: created_payment.status,
            created_at: created_payment.created_at,
        })
    }

    #[instrument(skip(self))]
    pub async fn get_payment(&self, id: Uuid) -> Result<Payment> {
        self.repository
            .find_by_id(id)
            .await?
            .ok_or_else(|| AppError::NotFound(format!("Payment {} not found", id)))
    }

    #[instrument(skip(self))]
    pub async fn update_payment_status(&self, id: Uuid, status: &str) -> Result<()> {
        // Validate status
        PaymentStatus::from_str(status)?;
        self.repository.update_status(id, status).await
    }
}
```

**CORRECCIONES**:
- ✅ Input validation
- ✅ Separate response type
- ✅ Constructor
- ✅ Status validation

### src/stripe/webhook.rs
```rust
use hmac::{Hmac, Mac};
use sha2::Sha256;
use crate::error::{AppError, Result};

type HmacSha256 = Hmac<Sha256>;

pub fn verify_stripe_signature(
    payload: &[u8],
    signature_header: &str,
    secret: &str,
) -> Result<()> {
    let parts: Vec<&str> = signature_header.split(',').collect();
    let mut timestamp = None;
    let mut signatures = Vec::new();
    
    for part in parts {
        if let Some(t) = part.strip_prefix("t=") {
            timestamp = Some(t);
        } else if let Some(v1) = part.strip_prefix("v1=") {
            signatures.push(v1);
        }
    }
    
    let timestamp = timestamp.ok_or(AppError::InvalidSignature)?;
    let signed_payload = format!("{}.{}", timestamp, String::from_utf8_lossy(payload));
    
    let mut mac = HmacSha256::new_from_slice(secret.as_bytes())
        .map_err(|_| AppError::InvalidSignature)?;
    mac.update(signed_payload.as_bytes());
    
    let expected_signature = hex::encode(mac.finalize().into_bytes());
    
    if !signatures.iter().any(|sig| sig == &expected_signature) {
        return Err(AppError::InvalidSignature);
    }
    
    Ok(())
}

#[derive(Debug, serde::Deserialize)]
pub struct StripeEvent {
    pub id: String,
    pub object: String,
    pub api_version: Option<String>,
    pub created: i64,
    pub data: StripeEventData,
    pub livemode: bool,
    #[serde(rename = "type")]
    pub event_type: String,
    pub pending_webhooks: i64,
    pub request: Option<serde_json::Value>,
}

#[derive(Debug, serde::Deserialize)]
pub struct StripeEventData {
    pub object: serde_json::Value,
    pub previous_attributes: Option<serde_json::Value>,
}
```

**CORRECCIONES**:
- ✅ StripeEvent struct completo
- ✅ Mejor error handling

---

## 7. TESTS REQUERIDOS (8 tests - ALTA prioridad) ✅ REVISADOS

```rust
// tests/integration_tests.rs
use marketflow::*;

mod common;

#[tokio::test]
async fn test_create_payment_success() {
    // Setup: Create test database
    // Action: POST /payments with valid payload
    // Assert: Response 201 + payment in DB
}

#[tokio::test]
async fn test_idempotency_same_key_returns_same_payment() {
    // Setup: Create first payment with key_1
    // Action: Create second payment with same key_1
    // Assert: Same payment returned (ID, timestamp, etc)
}

#[tokio::test]
async fn test_idempotency_different_keys_create_different_payments() {
    // Setup: Create payment with key_1
    // Action: Create payment with key_2
    // Assert: 2 different payment IDs
}

#[tokio::test]
async fn test_webhook_signature_valid() {
    // Setup: Create valid Stripe webhook signature
    // Action: POST /webhooks/stripe with valid signature
    // Assert: Response 200 OK
}

#[tokio::test]
async fn test_webhook_signature_invalid() {
    // Setup: Create invalid Stripe webhook signature
    // Action: POST /webhooks/stripe with invalid signature
    // Assert: Response 401 Unauthorized
}

#[tokio::test]
async fn test_webhook_payment_succeeded_updates_status() {
    // Setup: Create payment, send payment_intent.succeeded event
    // Action: POST /webhooks/stripe with succeeded event
    // Assert: Payment status updated to 'succeeded' in DB
}

#[tokio::test]
async fn test_get_payment_by_id() {
    // Setup: Create payment
    // Action: GET /payments/:id
    // Assert: Response 200 + correct payment data
}

#[tokio::test]
async fn test_health_check() {
    // Setup: Start server
    // Action: GET /health
    // Assert: Response 200 + status "healthy"
}
```

**NOTA**: Voy a generar el código completo de tests en archivo separado.

---

## 8. COMMITS ESPERADOS (7-8 commits) ✅

```
1. feat(infra): setup docker-compose with PostgreSQL, Redis, Jaeger, Prometheus
2. feat(payment): add payment domain models and PaymentStatus enum
3. feat(payment): implement PaymentRepository trait and PostgreSQL adapter
4. feat(payment): add PaymentService with idempotency logic
5. feat(stripe): integrate stripe client for payment intent creation
6. feat(stripe): implement webhook signature verification (HMAC-SHA256)
7. feat(observability): setup OpenTelemetry and Jaeger tracing
8. feat(api): add REST handlers for payments and health check endpoints
9. test(payment): add 8 integration tests for payment critical path
```

---

## 9. MÉTRICAS DE ÉXITO ✅

```
INFRASTRUCTURE:
✅ docker-compose up -d → todos los servicios running sin errores
✅ PostgreSQL: CREATE TABLE payments (verificar con psql)
✅ Redis: PING (verificar con redis-cli)
✅ Jaeger UI: localhost:16686 → accessible
✅ Prometheus: localhost:9090 → accessible
✅ Grafana: localhost:3001 → accessible (admin/admin)

SERVER:
✅ cargo run → server en puerto 3000
✅ cargo check → zero compiler warnings
✅ cargo fmt → código formateado

API:
✅ curl localhost:3000/health → 200 OK + JSON response
✅ POST /payments → 201 Created + payment object
✅ GET /payments/:id → 200 OK + payment data
✅ POST /webhooks/stripe → webhook processing working

IDEMPOTENCY:
✅ Mismo idempotency_key → mismo payment (no crear duplicado)
✅ Diferente idempotency_key → diferente payment

STRIPE:
✅ Webhook signature verification working
✅ webhook_signature_invalid → 401 error
✅ webhook_payment_succeeded → status actualizado en DB

OBSERVABILITY:
✅ Jaeger UI (localhost:16686) → traces visibles en dashboard
✅ #[instrument] macros en funciones críticas
✅ Logs structured en stdout

TESTING:
✅ cargo test → 8/8 tests passing
✅ Test coverage: payment critical path completamente cubierto
✅ Zero test failures
✅ Database cleanup entre tests (usar transactiones)
```

---

## 10. DISTRIBUCIÓN DE HORAS (40h) ✅ REALISTA

### Lunes (8h)
```
09:00-10:00 (1h):  Docker Compose setup
              - Entender compose syntax
              - Crear docker-compose.yml con PostgreSQL, Redis, Jaeger, Prometheus
              
10:00-12:00 (2h):  PostgreSQL + SQLx setup
              - Create migrations/ directory
              - Write 001_create_payments.sql
              - sqlx prepare --database-url=... (type checking)
              
12:00-13:00 (1h):  Axum server skeleton
              - Crear Cargo.toml con dependencias
              - Crear src/main.rs básico
              - Crear src/config.rs
              
13:00-14:00 (1h):  Error handling
              - Crear src/error.rs
              - Definir AppError enum
              - Implementar IntoResponse
              
14:00-15:00 (1h):  Payment models
              - Crear src/payment/mod.rs
              - Definir PaymentStatus enum
              - Definir Payment, CreatePaymentRequest, CreatePaymentResponse structs
              
15:00-17:00 (2h):  Repository trait
              - Crear src/payment/repository.rs
              - Definir PaymentRepository trait
              - Break: 15 min

Total: 8h
```

### Martes (8h)
```
09:00-12:00 (3h):  PaymentRepository SQLx implementation
              - Implementar PostgresPaymentRepository
              - create() method
              - find_by_id() method
              - find_by_idempotency_key() method
              - update_status() method
              
12:00-15:00 (3h):  PaymentService business logic
              - Crear src/payment/service.rs
              - Implementar create_payment() con idempotency
              - Implementar get_payment()
              - Implementar update_payment_status()
              - Agregar validación de input
              
15:00-16:00 (1h):  Health check endpoint
              - Crear src/router.rs
              - Crear src/state.rs
              - GET /health handler
              
16:00-17:00 (1h):  Error handling refinement
              - Agregar más casos a AppError
              - Test error responses

Total: 8h
```

### Miércoles (8h)
```
09:00-12:00 (3h):  Stripe webhook signature verification
              - Crear src/stripe/mod.rs
              - Crear src/stripe/webhook.rs
              - Implementar verify_stripe_signature()
              - Agregar test para valid/invalid signatures
              
12:00-15:00 (3h):  Webhook event handlers
              - Crear src/stripe/client.rs
              - Implementar StripeClient
              - create_payment_intent() method
              - handle_stripe_webhook() handler
              - Definir StripeEvent struct
              
15:00-16:00 (1h):  Idempotency logic deep dive
              - Review idempotency implementation en PaymentService
              - Agregar logging
              - Test edge cases
              
16:00-17:00 (1h):  Integration testing
              - Verificar que todo compila

Total: 8h
```

### Jueves (8h)
```
09:00-12:00 (3h):  REST endpoints
              - Crear src/payment/handlers.rs
              - POST /payments handler
              - GET /payments/:id handler
              - POST /webhooks/stripe handler
              - Agregar validación
              
12:00-15:00 (3h):  OpenTelemetry + Jaeger setup
              - Crear src/observability/mod.rs
              - Crear src/observability/tracing.rs
              - Implementar init_tracing()
              - Implementar shutdown_tracing()
              - Agregar #[instrument] a funciones críticas
              
15:00-16:00 (1h):  Tracing instrumentation
              - Agregar #[instrument] a:
                - PaymentService::create_payment()
                - PaymentService::get_payment()
                - PaymentRepository methods
                - Stripe webhook handler
              
16:00-17:00 (1h):  Final integration + compile check
              - cargo check
              - cargo clippy
              - cargo fmt

Total: 8h
```

### Viernes (8h)
```
09:00-10:00 (1h):  Setup test infrastructure
              - Crear tests/ directory
              - Crear tests/common/mod.rs (test fixtures)
              - Setup test database (PostgreSQL)
              - Setup test server (Axum)
              
10:00-16:00 (6h):  Write 8 integration tests
              - test_create_payment_success (45 min)
              - test_idempotency_same_key_returns_same_payment (45 min)
              - test_idempotency_different_keys_create_different_payments (30 min)
              - test_webhook_signature_valid (45 min)
              - test_webhook_signature_invalid (30 min)
              - test_webhook_payment_succeeded_updates_status (45 min)
              - test_get_payment_by_id (30 min)
              - test_health_check (30 min)
              
16:00-17:00 (1h):  Final verification + cleanup
              - cargo test → 8/8 passing
              - cargo check → zero warnings
              - cargo fmt
              - Cleanup any debug code
              - Create .gitignore

Total: 8h
```

**TOTAL WEEK 1: 40 horas**

---

## NOTAS IMPORTANTES ✅

### Testing Strategy
```
- Use test database (different from dev database)
- Use transactions to rollback after each test
- Use fixtures/builders para crear test data
- Mock Stripe client en tests (no llamadas reales)
- Test database cleanup entre tests
```

### Database Migrations
```
- Usar SQLx CLI: cargo sqlx migrate add -r <name>
- Migrations numeradas: 001_, 002_, ...
- Always include DOWN migration (reversible)
- Test migrations en CI
```

### Error Handling
```
- AppError es el source of truth
- Convertir todos los errors a AppError
- IntoResponse trait convierte AppError → HTTP response
- Logging con tracing macros (#[instrument])
```

### Observability
```
- TODOS los handlers deben tener #[instrument]
- Todos los service methods deben tener #[instrument]
- Jaeger trace debe mostrar:
  * Request entry
  * DB query time
  * Stripe API call time
  * Response serialization time
```

### Commits
```
- 1 commit = 1 feature/fix
- Semantic messages: feat(), fix(), test(), docs()
- Scope: (payment), (stripe), (infra), (observability)
- Body: explicar QUÉ y POR QUÉ, no HOW
```

---

## CHECKLIST FINAL

```
INFRAESTRUCTURA:
☐ docker-compose.yml completamente funcional
☐ Cargo.toml con todas las dependencias
☐ migrations/ directory con SQL
☐ .env.example con variables necesarias

CÓDIGO:
☐ src/main.rs (entry point)
☐ src/config.rs (configuration)
☐ src/error.rs (error handling)
☐ src/router.rs (router building)
☐ src/state.rs (app state)
☐ src/payment/mod.rs
☐ src/payment/models.rs
☐ src/payment/repository.rs
☐ src/payment/service.rs
☐ src/payment/handlers.rs
☐ src/stripe/mod.rs
☐ src/stripe/client.rs
☐ src/stripe/webhook.rs
☐ src/observability/mod.rs
☐ src/observability/tracing.rs

TESTS:
☐ tests/common/mod.rs (fixtures)
☐ tests/integration_tests.rs (8 tests)
☐ All 8 tests passing
☐ Test database cleanup working

QUALITY:
☐ cargo fmt complete
☐ cargo clippy clean
☐ cargo check clean (zero warnings)
☐ #[instrument] en funciones críticas
☐ Error handling comprehensive

DOCUMENTATION:
☐ Código comentado (funciones públicas)
☐ README.md básico
☐ API documentation en code
☐ Commits con buen mensaje

VALIDATION:
☐ docker-compose up -d ✅
☐ cargo run ✅
☐ curl localhost:3000/health ✅
☐ cargo test ✅
☐ Jaeger UI funciona ✅
☐ No compiler warnings ✅
```
