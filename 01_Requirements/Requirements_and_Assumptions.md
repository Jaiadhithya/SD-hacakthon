# SALESTORM – Requirements & Assumptions

> Deliverable 1 · Phase I · Owner: whole team (Architect leads)

## 1. Problem Statement

SALESTORM runs limited-stock flash sales. At sale start, **10,000 customers press "Buy Now" for 100 units** of Product X within seconds. The platform must:

- survive the traffic spike without collapsing,
- sell **at most 100 units** (never oversell, never go negative),
- never create duplicate reservations, payments or orders,
- drive every paid purchase to a valid order, even when services fail.

**Business question:** *How do we handle thousands of simultaneous purchase requests for limited inventory without overselling, while keeping payment and order processing reliable?*

## 2. Scope – the Required Pipeline

```
Customer → Product Discovery → Cart → Inventory Check → Inventory Reservation → Checkout
        → Payment → Order → Fulfilment → Shipment → Notification → Delivery Tracking
```

**In scope:** everything in the pipeline above, plus sale/promotion rules, admission control (waiting room), observability, security.
**Out of scope:** seller onboarding, returns/refunds UI, recommendation engine, warehouse robotics, tax engine internals (treated as an external call).

## 3. Functional Requirements

| ID | Requirement | Pipeline stage |
|---|---|---|
| FR-01 | Customers can browse/search products and view sale pages (price, stock badge, countdown). | Discovery |
| FR-02 | Customers can add/remove items in a cart; flash-sale items are limited to **1 unit per customer**. | Cart |
| FR-03 | System shows an *approximate* live stock indicator (may lag by ≤ 1 s). | Inventory Check |
| FR-04 | "Buy Now" creates a **temporary reservation** (hold) for the unit if stock remains; otherwise returns *SOLD OUT* immediately. | Reservation |
| FR-05 | A reservation expires after **10 minutes** if unpaid and the unit returns to available stock. | Reservation |
| FR-06 | Repeated "Buy Now" from the same customer/request returns the **same** reservation (idempotent). | Reservation |
| FR-07 | Checkout validates the reservation, applies sale price/coupon, creates the order in `CREATED → PAYMENT_PENDING`. | Checkout |
| FR-08 | Customer pays via an external payment gateway (card/UPI/wallet); duplicate pay requests never double-charge. | Payment |
| FR-09 | On payment success: reservation → `CONFIRMED → SOLD`, order → `CONFIRMED`. On failure/timeout: reservation → `RELEASED`, order → `CANCELLED`. | Payment / Order |
| FR-10 | Confirmed orders flow to fulfilment → shipment (`PROCESSING → SHIPPED → OUT_FOR_DELIVERY → DELIVERED`). | Fulfilment / Shipment |
| FR-11 | Customers receive notifications (email/SMS/push) on reservation, payment, order and shipment events. | Notification |
| FR-12 | Customers can track order and delivery status. | Tracking |
| FR-13 | Admins can create a sale (product, stock, price, start/end time, per-customer limit). | Sale mgmt |
| FR-14 | Every state change of inventory, reservation, payment and order is **audited**. | Cross-cutting |

## 4. Non-Functional Requirements (measurable)

| Category | Target | Type |
|---|---|---|
| **Inventory correctness** | `sold_quantity ≤ initial_stock` and `available_quantity ≥ 0` at all times — **0 oversells** | **Strict guarantee** |
| **Idempotency** | 0 duplicate reservations / payments / orders for the same idempotency key | **Strict guarantee** |
| **Order completeness** | Every successful payment reaches a valid order state (`CONFIRMED` or refunded) within **5 min** | **Strict guarantee (eventual)** |
| Throughput – normal | ~10,000 req/s | Target |
| Throughput – flash sale | Absorb up to **500,000 req/s** at the edge; **≥ 20,000 reservation attempts/s** at Inventory | Target |
| Latency – reserve API | p99 **< 200 ms** (sold-out response p99 < 50 ms) | Target |
| Latency – browse (cached) | p95 < 100 ms | Target |
| Latency – payment | p99 < 3 s end-to-end (gateway-bound) | Target |
| Availability | Checkout/Reserve/Payment **99.95%**; Browse 99.99% (CDN) | Target |
| Consistency | **Strong** for inventory, payment, order rows; **eventual** (≤ 1 s) for stock display, search, notifications | Design rule |
| Durability / Recovery | **RPO = 0** for payments & orders (sync replica); **RTO < 5 min** for DB failover | Target |
| Reservation release | Expired holds released within **≤ 30 s** of `expires_at` | Target |
| Security | TLS 1.2+, OAuth2/JWT auth, PCI-DSS scope minimised (tokenised card data – we never store PAN) | Strict |
| Observability | 100% of checkout/payment/order requests traced; alert within 1 min on inventory mismatch | Target |

## 5. Assumptions

1. **One sale = one SKU** (Product X) with **100 units**; one warehouse. Design generalises to many SKUs by sharding on `sku_id`.
2. **Per-customer limit = 1 unit** for the flash sale (common practice; also prevents hoarding bots).
3. Customers must be **logged in** before "Buy Now" (no guest checkout during flash sales).
4. Traffic profile: ~70% of requests arrive in the **first 10 seconds** of the sale; most of them are page refreshes and stock polling.
5. Payment success rate ≈ **95%**, failure ≈ **5%**, duplicate client requests ≈ **2%** (from the practical test case).
6. External payment gateway supports **idempotency / merchant reference** and a **status query API** (true for Razorpay, Stripe, PayU, etc.).
7. Payment gateway latency 300 ms–3 s; it may time out or be unavailable.
8. Order Service may be **unavailable for up to 30 s** (practical test case) – design must tolerate this with no lost orders.
9. Cloud deployment (e.g. AWS / GCP) with managed Kubernetes, managed PostgreSQL, Redis and Kafka.
10. Reservation hold time = **10 minutes** (business-configurable per sale).

## 6. Constraints

- Inventory is a **single hot record** per SKU → classic hotspot; cannot be solved by adding app servers alone.
- Payment gateway is **external** – we cannot make it transactional with our DB → requires idempotency + reconciliation.
- Distributed services → **no distributed (2PC) transactions**; consistency via local transactions + events (Saga / Outbox).
- Hackathon scope: design-first; implementation optional.

## 7. Expected Traffic (back-of-envelope)

| Quantity | Estimate |
|---|---|
| Concurrent buyers at T0 | 10,000 (test case) → design headroom for 50× = **500,000** |
| Page/stock polls | 500k users × 1 req/s = **500k req/s** → served by **CDN + Redis cache** |
| "Buy Now" attempts | burst of 10k–500k in ~5 s → **2k–100k req/s** → **waiting room** admits at a controlled rate (e.g. 20k/s) |
| Successful reservations | **100** (hard cap) |
| Payment calls | ~100 + retries (≈ 5 failures may re-release ~5 units to others) |
| Orders written | ~100 → tiny write load; the **contention**, not volume, is the problem |

**Key insight:** The flash sale is a *read-heavy, contention-heavy, write-light* problem. 99% of buyers must be told "sold out" **fast and cheaply**, while the 100 winners must be processed **correctly**.

## 8. Critical Dependencies

| Dependency | Why critical | Mitigation |
|---|---|---|
| Redis (stock gate) | Absorbs contention | Redis Cluster + replica; DB remains source of truth |
| PostgreSQL (inventory, orders, payments) | Source of truth | Sync replica, auto-failover |
| Kafka (events) | Order/notification flow | 3 brokers, RF = 3, `acks=all` |
| Payment gateway | Money movement | Circuit breaker, timeouts, reconciliation, optional 2nd provider |
| Auth / Identity provider | Login before buy | Token validation at gateway (stateless JWT) |

## 9. Strict Guarantees vs Targets (summary)

| Strict guarantees (must never be violated) | Targets (best effort, measured) |
|---|---|
| No overselling (≤ 100 sold) | Latency p99 numbers |
| Inventory never negative | 500k req/s throughput |
| No duplicate reservation / payment / order per idempotency key | 99.95% availability |
| Every captured payment → order or refund | Release of expired holds within 30 s |
| No card data stored (PCI) | Notification delivery within 10 s |
