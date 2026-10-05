# SALESTORM – Scalability & Reliability Design

> Deliverable 16 · Phases III & V · Owner: System Architect + Reliability Engineer

## Part A – Scalability

### A1. Capacity Plan

| Tier | Normal (10k req/s) | Flash sale (500k req/s at edge) | Scaling mechanism |
|---|---|---|---|
| CDN / WAF | 8k req/s served at edge | ~475k req/s served at edge (95%+) | Global PoPs, pre-warmed, cache sale page 5–30 s |
| Load balancer | 2k req/s | ~25k req/s dynamic | Managed ALB, pre-warmed via cloud support ticket |
| API Gateway | 6 pods | 60 pods (~500 req/s/pod) | HPA on RPS + **scheduled pre-scale T-30 min** |
| Waiting room | 3 pods | 30 pods; admits 20k/s | Stateless + Redis sorted set |
| Inventory service | 6 pods | 30 pods (CPU-light; mostly Redis calls) | HPA |
| Redis cluster | 3 shards | Hot key ≈ 20k–100k ops/s on one shard (Redis handles ~100k+ ops/s single-thread) | Stock bucketing for 50×+ (below) |
| Inventory DB | – | ~100–200 write txns total on hot row | Not a bottleneck thanks to gate |
| Payment service | 3 pods | 15 pods; ≈ 100–200 payments | Gateway-bound, not CPU-bound |
| Kafka | 3 brokers | Few thousand events | Partitions = 12 per topic |

**Pre-scaling, not auto-scaling, is the primary strategy:** a flash sale spikes in < 5 s; HPA reacts in 30–90 s. We scale up 30 min before `starts_at` (from Sale config) and scale down 30 min after `SOLD_OUT`.

### A2. What Changes at 50× Traffic (10k → 500k concurrent buyers)

| Concern | At 10k | At 500k (50×) – change |
|---|---|---|
| Edge | CDN + WAF | Same, plus **static "sold out" page flip** at CDN as soon as `inventory.sold_out` fires; bot challenge (JS / CAPTCHA by CDN vendor) for suspicious traffic |
| Admission | Waiting room optional | Waiting room **mandatory**; randomised admission among those who arrived in first N seconds (fairness, defeats "fastest bot wins") |
| Hot key | One `stock:{sale}` key | **Stock bucketing**: split 100 units into 10 keys × 10 units on different Redis shards; requests hash to a bucket, on empty bucket try 1–2 others. Spreads load 10× |
| Gate tier | Redis | Optionally **local in-pod token pre-allocation** (each Inventory pod leases e.g. 5 units) → zero network hop for rejections |
| DB | Single primary | Unchanged for writes (still ≤ 100 winners); read replicas for order history |
| Alternative | – | Queue-based serialisation per SKU (Kafka partition, single consumer) if we prefer strict FIFO fairness over instant response |
| Rate limits | 5 req/s/user | Tighter per-IP + per-device fingerprint limits |

### A3. Bottlenecks Ranked

1. **Hot inventory key/row** → Redis gate + bucketing (DB sees only winners).
2. **Connection storms to DB** → PgBouncer; admission caps concurrency.
3. **Payment gateway rate limits** → only ~100 payments; circuit breaker; secondary provider.
4. **Gateway / LB cold start** → pre-warm.
5. **Kafka consumer lag** → partitions & consumer replicas; lag alerts.

### A4. Caching Strategy

| Data | Cache | TTL / invalidation |
|---|---|---|
| Static assets, product images | CDN | Long TTL, versioned URLs |
| Sale page HTML/JSON | CDN | 5–30 s; purge on SOLD_OUT |
| Product details | Redis cache-aside | 5 min; evict on update event |
| Stock badge | Redis (written by Inventory) | ≤ 1 s staleness acceptable |
| Inventory truth | **Never cached for decisions** | – |

## Part B – Reliability & Failure Handling

### B1. Failure Mode Matrix

| Failure | Detection | Impact | Recovery | Data loss? |
|---|---|---|---|---|
| Inventory pod crash mid-request | K8s liveness | Request fails | Client retries with **same Idempotency-Key** → replay or completes | No |
| Redis primary fails | Sentinel/cluster health | Gate unavailable ~5–15 s | Replica promoted; `StockReconciler` rebuilds `stock` from DB `available_quantity` before re-opening; during failover requests get 503 (safe) | No (DB is truth) |
| Redis data lost entirely | Reconciler mismatch | Gate empty | Re-seed from DB; never over-allocate because DB still enforces | No |
| **Inventory DB primary fails** | Health checks | Reservations pause ~30 s | Auto-failover to **sync standby** (RPO 0); app retries; waiting room holds users | No |
| Order DB fails | Health checks | Order writes pause | Failover; payment events wait in Kafka | No |
| **Payment gateway down / slow** | Circuit breaker metrics | Can't charge | Breaker OPEN → fail fast 503; reservations held to expiry; switch to secondary provider; reconciler resolves in-flight payments | No |
| Payment timeout (unknown result) | Timeout | Customer unsure | `TIMED_OUT` → status query + webhook → resolve; never double charge | No |
| **Payment succeeds, Order Service down 30 s** | Consumer lag alert | Order confirmation delayed | Kafka redelivery + idempotent consumer + OrderReconciler | No |
| Kafka broker fails | Broker health | None (RF=3, min.insync=2) | Leader election | No |
| Outbox relay down | Unpublished-rows age alert | Events delayed | Relay restarts, resumes from DB | No |
| Poison message | Retry count | One message stuck | DLQ + alert + replay | No |
| Notification provider down | Error rate | Emails delayed | Retry, DLQ, alternate provider | No |
| AZ outage | Cloud health | 1/3 capacity lost | Multi-AZ pods & DBs; LB routes away | No |
| Region outage | Global health checks | Platform down | DR region warm standby; RTO ~30–60 min, RPO seconds (async cross-region) | ≤ seconds |

### B2. Recovery When the Database Fails (jury question)

1. Primary in AZ-a dies → managed PostgreSQL promotes **synchronous standby** in AZ-b (~30 s). Because replication is synchronous, every committed reservation/payment/order exists on the standby (**RPO = 0**).
2. During failover: Inventory returns `503 SERVICE_UNAVAILABLE + Retry-After`; waiting room stops admitting; clients retry with same idempotency key.
3. After failover: connection pools reconnect (PgBouncer); `StockReconciler` re-syncs Redis `stock` from DB; waiting room resumes.
4. In-flight transactions not committed are rolled back → their Redis decrements are corrected by reconciler (undersell, never oversell).

### B3. Recovery When the Payment Gateway Fails (jury question)

1. Timeouts / 5xx rise → circuit breaker OPENS after 50% failures over 20 calls.
2. New payment attempts: **fail fast** (`503 PAYMENT_UNAVAILABLE`) or **route to secondary provider** via `PaymentGatewayFactory`.
3. In-flight payments (`TIMED_OUT/PENDING`): reconciler polls gateway status once it recovers; webhooks also arrive late and are processed idempotently.
4. Reservations remain held until `expires_at` (optionally **extended** by ops for this sale if the outage is gateway-side), so customers don't lose their unit due to our provider's outage.
5. HALF_OPEN after 30 s → 5 trial calls → CLOSED if healthy.

### B4. Timeouts, Retries & Idempotency Policy (summary)

- Every remote call has a **timeout**; every retry carries the **same idempotency key**; only **idempotent operations** are retried automatically.
- Exponential backoff with jitter; max attempts bounded; then DLQ or user-visible error.
- Kafka consumers: at-least-once + `processed_event` dedupe.

### B5. Stress Scenario Walkthroughs (Phase V)

| # | Scenario | What happens |
|---|---|---|
| 1 | **One successful purchase** | Gateway auth → waiting room admits → Lua OK → DB reserve (txn) → 201 → checkout → order PAYMENT_PENDING → payment SUCCEEDED + outbox → Kafka → Inventory SOLD, Order CONFIRMED → Fulfilment PROCESSING → Notification. |
| 2 | **Failed payment** | Gateway declines → payment FAILED + outbox → Kafka → Inventory: guarded release (reserved-1, available+1), Redis `INCRBY`, `SREM` → `inventory.restocked` → waiting room admits next buyer → Order CANCELLED → customer notified. |
| 3 | **Duplicate Buy** | Same key → Redis `idem` hit → 200 same reservation. Different key, same user → `buyers` set → 409 LIMIT_REACHED. Concurrent race → DB unique constraint picks one. |
| 4 | **Payment success + Order Service down 30 s** | Event durable in Kafka; offset uncommitted; consumer lag alert; service recovers → redelivered → idempotent confirm. Reconciler as safety net. |
| 5 | **Inventory reaches zero** | Lua returns SOLD_OUT in < 1 ms; `inventory.sold_out` → sale status SOLD_OUT, CDN purge, waiting room closes. |
| 6 | **50× traffic** | See A2: mandatory waiting room, stock bucketing, CDN sold-out flip, tighter limits. Correctness unchanged since DB still sees only winners. |
| 7 | **DB / gateway failure** | See B2 / B3. |

## Part C – Trade-offs Accepted

| Choice | Benefit | Cost |
|---|---|---|
| CP for inventory (consistency over availability) | Never oversell | Short reservation pauses during DB failover |
| AP for browse/stock badge | Always fast | Stock number may be ~1 s stale |
| Async post-payment | Survives downstream failures | "Confirming order…" for a few seconds; eventual consistency to explain |
| Pre-scaling | Ready for the spike | Pay for idle capacity ~1 hour per sale |
| Waiting room | Stability & fairness | Users wait in a queue |
