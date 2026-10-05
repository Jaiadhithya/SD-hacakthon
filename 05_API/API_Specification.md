# SALESTORM – API & Event Specification

> Deliverable 15 · Owner: Data & API Engineer · Machine-readable spec: `openapi.yaml`

## 1. Conventions

| Aspect | Rule |
|---|---|
| Base URL | `https://api.salestorm.com/v1` |
| Transport | HTTPS only (TLS 1.2+); HSTS |
| Auth | `Authorization: Bearer <JWT>` (OIDC access token, 15 min). Admin APIs need `role=admin` scope. Webhooks: HMAC signature header, no JWT. |
| **Idempotency** | `Idempotency-Key: <uuid>` **required** on all POSTs that create business transactions (`/reservations`, `/checkout`, `/payments`, `/orders/{id}/cancel`). Same key + same body → same response (stored 24 h). Same key + different body → `422 IDEMPOTENCY_KEY_REUSED`. |
| Tracing | `X-Request-ID` (generated at gateway if absent) + W3C `traceparent` propagated everywhere |
| Rate limits | Per user 5 req/s on buy/checkout, 50 req/s per IP; headers `X-RateLimit-Remaining`, `Retry-After` |
| Errors | RFC 7807 Problem JSON: `{ "type", "title", "status", "code", "detail", "traceId" }` |
| Versioning | URI version `/v1`; additive changes only within a version |

### Standard error codes

| HTTP | `code` | Meaning |
|---|---|---|
| 400 | `VALIDATION_ERROR` | Bad input |
| 401 | `UNAUTHENTICATED` | Missing/invalid token |
| 403 | `FORBIDDEN` / `ADMISSION_REQUIRED` | No permission / no waiting-room token |
| 404 | `NOT_FOUND` | Unknown resource |
| 409 | `SOLD_OUT` / `LIMIT_REACHED` / `RESERVATION_EXPIRED` / `INVALID_STATE` | Business conflict |
| 402 | `PAYMENT_FAILED` | Declined |
| 422 | `IDEMPOTENCY_KEY_REUSED` | Key reused with different payload |
| 429 | `RATE_LIMITED` | Too many requests (+ `Retry-After`) |
| 503 | `PAYMENT_UNAVAILABLE` / `SERVICE_UNAVAILABLE` | Circuit open / dependency down (+ `Retry-After`) |

## 2. REST Endpoints

### 2.1 Discovery & Sale

| Method | Endpoint | Auth | Description | Responses |
|---|---|---|---|---|
| GET | `/products?category=&q=&page=` | Public | Search/browse (CDN-cached 30 s) | 200 |
| GET | `/products/{productId}` | Public | Product detail | 200, 404 |
| GET | `/sales/{saleId}` | Public | Sale info: price, start/end, status | 200, 404 |
| GET | `/sales/{saleId}/stock` | Public | Approximate stock `{ "available": 37, "status": "LIVE" }` (cached 1 s) | 200 |
| POST | `/admin/sales` | Admin | Create sale (product, stock, price, window, limit) | 201, 400 |

### 2.2 Cart

| Method | Endpoint | Description | Responses |
|---|---|---|---|
| GET | `/cart` | Current cart | 200 |
| POST | `/cart/items` | `{ productId, quantity }` | 201, 409 `LIMIT_REACHED` |
| DELETE | `/cart/items/{itemId}` | Remove | 204 |

### 2.3 Waiting Room

| Method | Endpoint | Description | Responses |
|---|---|---|---|
| POST | `/sales/{saleId}/queue` | Join queue | 202 `{ ticketId, position, estimatedWaitSec }` |
| GET | `/sales/{saleId}/queue/{ticketId}` | Poll | 200 `{ status: WAITING \| ADMITTED \| SOLD_OUT, admissionToken? }` |

### 2.4 Reservation (critical)

**`POST /sales/{saleId}/reservations`**

Headers: `Authorization`, `Idempotency-Key`, `X-Admission-Token`

```json
// Request
{ "productId": "p-1", "quantity": 1 }

// 201 Created
{ "reservationId": "r-77", "status": "RESERVED", "quantity": 1,
  "expiresAt": "2026-10-05T10:10:00Z", "checkoutUrl": "/v1/checkout" }

// 200 OK (idempotent replay – same body as original 201, header Idempotent-Replayed: true)

// 409 Conflict
{ "type": "https://salestorm.com/errors/sold-out", "title": "Sold out",
  "status": 409, "code": "SOLD_OUT", "traceId": "4bf92f..." }
```

| Method | Endpoint | Description | Responses |
|---|---|---|---|
| GET | `/reservations/{reservationId}` | Status + `expiresAt` | 200, 404 |
| DELETE | `/reservations/{reservationId}` | Customer releases hold | 204, 409 `INVALID_STATE` |

### 2.5 Checkout

**`POST /checkout`** (Idempotency-Key required)

```json
// Request
{ "reservationId": "r-77", "couponCode": "FLASH10",
  "shippingAddressId": "a-3", "paymentMethod": { "type": "UPI" } }

// 201 Created
{ "orderId": "o-55", "orderStatus": "PAYMENT_PENDING",
  "amount": { "value": 8999.00, "currency": "INR" },
  "payment": { "paymentId": "p-1", "status": "PENDING",
               "nextAction": { "type": "REDIRECT", "url": "https://gateway/..."} } }
```
Errors: 409 `RESERVATION_EXPIRED`, 409 `INVALID_STATE`, 503 `PAYMENT_UNAVAILABLE`.

### 2.6 Payment

| Method | Endpoint | Auth | Description | Responses |
|---|---|---|---|---|
| POST | `/payments` | JWT (internal: service token) | `{ orderId, amount, method }` + Idempotency-Key | 201 PENDING, 200 replay, 402, 503 |
| GET | `/payments/{paymentId}` | JWT | Status | 200 |
| POST | `/webhooks/payments/{provider}` | HMAC signature | Gateway callback `payment.captured / failed / refunded` | 200 (always ack after durable store), 401 bad signature |
| POST | `/admin/payments/{paymentId}/refund` | Admin | Manual refund | 202 |

### 2.7 Orders & Tracking

| Method | Endpoint | Description | Responses |
|---|---|---|---|
| GET | `/orders?status=&page=` | Customer's orders | 200 |
| GET | `/orders/{orderId}` | Order detail + status history | 200, 404 |
| POST | `/orders/{orderId}/cancel` | Cancel (if state allows) | 202, 409 `INVALID_STATE` |
| GET | `/orders/{orderId}/tracking` | Shipment timeline | 200 |
| POST | `/webhooks/shipments/{carrier}` | Carrier status webhook (HMAC) | 200 |

## 3. Commands vs Events

| Type | Name | Producer | Consumers | Transport |
|---|---|---|---|---|
| Command (sync) | ReserveStock | API Gateway | Inventory | REST |
| Command (sync) | CreateOrder | Checkout | Order | REST/gRPC |
| Command (sync) | InitiatePayment | Checkout | Payment | REST/gRPC |
| Command (async) | ReleaseReservation | Expiry job / Payment events | Inventory | Kafka |
| Command (async) | SendNotification | any | Notification | Kafka |

## 4. Event Catalogue (Kafka)

| Topic | Event | Owner (producer) | Consumers & responsibility | Key |
|---|---|---|---|---|
| `reservation.events` | `reservation.created` | Inventory | Notification (hold confirmation), Analytics | reservationId |
| | `reservation.released` | Inventory | Sale/Waiting Room (re-open admission), Order (cancel if pending) | reservationId |
| | `reservation.expired` | Inventory | Order → CANCELLED, Notification | reservationId |
| `inventory.events` | `inventory.sold_out` / `inventory.restocked` | Inventory | Sale (status), Waiting Room (close/open), CDN purge | saleId |
| `payment.events` | `payment.succeeded` | Payment | **Order → CONFIRMED**, **Inventory → SOLD**, Notification | orderId |
| | `payment.failed` | Payment | **Inventory → RELEASE**, Order → CANCELLED, Notification | orderId |
| | `payment.refunded` | Payment | Order, Notification | orderId |
| `order.events` | `order.confirmed` | Order | Fulfilment (start), Notification | orderId |
| | `order.cancelled` | Order | Payment (refund if paid), Inventory (release), Notification | orderId |
| `shipment.events` | `shipment.shipped / out_for_delivery / delivered` | Fulfilment | Order (state), Notification | orderId |
| `*.dlq` | failed messages | consumers | Ops tooling / replay | same |

### Event envelope (all events)

```json
{
  "eventId": "e-1f0c...",           // UUID – consumers dedupe on this
  "eventType": "payment.succeeded",
  "eventVersion": 1,
  "occurredAt": "2026-10-05T10:02:11.402Z",
  "producer": "payment-service",
  "aggregateId": "o-55",
  "traceId": "4bf92f3577b34da6a3ce929d0e0e4736",
  "payload": { "paymentId": "p-1", "orderId": "o-55", "reservationId": "r-77",
               "amount": 8999.00, "currency": "INR", "gatewayTxnId": "G-888" }
}
```

### Delivery guarantees

- Producers: **transactional outbox** → at-least-once publish, `acks=all`, idempotent producer enabled.
- Consumers: **at-least-once** consumption + **idempotent handlers** (`processed_event` table / guarded UPDATEs) ⇒ effectively-once business effect.
- Ordering: per aggregate via partition key (`orderId`, `reservationId`).
- Schema: JSON Schema / Avro in a schema registry; backward-compatible evolution.
