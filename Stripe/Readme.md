# Stripe-like Payment Processing System

Payment processing systems like Stripe allow **businesses** (referred to throughout as **merchants**) to accept payments from customers without building their own payment infrastructure. Customers enter their payment details on the merchant's website; the merchant sends the details to the payment system, which processes the payment and returns the result.

---

## Table of Contents

- [Functional Requirements](#functional-requirements)
- [Non-Functional Requirements](#non-functional-requirements)
- [Core Entities](#core-entities)
- [API](#api)
- [High-Level Design](#high-level-design)
- [Deep Dives](#deep-dives)
  - [Security](#1-security)
  - [Durability & Auditability](#2-durability-and-auditability)
  - [Transaction Safety & Financial Integrity](#3-transaction-safety-and-financial-integrity)
  - [Scalability](#4-scalability)
- [Bonus: Webhooks](#bonus-webhooks)
- [WhiteBoarding](https://excalidraw.com/#json=fTtIT0LbP07pvgi4qVgdV,xjreafMtEQnA4fFOxLaGnQ)

---

## Functional Requirements

### In scope

- Merchants can **initiate payment requests** (charge a customer for a specific amount).
- Users can **pay with credit/debit cards**.
- Merchants can **view payment status** (e.g., pending, success, failed).

### Out of scope

- Saving payment methods for future use
- Full or partial refunds
- Transaction history and reports
- Alternative payment methods (bank transfers, digital wallets)
- Recurring payments (subscriptions)
- Payouts to merchants

---

## Non-Functional Requirements

### In scope

- **Highly secure** (auth, data protection).
- **Durable and auditable** — no transaction data lost, even on failures.
- **Transaction safety and financial integrity** despite asynchronous external payment networks.
- **Scalable** to high transaction volume (10,000+ TPS) and bursty traffic (e.g., holiday sales).

### Out of scope

- Global financial regulations (region-dependent)
- Extensibility for new payment methods or features

---

## Core Entities

| Entity | Description |
|--------|-------------|
| **Merchant** | Businesses on the platform: identity, bank account info, API keys. |
| **PaymentIntent** | Merchant’s intention to collect a specific amount. Tracks lifecycle: `created` → `authorized` → `captured` / `canceled` / `refunded`. Enforces idempotency for retries. |
| **Transaction** | Polymorphic money-movement record tied to one PaymentIntent. Types: Charge (funds in), Refund (funds out), Dispute (potential reversal), Payout (merchant withdrawal). Each row has amount, currency, status, timestamps, and references to the intent and merchant. *In-scope we focus on Charges only.* |

> **Note:** In production you’d typically have discrete types (Charge, Refund, Dispute, Payout) and double-entry LedgerEntry rows. For this design, a single polymorphic Transaction is used so one PaymentIntent can have many money-movement events with amount, currency, status, and timestamps—enough to reason about idempotency, auditability, and failure handling.

---

## API

### 1. Create PaymentIntent

```http
POST /payment-intents
```

**Request body:**

```json
{
  "amountInCents": 2499,
  "currency": "usd",
  "description": "Order #1234"
}
```

**Response:** `paymentIntentId`

---

### 2. Create charge (submit card)

```http
POST /payment-intents/{paymentIntentId}/transactions
```

**Request body:**

```json
{
  "type": "charge",
  "card": {
    "number": "4242424242424242",
    "exp_month": 12,
    "exp_year": 2025,
    "cvc": "123"
  }
}
```

---

### 3. Get PaymentIntent status

```http
GET /payment-intents/{paymentIntentId}
```

**Response:** PaymentIntent object (including status and related data).

---

### 4. Webhooks (real-time status)

Merchants can register a callback URL. We POST to it when payment status changes.

```http
POST {merchant_webhook_url}
```

**Example payload:**

```json
{
  "type": "payment.succeeded",
  "data": {
    "paymentId": "pay_123",
    "amountInCents": 2499,
    "currency": "usd",
    "status": "succeeded"
  }
}
```

---

## High-Level Design

### 1. Merchant initiates payment

1. Merchant sends `POST /payment-intents` with amount, currency, description.
2. Request is authenticated and routed to the **PaymentIntent Service**.
3. Service creates a PaymentIntent with status `"created"` and persists it.
4. System returns a unique **PaymentIntent ID**.

No charge happens yet; we only record the intent and create a reference for the rest of the lifecycle.

---

### 2. Customer pays with card

1. Customer enters card details on the merchant site; merchant sends them to the **Transaction Service** with the PaymentIntent ID.
2. Transaction Service creates a transaction with status `"pending"`.
3. Transaction Service talks to the payment network:
   - Sends authorization request
   - Gets approval/decline
   - Updates transaction status
   - Listens for callbacks (settlement, chargebacks, etc.) and updates records
4. Transaction Service updates the PaymentIntent status as the transaction progresses.

---

### 3. Merchant checks payment status

1. Merchant calls `GET /payment-intents/{paymentIntentId}`.
2. API Gateway authenticates and routes to PaymentIntent Service.
3. Service reads current PaymentIntent (status, errors, related transactions) from the database.
4. Service returns a structured response.

---

## Deep Dives

### 1. Security

Two main concerns:

1. **Authentication:** Is the caller (merchant) who they claim to be?
2. **Data protection:** Are we protecting customer payment data from theft or misuse?

#### 1.1 Merchant authentication: API keys + request signing

- **Idea:** Public API key (identify merchant) + private secret (never in client code). For each request, the merchant’s server signs: method, path, body, timestamp, nonce with the secret.
- **Verification:** API Gateway looks up the secret by API key, recomputes the signature (e.g. HMAC-SHA256), checks it matches, validates timestamp window (e.g. 5–15 min), and ensures nonce is not reused (cache/DB).
- **Result:** Authenticity, integrity, and replay protection.

**Example request with signature:**

```json
{
  "method": "POST",
  "path": "/payment-intents/{paymentIntentId}/transactions",
  "body": {},
  "headers": {
    "Authorization": "Bearer pk_live_51NzQRtGswQnXYZ8o",
    "X-Request-Timestamp": "2023-10-15T14:22:31Z",
    "X-Request-Nonce": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "X-Signature": "sha256=7f83b1657ff1fc53b92dc18148a1d65dfc2d4b1fa3d677284addd200126d9069"
  }
}
```

#### 1.2 Card data: iframe + encryption

- **Idea:** Our JS SDK runs in an iframe; we manage encryption keypairs. Card data is encrypted with our public key in the browser before leaving the device.
- **Flow:** Encrypted payload goes over HTTPS to our servers; private key (in HSMs) decrypts. So: encrypt at origin, protect in transit, decrypt only in a controlled environment.
- **Benefit:** Even if the merchant site or iframe is compromised, attackers only see ciphertext they cannot decrypt.

---

### 2. Durability and auditability

Requirement: **no transaction data ever lost**, even on failures. This also supports webhooks, reconciliation, and fraud analysis.

#### 2.1 Good: separate audit tables

- Main tables for current state; **append-only audit tables** for every state change.
- Same transaction: update main row + insert audit row (e.g. `payment_id`, `change_type`, `old_status`, `new_status`, `changed_by`, `changed_at`, `metadata`).
- **Limitations:** App code must always write both; audit tables live in the same DB as high-TPS operational data, which can hurt performance and scaling.

**Example:**

```sql
BEGIN TRANSACTION;
  UPDATE payments
  SET status = 'captured', updated_at = NOW()
  WHERE payment_id = 'pay_123';

  INSERT INTO payment_audit_log (
    payment_id, change_type, old_status, new_status,
    changed_by, changed_at, metadata
  ) VALUES (
    'pay_123', 'status_change', 'authorized', 'captured',
    'payment_service', NOW(),
    '{"amount": 2500, "auth_code": "ABC123"}'
  );
COMMIT;
```

#### 2.2 Great: database + CDC + event stream + S3

- **Operational DB:** Serves merchant APIs; no audit tables in the hot path.
- **CDC:** Reads DB WAL/oplog and emits every committed change as an event (at DB level, so nothing is missed by app oversight).
- **Event stream (e.g. Kafka):** Append-only log of all state changes, keyed by `payment_intent_id`, with before/after state.
- **Consumers:** Audit, analytics, reconciliation, webhooks—each builds its own view without loading the operational DB.
- **Durability:** Kafka replication (e.g. 3x), retention (e.g. 7–30 days), then archive to S3 for long-term audit and compliance.

**Example CDC events:**

```json
{
  "op": "insert",
  "source": "payment_intents_db",
  "table": "payment_intents",
  "ts_ms": 1681234567890,
  "before": null,
  "after": {
    "payment_intent_id": "pi_123",
    "merchant_id": "merch_456",
    "amount": 2500,
    "status": "created"
  }
}
```

```json
{
  "op": "update",
  "source": "payment_intents_db",
  "table": "payment_intents",
  "ts_ms": 1681234568901,
  "before": { "payment_intent_id": "pi_123", "status": "created" },
  "after": { "payment_intent_id": "pi_123", "status": "authorized" }
}
```

**CDC as single point of failure?** In practice, run multiple CDC instances (e.g. to different Kafka clusters), monitor lag, and have procedures to replay from DB logs or application-level fallbacks that write critical events to Kafka if CDC is delayed.

---

### 3. Transaction safety and financial integrity

External payment networks are asynchronous; we must stay correct under timeouts and retries.

#### Event-driven safety with reconciliation

- **Record the attempt:** Before calling the network, write an attempt (network, reference ID, intent) to the DB → CDC event.
- **Call the network** with a timeout.
- **On response:**
  - **Success:** Mark attempt `succeeded` → CDC event.
  - **Timeout:** Mark attempt `timeout` → CDC event for reconciliation.
  - **Failure:** Mark attempt `failed` with reason.
- **Reconciliation service:** Consumes timeout (and other) events; queries the network by reference ID and/or processes batch settlement/reconciliation files. Updates our state from the **authoritative** network data.

Result: full audit trail and correct final state even when the initial call times out.

---

### 4. Scalability (10,000+ TPS)

#### Kafka

- Cluster can do high throughput; per partition ~5k–10k msg/s. For ~10k TPS, use **multiple partitions** (e.g. 3–5).
- **Partition by `payment_intent_id`** so all events for one PaymentIntent are ordered, while different intents are processed in parallel.

#### Database

- ~10k writes/s is at the upper end for a single PostgreSQL with replicas and indexing. **Shard by `merchant_id`** to scale writes and isolate load.

---

## Bonus: Webhooks

Merchants provide:

- **Callback URL** — we POST status updates here.
- **Subscribed events** — which event types they want.

**Flow:**

1. **DB updates:** PaymentIntent and Transaction services update the operational DB as payments move through their lifecycle.
2. **CDC:** Changes are published to the event stream (e.g. Kafka).
3. **Webhook service:** Consumes the same stream (with reconciliation, etc.). For each event, checks if the merchant has a webhook for that type; if so, builds payload, signs it with a shared secret, and POSTs to the callback URL.
4. **Delivery:** Record each attempt; on failure, retry with exponential backoff (e.g. 5s, 25s, 125s, … up to ~1 hour).
5. **Merchant:** Expose HTTPS endpoint, verify signature with the shared secret, process payload, return 2xx to avoid retries.

**Example webhook payload:**

```json
{
  "id": "evt_1JklMnOpQrStUv",
  "type": "payment.succeeded",
  "created": 1633031234,
  "data": {
    "object": {
      "id": "pay_1AbCdEfGhIjKlM",
      "amountInCents": 2499,
      "currency": "usd",
      "status": "succeeded",
      "created": 1633031200
    }
  }
}
```
