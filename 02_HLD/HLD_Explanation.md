# SALESTORM – High-Level Design Explanation

> Deliverables 2–6 companion notes · Phase I · Owner: System Architect

Diagrams in this folder (sources `.puml`, rendered `.png`):

| File | Deliverable |
|---|---|
| `01_System_Context.puml` | #2 System Context Diagram |
| `03_HLD_Architecture.puml` | #3 HLD Architecture |
| `02_Container_Diagram.puml` | #4 Container Diagram |
| `05_Component_Diagram.puml` *(Phase IV)* | #5 Component Diagram |
| `04_Deployment_Diagram.puml` | #6 Deployment Diagram |

---

## 1. The Core Idea in One Sentence

**Filter the flood at the edge, serialise contention at exactly one place (Inventory), and make everything after the reservation asynchronous, idempotent and recoverable.**

```
500k req/s ──► CDN/WAF ──► Gateway (auth + rate limit) ──► Waiting Room (admit 20k/s)
             (drops bots,   (drops abusive users,          (smooths the spike)
              serves cache)  dup requests)
                                    │
                                    ▼
                        Inventory Service ──► Redis Lua gate  ──► 9,900 get SOLD OUT in < 50 ms
                                    │                 (100 tokens)
                                    ▼
                         PostgreSQL ACID reserve ──► exactly ≤ 100 reservations
                                    │
                                    ▼
                  Checkout → Payment → (Kafka) → Order → Fulfilment → Notification
```

## 2. Why Each Major Component Exists

| Component | Why it exists | What breaks without it |
|---|---|---|
| **CDN + WAF** | Serves product/sale pages & static assets from edge; blocks bots/DDoS. ~95% of requests never reach us. | Origin overwhelmed by page refreshes. |
| **Load Balancer** | TLS termination, spreads load across gateway pods, health-based failover across AZs. | Single entry pod = SPOF. |
| **API Gateway** | Central JWT auth, per-user/IP rate limiting, routing, `X-Request-ID` & `Idempotency-Key` enforcement. | Every service re-implements auth; no abuse protection. |
| **Waiting Room** | Converts an uncontrolled spike into a **steady admitted rate** (e.g. 20k/s) with fair FIFO-ish ordering. Admission token is signed (HMAC) and short-lived. | Inventory & DB see the full spike; latency explodes, timeouts → retries → retry storm. |
| **Product / Catalog** | Read-heavy browsing, search. Cache-aside on Redis + CDN. | — |
| **Sale / Promotion** | Owns sale window, sale price, coupons, per-customer limit (Strategy pattern for pricing). | Pricing logic leaks into checkout. |
| **Cart** | Normal shopping flow. Flash-sale "Buy Now" **bypasses the cart** to shorten the critical path. | — |
| **Inventory & Reservation** | **The single place where contention is controlled.** Atomic reserve, confirm, release; expiry sweeper. | Overselling. |
| **Checkout (Facade/Saga orchestrator)** | One entry point that coordinates reservation → pricing → order → payment. | Client must call 4 services; partial failures uncoordinated. |
| **Payment** | Isolates money movement: idempotency, gateway adapters, circuit breaker, reconciliation. | Duplicate charges, gateway outages cascade. |
| **Order** | Owns order state machine; consumes payment events. | No single source of order truth. |
| **Fulfilment / Shipment** | Courier booking, tracking webhooks. | — |
| **Notification** | Async fan-out to email/SMS/push with retries. | Slow providers block checkout. |
| **Redis Cluster** | In-memory atomic counter (Lua) absorbs ~10k-500k contention attempts at µs latency; also rate-limit buckets & idempotency cache. | All contention hits the DB row lock. |
| **PostgreSQL (per service)** | ACID **source of truth**; `CHECK (available_quantity >= 0)` makes overselling *physically impossible*. | No hard guarantee. |
| **Kafka** | Durable, replayable events decouple payment → order → fulfilment → notification. Survives the Order Service being down for 30 s. | Synchronous chain: one failure loses the order. |
| **Outbox tables** | Atomically "save state + publish event" (no dual-write problem). | Payment saved but event lost (or vice versa). |
| **Observability stack** | Metrics, traces, logs, alerts for inventory mismatch / payment failures / queue lag. | Flying blind during the sale. |

## 3. Synchronous vs Asynchronous

| Interaction | Mode | Why |
|---|---|---|
| Browse / search | Sync (cached) | User waits; read-only. |
| **Reserve (Buy Now)** | **Sync** | Customer needs an immediate yes/no; consistency must be decided *now*. |
| Checkout → Inventory (validate hold) | Sync | Must not charge for an expired hold. |
| Checkout → Order (create `PAYMENT_PENDING`) | Sync | Order ID needed as payment reference. |
| Checkout → Payment → Gateway | Sync with timeout (≤ 10 s) | User is on the payment page; timeout falls back to async reconciliation. |
| Gateway → Payment (webhook) | Async | Final truth of payment status. |
| Payment → Order / Inventory (confirm, release) | **Async (Kafka)** | Must survive Order Service downtime; retried; idempotent. |
| Order → Fulfilment → Shipment | Async | Long-running, external partners. |
| Anything → Notification | Async | Never block a business transaction on email/SMS. |

**Rule of thumb we used:** *Sync where the user needs the answer to make a decision; async where the system needs the work done reliably.*

## 4. Where Requests Are Throttled, Queued or Rejected

| Layer | Mechanism | Rejection response |
|---|---|---|
| CDN/WAF | Bot score, IP reputation, geo/IP rate limits | 403 / challenge |
| API Gateway | Token bucket: 5 req/s per user, 50 req/s per IP; missing JWT | 429 / 401 |
| Waiting Room | Queue; admit fixed rate; queue closed once stock gate shows 0 | 202 "in queue" → later "SOLD OUT" |
| Inventory (Redis gate) | `stock:{sku}` reaches 0 → instant reject | 409 `SOLD_OUT` |
| Inventory (DB) | Conditional update affects 0 rows | 409 `SOLD_OUT` |
| Idempotency | Same key → return original result | 200 (replayed) |

## 5. Bottlenecks and How We Address Them

| Likely bottleneck | Why | Mitigation |
|---|---|---|
| **Hot inventory row / key** | All 10k requests target one SKU | Redis Lua gate rejects 99% in memory; only ≤ 100 + small retry margin reach the DB row; optional **stock bucketing** (split 100 units into 10 keys × 10) for 50× load |
| DB connections | Thundering herd of pods | PgBouncer pooling; waiting-room admission caps concurrency |
| Payment gateway | External, rate-limited, slow | Only ~100 payments; circuit breaker + timeout + reconciliation; second provider via Strategy/Adapter |
| Kafka consumer lag | Burst of events | Partition by `order_id`; scale consumers to partition count |
| Notification provider | Slow / rate-limited | Async queue, retries with backoff, DLQ |

## 6. Horizontal Scaling & State

- All services are **stateless** → scale on Kubernetes HPA (CPU/RPS). Pre-scale 30 minutes before the sale (scheduled scaling) — autoscaling is too slow for a 5-second spike.
- **State lives only in**: Redis (ephemeral, rebuildable), PostgreSQL (truth), Kafka (durable log).
- Sessions = stateless JWT; no sticky sessions needed.
- Sharding key for multi-SKU sales: `sku_id` (Redis hash slot + DB partition).

## 7. Trade-offs We Accept

| We chose | We sacrifice | Why acceptable |
|---|---|---|
| Strong consistency for inventory | Some availability: if the Inventory DB primary is down, reservations pause (≈ 30 s failover) | Overselling is worse than a brief pause |
| Waiting room | Users wait / see queue position | Fairness + system stability beats a crash |
| Async order confirmation | Order shows `PAYMENT_PENDING` for a few seconds | Survives Order Service failure with zero lost orders |
| Redis + DB double gate | Two systems to keep in sync; reconciliation job needed | Redis gives speed, DB gives the guarantee |
| Microservices | Operational complexity, distributed tracing needed | Independent scaling of the hot path (Inventory) from cold paths |
