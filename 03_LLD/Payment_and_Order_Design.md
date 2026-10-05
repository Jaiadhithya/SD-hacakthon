# SALESTORM – Payment & Order Workflow Design

> Stage 4 · Phase III · Owner: Reliability Engineer + LLD Engineer
> Diagrams: `03_Sequence_Payment.png`, `04_Sequence_Order.png`, `06_State_Order.png`

## 1. Workflow Style: Orchestrated Saga + Choreographed Events

- **Checkout Service** orchestrates the *synchronous* part the user waits for: validate reservation → price → create order (`PAYMENT_PENDING`) → initiate payment.
- After the payment outcome is known, the flow is **event-driven** (choreography over Kafka): `payment.succeeded / payment.failed` → Inventory (sell / release) and Order (confirm / cancel) → Fulfilment, Notification.
- No 2-phase commit. Each step is a **local transaction + outbox event**; failures are handled by **retries, idempotent consumers, reconciliation and compensation**.

## 2. Payment Design

### 2.1 Idempotency (no duplicate charges)

| Layer | Mechanism |
|---|---|
| Client → Payment API | `Idempotency-Key` header (= `pay-<orderId>-<attemptNo>`). Stored in `PAYMENT.idempotency_key UNIQUE`. Repeat → return stored response. |
| Payment → Gateway | Unique `merchant_txn_ref` sent as gateway idempotency key / order reference. Gateway dedupes on its side. |
| Retry after timeout | **Always query status first** (`GET /charges?ref=`); never fire a blind second charge. |
| DB safety net | Partial unique index: one `SUCCEEDED` payment per `order_id`. |
| Webhooks | Verified via HMAC signature; processed idempotently by `gateway_txn_id`. |

### 2.2 Payment State Machine

```
INITIATED → PENDING → SUCCEEDED → (REFUNDED)
                   ↘ FAILED
                   ↘ TIMED_OUT → (reconciler) → SUCCEEDED | FAILED
```

### 2.3 Scenarios

| Scenario | Design response |
|---|---|
| **Payment succeeds** | Commit `SUCCEEDED` + outbox `payment.succeeded` atomically → Kafka → Order `CONFIRMED`, Reservation `SOLD`, Notification sent. |
| **Payment fails** | Commit `FAILED` + outbox `payment.failed` → Inventory **releases** unit (DB + Redis) → Order `CANCELLED` → customer notified; customer may retry with a new attempt while reservation is still valid. |
| **Payment times out** | Mark `TIMED_OUT` (outcome unknown). Customer sees "Processing". `PaymentReconciler` queries gateway with backoff (15 s → … → 10 min) and also listens to webhooks. Resolves to SUCCEEDED or FAILED path. Reservation is **not released** while payment is unresolved. |
| **Duplicate payment request** | Same idempotency key → stored result returned; gateway receives the same merchant reference → no second charge. |
| **Payment succeeds, Order Service fails** | Event sits durably in Kafka; consumer offset not committed; redelivered after recovery; idempotent processing (`processed_events`). After 5 retries → DLQ + alert. `OrderReconciler` re-drives orders stuck in `PAYMENT_PENDING` with a `SUCCEEDED` payment. If the order truly cannot be fulfilled → **compensating refund**. |

### 2.4 Timeouts, Retries, Circuit Breaker

| Call | Timeout | Retry policy | Circuit breaker |
|---|---|---|---|
| Checkout → Inventory/Order/Payment (internal) | 500 ms / 1 s / 12 s | 2 retries, exp. backoff + jitter, **only with same idempotency key** | Yes (Resilience4j), 50% failure over 20 calls → open 30 s |
| Payment → Gateway charge | 10 s | **No blind retry**; status query instead | Yes: 50% failures or slow calls > 5 s over 20 calls → OPEN 30 s → HALF_OPEN 5 trial calls |
| Payment → Gateway status query | 3 s | 10 attempts, exp. backoff up to 10 min | Shared breaker |
| Kafka consumers | – | 5 retries (1,2,4,8,16 s) → DLQ | – |
| Notification → providers | 5 s | 3 retries → DLQ | Per provider |

**Circuit OPEN behaviour:** fail fast with `503 PAYMENT_UNAVAILABLE`; reservation stays held until expiry so the customer can retry; if a **secondary provider** is configured, `PaymentGatewayFactory` routes new attempts to it (Strategy + Adapter).

## 3. Order Design

### 3.1 States & Transitions

`CREATED → PAYMENT_PENDING → CONFIRMED → PROCESSING → SHIPPED → OUT_FOR_DELIVERY → DELIVERED`
plus `CANCELLED` from `PAYMENT_PENDING`, `CONFIRMED`, `PROCESSING`.

| From | Event | To | Side effects |
|---|---|---|---|
| – | checkout | CREATED | link reservation |
| CREATED | payment initiated | PAYMENT_PENDING | – |
| PAYMENT_PENDING | payment.succeeded | CONFIRMED | outbox `order.confirmed` |
| PAYMENT_PENDING | payment.failed / reservation.expired | CANCELLED | notify |
| CONFIRMED | fulfilment accepted | PROCESSING | – |
| CONFIRMED / PROCESSING | cancel / stock issue | CANCELLED | **refund + release** (compensation) |
| PROCESSING | courier pickup | SHIPPED | AWB stored |
| SHIPPED | courier webhook | OUT_FOR_DELIVERY | notify |
| OUT_FOR_DELIVERY | delivered webhook | DELIVERED | notify |
| OUT_FOR_DELIVERY | failed attempt | SHIPPED | re-attempt |

Implemented with the **State pattern** (`OrderState` classes); persisted with guarded `UPDATE … WHERE status = :expected AND version = :v`.

### 3.2 Recovery Paths

| Failure | Detection | Recovery |
|---|---|---|
| Order Service down (30 s) | Kafka consumer lag alert | Auto redelivery; idempotent consumer |
| Poison message | 5 failed retries | DLQ + alert; replay after fix |
| Event lost between DB and Kafka | Impossible by design (outbox) | Outbox relay retries unpublished rows |
| Paid but order never confirmed | `OrderReconciler` every 1 min | Re-drive confirm, else refund |
| Order confirmed but reservation already released (late payment after expiry) | Inventory `confirm` affects 0 rows | Try re-reserving a unit; if none → cancel + **refund** + apology notification |

## 4. Practical Test Case Walkthrough

*Stock 100 · 10,000 users · 95% success · 5% failures · 2% duplicates · Order Service down 30 s*

1. 10,000 requests → ~200 duplicates answered from idempotency store → 100 reservations, the rest `SOLD_OUT`.
2. 100 checkouts → 100 orders `PAYMENT_PENDING`.
3. ~95 payments succeed, ~5 fail → 5 units released → waiting room admits next 5 buyers → ~5 more reservations → (95% of those succeed …) → converges to **100 SOLD**.
4. Order Service down for 30 s exactly while `payment.succeeded` events arrive → events accumulate in Kafka (lag ≈ 95 messages) → service recovers → all consumed in < 1 s → 100 orders `CONFIRMED`. **Zero lost, zero duplicate orders.**
5. Duplicate pay clicks (2%) → same payment returned; gateway sees same merchant ref → **0 double charges**.
