# SALESTORM – Architecture Decision Records

> Deliverable 18 · Format: Context → Options → Decision → Consequences · Status: all **Accepted** (5 Oct 2026)

---

## ADR-001: Microservices with a dedicated Inventory & Reservation service

**Context.** Flash-sale load is wildly uneven: browse and the reserve path spike 50×, while order/fulfilment volume is tiny (~100 orders). Payment must be isolated for security (PCI scope).

**Options.** (a) Modular monolith · (b) Microservices by business capability.

**Decision.** Microservices: Catalog, Sale, Cart, **Inventory & Reservation**, Checkout, Payment, Order, Fulfilment, Notification — each owning its data.

**Consequences.** ✅ Scale the hot path (Inventory, Gateway) independently; failure isolation; PCI scope limited to Payment. ❌ Distributed transactions → need Saga/outbox; operational overhead (tracing, deployment). *Mitigation:* a small team could start as a modular monolith with the same boundaries.

**Why this boundary?** Inventory is the only place where contention on a shared resource must be serialised; giving it one owner means **one place** to reason about consistency.

---

## ADR-002: PostgreSQL (SQL) as source of truth; Redis for speed

**Context.** We need *guarantees* (no oversell, no duplicate payment), not probabilities.

| Option | Pros | Cons |
|---|---|---|
| PostgreSQL | ACID, row locks, CHECK/UNIQUE constraints, mature HA | Vertical write limits per row |
| NoSQL (DynamoDB/Cassandra/Mongo) | Horizontal scale | Weaker multi-item transactions; conditional writes possible (DynamoDB) but constraints across items are application-enforced |
| Redis only | Fastest | Not durable enough as money/stock truth |

**Decision.** PostgreSQL for Inventory, Order, Payment; Redis for gate, cache, rate limiting, cart; OpenSearch for search.

**Consequences.** ✅ Constraints make violations *impossible*. ❌ Single-row hotspot → solved by ADR-003 gate. DB scaling via replicas/partitioning later.

---

## ADR-003: Concurrency control = Redis Lua gate + atomic conditional UPDATE

**Context.** 10,000 concurrent requests for 100 units on one row.

**Options.** Pessimistic `SELECT FOR UPDATE` · Optimistic version check · Atomic conditional UPDATE · Redis atomic counter · Single-consumer queue per SKU · Distributed lock.

**Decision.** Two-layer: **Redis Lua script** (atomic check + decrement + dedupe) rejects ~99% in memory; winners execute **`UPDATE … WHERE available_quantity >= qty`** with **`CHECK (available_quantity >= 0)`** in one ACID transaction with the reservation insert.

**Consequences.** ✅ Sub-ms rejections; DB touched ≈ 100 times; hard guarantee in DB. ❌ Two stores can drift → compensation + `StockReconciler`; drift only ever toward underselling. **Rejected:** optimistic locking (retry storm under contention), pessimistic alone (lock convoy), distributed locks (slow, complex). **Fallback/scale path:** queue-per-SKU or stock bucketing at 50×.

---

## ADR-004: Temporary reservation with 10-minute TTL

**Context.** Units must be held while the customer pays, but abandoned holds must return to stock.

**Decision.** Reservation row with `expires_at`; `ReservationExpiryJob` (every 5 s, `FOR UPDATE SKIP LOCKED`) releases expired holds; guarded transitions; payment-in-flight holds are resolved via the reconciler before release.

**Consequences.** ✅ No permanent loss of stock; fair re-offering. ❌ Late-payment-after-expiry edge case → compensating refund. TTL is a business trade-off (shorter = more sales churn, longer = more blocked stock).

---

## ADR-005: Idempotency keys on all business-creating APIs + idempotent consumers

**Decision.** `Idempotency-Key` header required on reserve/checkout/payment/cancel; UNIQUE DB constraints; gateway receives unique merchant reference; Kafka consumers dedupe on `event_id`.

**Consequences.** ✅ Safe retries everywhere (client, gateway, Kafka). ❌ Key storage & TTL management; clients must generate keys correctly.

---

## ADR-006: Synchronous reservation & payment initiation; asynchronous post-payment flow

| Option | Pros | Cons |
|---|---|---|
| Fully synchronous chain | Simple, immediate | One failure (e.g. Order down) fails/loses the purchase |
| Fully asynchronous (queue everything) | Most resilient | User doesn't know if they got the item; complex UX |
| **Hybrid** | Instant yes/no on stock; resilient afterwards | Eventual consistency after payment |

**Decision.** Hybrid. Sync where the user needs a decision (reserve, checkout, payment start); async (Kafka) after payment outcome.

**Consequences.** ✅ Order Service outage of 30 s loses nothing. ❌ Order shows "confirming…" briefly; need tracing across async hops.

---

## ADR-007: Transactional Outbox + Kafka for events

**Context.** Writing to DB and publishing to Kafka separately risks "paid but no event".

**Options.** Dual write · 2PC/XA · **Outbox** · Event sourcing.

**Decision.** Outbox table in each service DB, written in the same transaction; relay (Debezium CDC or poller) publishes to Kafka (RF=3, `acks=all`).

**Consequences.** ✅ Atomic state+event; replayable. ❌ Extra component; ~100 ms event latency; outbox cleanup job.

---

## ADR-008: Saga with compensation instead of distributed transactions

**Decision.** Checkout orchestrates the synchronous steps; events choreograph the rest. Compensations: release reservation (payment failed/expired), refund (paid but can't fulfil), cancel order.

**Consequences.** ✅ No 2PC, services stay available. ❌ Eventual consistency; compensations must be idempotent and tested.

---

## ADR-009: Payment resilience – circuit breaker, no blind retries, reconciliation

**Decision.** Resilience4j circuit breaker + 10 s timeout on charge; on timeout mark `TIMED_OUT` and resolve via status query/webhook; `PaymentReconciler` job; secondary provider through Factory/Adapter.

**Consequences.** ✅ No double charges, gateway outages contained. ❌ Customers may see "processing" for up to minutes in rare cases.

---

## ADR-010: Virtual waiting room + layered rate limiting

**Context.** A 500k req/s spike in seconds; autoscaling too slow; bots.

**Decision.** CDN/WAF → gateway token buckets → waiting room issuing signed admission tokens at a controlled rate (e.g. 20k/s), closes when sold out.

**Consequences.** ✅ Backend sees a smooth, bounded load; fairness; bot resistance. ❌ Extra latency/UX step; another component to operate.

---

## ADR-011: Cache strategy – cache reads, never cache inventory decisions

**Options.** Write-through · Write-back · **Cache-aside** · No cache.

**Decision.** CDN + Redis cache-aside for catalog/sale pages and the approximate stock badge (≤ 1 s stale). Inventory decisions always go through the Lua gate + DB.

**Consequences.** ✅ 95%+ of reads never reach services. ❌ Stock badge may briefly show "3 left" when it's 0 — reserve call returns SOLD_OUT correctly.

---

## ADR-012: Kubernetes, multi-AZ, pre-scaling before sales

**Decision.** Stateless services on managed Kubernetes across 3 AZs; managed PostgreSQL with sync standby; Redis Cluster with replicas; Kafka 3 brokers; scheduled pre-scaling 30 min before sale; warm DR region.

**Consequences.** ✅ AZ failure tolerance, predictable capacity. ❌ Cost of idle pre-scaled capacity during the sale window.

---

### Summary of Major Trade-offs

| Dimension | Our choice | What we gave up |
|---|---|---|
| Consistency vs availability (inventory) | **Consistency** | Brief unavailability during DB failover |
| Sync simplicity vs async resilience | **Hybrid** | Eventual consistency after payment |
| Optimistic vs pessimistic | **Neither alone** – atomic update behind a gate | Two-store reconciliation |
| SQL vs NoSQL | **SQL for truth**, NoSQL/Redis for speed | Harder horizontal write scaling |
| Simplicity vs scalability | Microservices + infra components | Operational complexity |
