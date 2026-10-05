# SALESTORM – Design Pattern Mapping

> Deliverable 14 · Owner: LLD & Design Engineer · Reference: `03_LLD/Class_Diagram.png`, `02_HLD/Component_Diagram.png`

## 1. Object-Oriented (GoF) Patterns

| Pattern | Where (classes) | Problem it solves | Trade-off introduced |
|---|---|---|---|
| **Strategy** | `PricingStrategy` → `FlashSalePricing`, `CouponPricing`, `RegularPricing`; payment-method routing; courier selection by pincode | Pricing rules change every campaign; avoid `if/else` chains in checkout | More classes; strategy selection logic must be configured & tested |
| **Factory** | `PaymentGatewayFactory.forProvider()` | Create correct gateway adapter (+ wrap in circuit breaker) per provider/config without callers knowing concrete classes | One more indirection; factory must be updated when adding providers |
| **State** | `OrderState` → `CreatedState`, `PaymentPendingState`, `ConfirmedState`, …, `CancelledState`; same idea for `Reservation` | Enforce **legal transitions only**; avoid scattered `switch(status)` and illegal jumps (e.g. DELIVERED → PAYMENT_PENDING) | Many small classes; state must still be persisted as enum + guarded UPDATE |
| **Observer** | `OrderEventListener` → `NotificationListener`, `FulfilmentListener` (in-process); Kafka consumers (distributed) | Many parties react to order events; producer shouldn't know them | Harder to follow control flow; need tracing and idempotent listeners |
| **Adapter** | `RazorpayAdapter`, `StripeAdapter` (→ `PaymentGateway`); `DelhiveryAdapter`, `ShiprocketAdapter` (→ `DeliveryPartner`) | External APIs have different formats/errors; isolate them behind our interface | Lowest-common-denominator interface may hide provider-specific features |
| **Facade** | `CheckoutFacade` | Client would otherwise call Inventory, Pricing, Order, Payment separately and handle partial failures | Facade can become a "god service" – kept thin, orchestration only |
| **Repository** | `InventoryRepository`, `ReservationRepository`, `PaymentRepository`, `OrderRepository` | Decouple domain logic from SQL/ORM; enable in-memory fakes in tests | Hides SQL – critical queries (conditional UPDATE) must remain explicit, not ORM-generated |
| **Decorator** | `CircuitBreakerGateway` wraps any `PaymentGateway` | Add timeout/retry/circuit-breaking without changing adapters | Stacking decorators can obscure behaviour; configured centrally |
| **Template Method** *(optional)* | `AbstractScheduledJob` for `ReservationExpiryJob`, `PaymentReconciler`, `OrderReconciler` (lock → fetch batch → process → metrics) | Shared job skeleton with leader election & batching | Inheritance coupling |

## 2. Resilience & Distributed-System Patterns

| Pattern | Where | Problem solved | Trade-off |
|---|---|---|---|
| **Circuit Breaker** | Payment → Gateway; Checkout → internal services; Notification → providers | Stop hammering a failing dependency, fail fast, give it time to recover; prevent cascading failure / thread exhaustion | Some requests rejected while OPEN even if dependency recovered; thresholds need tuning |
| **Retry with exponential backoff + jitter** | Internal calls, Kafka consumers, reconcilers | Transient failures | Can amplify load (retry storm) → bounded retries + only idempotent operations |
| **Timeout** | Every remote call | Hung calls tie up threads | Too short → false failures (payment uses status query to handle that) |
| **Idempotency Key / Idempotent Receiver** | Reservation, checkout, payment APIs; all Kafka consumers (`processed_event`) | Duplicate requests/messages must not duplicate business effects | Storage for keys; TTL management |
| **Transactional Outbox** | Inventory, Payment, Order DBs → Kafka | Avoid dual-write (DB committed but event lost, or vice versa) | Extra table + relay process; slight event latency (~100 ms) |
| **Saga (orchestration + choreography)** | Checkout orchestrates sync steps; events choreograph post-payment | Long-running business transaction across services without 2PC | Eventual consistency; need compensating actions |
| **Compensating Transaction** | Release reservation on payment failure; refund when paid order can't be fulfilled | Undo a completed step in a distributed flow | Business must accept "undo" semantics (refund, apology) |
| **Dead Letter Queue** | `*.dlq` topics | Poison messages shouldn't block partitions | Requires ops tooling & replay process |
| **Bulkhead** | Separate thread pools / pods for reserve path vs browse; separate DB per service | Failure or overload in one area doesn't sink others | Lower resource utilisation |
| **Rate Limiter (Token Bucket)** | API Gateway (per user / per IP) | Abuse, bots, retry storms | Legitimate bursts may be throttled |
| **Queue-Based Load Leveling** | Virtual waiting room; Kafka | Smooth spikes to a rate the backend can handle | Added wait time for users |
| **Cache-Aside** | Product/sale pages in Redis + CDN | Read scalability | Staleness (≤ 30 s product, ≤ 1 s stock badge) |
| **Database per Service** | Inventory/Order/Payment DBs | Independent scaling & failure isolation | Cross-service queries need events/APIs |
| **Reconciliation (anti-entropy)** | `StockReconciler`, `PaymentReconciler`, `OrderReconciler` | Repair any drift that slipped through | Periodic extra load; must itself be idempotent |

## 3. How Patterns Combine on the Critical Path

```
Customer ─► [Rate Limiter] ─► [Queue-Based Load Leveling: Waiting Room]
        ─► ReservationController ─► [Idempotent Receiver]
        ─► ReservationService ─► StockGate [Strategy-able, Redis Lua]
                              ─► InventoryRepository [Repository] (atomic UPDATE)
                              ─► DomainEventPublisher [Transactional Outbox]
        ─► CheckoutFacade [Facade + Saga orchestrator]
              ─► PricingStrategy [Strategy]
              ─► PaymentService ─► PaymentGatewayFactory [Factory]
                                 ─► CircuitBreakerGateway [Decorator + Circuit Breaker]
                                 ─► RazorpayAdapter [Adapter]
        ─► Kafka [Observer / choreography] ─► OrderService [State] ─► FulfilmentListener ─► DeliveryPartner [Adapter]
                                          ─► failures ─► [Retry] ─► [DLQ] ─► [Reconciliation / Compensation]
```
