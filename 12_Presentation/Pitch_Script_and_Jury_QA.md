# SALESTORM – 5-Minute Pitch Script & Jury Q&A

> Deck: `SALESTORM_Pitch.pptx` (speaker notes included on every slide)

## Pitch Timing

| Time | Slide(s) | Speaker | Key line |
|---|---|---|---|
| 0:00–0:30 | 1–2 Problem | Architect | "Read-heavy, contention-heavy, write-light: 99% must hear *sold out* fast; the 100 winners must be perfect." |
| 0:30–1:00 | 3 Requirements | Architect | "Guarantees are made impossible to violate; targets are measured." |
| 1:00–2:00 | 4 HLD | Architect | "Filter at the edge, serialise in exactly one place, recover asynchronously." |
| 2:00–3:00 | 5–6 (7 backup) Critical design | Reliability Eng. | "Consistency is guaranteed by one SQL statement; Redis only decides who may try." |
| 3:00–3:45 | 8–9 Payment & Order | Data/API Eng. | "After a timeout we never assume failure and never charge twice." |
| 3:45–4:30 | 10 LLD + SOLID + Patterns | LLD Eng. | "Adding PayU is one adapter class and one factory entry." |
| 4:30–5:00 | 11 Scalability & Reliability, 13 close | Architect | "Every failure errs toward underselling, never overselling." |

Slide 12 is the scripted answer to the **final jury question**.

## Jury Questions – Prepared Answers

**Q1. Why is this the correct service boundary?**
Inventory is the only shared, contended resource. Giving it a single owner (Inventory & Reservation Service with its own DB) puts all contention logic in one place we can reason about and scale independently. Payment is separate for PCI isolation and external-failure containment; Order owns the lifecycle state machine. Boundaries follow data ownership and failure isolation.

**Q2. Where exactly is inventory consistency guaranteed?**
In the Inventory DB: `UPDATE inventory SET available = available - 1 … WHERE sale_id = ? AND available >= 1`, inside the same transaction as the reservation insert, backed by `CHECK (available_quantity >= 0)` and the conservation check `available + reserved + sold = total`.

**Q3. What happens if two requests reach the inventory service at the same time?**
Both run the Redis Lua script; Redis executes scripts one at a time, so only one sees stock ≥ 1. If both somehow reached the DB, PostgreSQL row-locks the inventory row; the second UPDATE re-evaluates `available >= 1` after the first commits, matches 0 rows, and gets `409 SOLD_OUT`.

**Q4. Why did you select this concurrency strategy?**
Optimistic locking causes a retry storm under 100:1 contention; pessimistic locking causes a lock convoy and pool exhaustion. An atomic conditional UPDATE is correct with microsecond lock hold, and the Redis gate makes sure only ~100 requests ever reach it. Fast and provably correct (ADR-003).

**Q5. Why is this operation synchronous while another is asynchronous?**
Sync where the user needs an answer to decide (reserve: did I get it? payment start: redirect me). Async where the system needs work done reliably regardless of failures (confirm order, sell stock, ship, notify). Async with outbox + Kafka survives the Order Service outage.

**Q6. How does the design recover from payment success followed by order failure?**
Payment + outbox commit atomically → Kafka holds the event durably → Order consumer offset not committed → redelivery after recovery → idempotent processing via `processed_event` + guarded UPDATE. Poison messages → 5 retries → DLQ + alert. `OrderReconciler` re-drives paid-but-unconfirmed orders; if impossible, compensating refund.

**Q7. Which component is the likely bottleneck and how will it scale?**
The hot inventory key/row for the single SKU. Redis gate absorbs it (100k+ ops/s); at 50× we split stock into 10 buckets across Redis shards and optionally pre-allocate tokens to Inventory pods. The DB row only sees ~100 winners. Secondary bottleneck: payment gateway rate limits → circuit breaker + secondary provider.

**Q8. What if Redis says OK but the DB transaction fails?**
Compensate Redis (`INCRBY`, `SREM`, delete idem key). If compensation is lost, Redis shows *fewer* units than the DB (undersell, safe); `StockReconciler` resyncs from the DB every 10 s.

**Q9. What if a payment succeeds after the reservation expired?**
Expiry job doesn't release reservations whose payment is still unresolved. If it truly happened, Inventory tries to re-reserve; if no stock, the order is cancelled and refunded automatically (compensation) and the customer is notified.

**Q10. What do you sacrifice?**
Availability of reservations during a DB failover (~30 s), a few seconds of "confirming order", operational complexity of microservices + Kafka, and the cost of pre-scaled capacity — in exchange for zero overselling and zero lost paid orders.

**Q11. SQL or NoSQL – why?**
SQL for inventory/order/payment: multi-row ACID and CHECK/UNIQUE constraints make violations impossible. Redis/NoSQL for cache, gate, cart and rate limits where speed matters and data is rebuildable (ADR-002).

**Q12. What happens when inventory reaches zero?**
Lua returns SOLD_OUT in < 1 ms; `inventory.sold_out` event → sale status SOLD_OUT, CDN purges the sale page, waiting room closes. Released units (failures/expiry) emit `inventory.restocked` and re-open admission for exactly that many buyers.

## Who Owns What (everyone must be able to answer all of the above)

| Member | Primary deliverables |
|---|---|
| System Architect | 01 Requirements, 02 HLD (context, container, HLD, component, deployment), 08 Scalability |
| LLD & Design Engineer | 03 Class, sequences, states; 06 SOLID; 07 Patterns |
| Data & API Engineer | 04 ER + DB design; 05 API spec + events |
| Reliability Engineer | 03 Concurrency + Payment/Order design; 08 Reliability; 09 Security & Observability; 10 ADRs |
