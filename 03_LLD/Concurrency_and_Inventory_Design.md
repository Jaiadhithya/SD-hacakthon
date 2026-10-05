# SALESTORM – Concurrency & Inventory Reservation Design

> Stage 3 · Phase II · Owner: Reliability Engineer + Data Engineer
> Diagrams: `02_Sequence_Purchase_Reservation.png`, `05_State_Reservation.png`

## 1. The Guarantee

> **Successful reservations + sales can never exceed 100, and `available_quantity` can never be negative — regardless of how many requests arrive or in what order.**

Enforced by **two layers**:

| Layer | Role | Guarantee level |
|---|---|---|
| **Layer 1 – Redis Lua gate** | *Performance.* Absorbs 10k–500k attempts in memory; rejects ~99% in < 1 ms. | Fast, *advisory* |
| **Layer 2 – PostgreSQL conditional UPDATE + CHECK constraints** | *Correctness.* Single row, single ACID transaction, row-level lock. | **Hard guarantee** |

> **Answer to "Where exactly is inventory consistency guaranteed?"**
> In the Inventory DB, at the statement `UPDATE inventory ... WHERE available_quantity >= :qty` inside the reservation transaction, backed by `CHECK (available_quantity >= 0)`. Redis only decides *who is allowed to try*.

## 2. Comparing Concurrency-Control Approaches

| # | Approach | How it works | Pros | Cons under 10k → 100 | Verdict |
|---|---|---|---|---|---|
| A | **Pessimistic lock** `SELECT … FOR UPDATE` then UPDATE | Lock row, check, decrement | Simple, strictly correct | All 10k transactions queue on **one row lock**; each holds lock for a round-trip → ~2–5k TPS max, connection pool exhaustion, timeouts | ❌ alone |
| B | **Optimistic lock** (`version` column) | Read version, `UPDATE … WHERE version = :v` | No locks held while thinking | Under heavy contention almost every attempt **fails and retries** → retry storm, wasted DB work; livelock risk | ❌ for hot SKU |
| C | **Atomic conditional UPDATE** `SET avail = avail-1 WHERE avail >= 1` | DB checks + decrements in one statement; row lock held only microseconds | Correct, no read-modify-write gap, no retry loop | Still serialises on one row → DB CPU/locks become bottleneck at 10k+ concurrent | ✅ as source of truth |
| D | **Redis atomic counter (Lua script)** | Single-threaded Redis runs check + DECR + dedupe atomically | ~100k+ ops/s on one key, sub-ms | In-memory; must be kept in sync with DB; can be lost on failover | ✅ as fast gate |
| E | **Queue-based serialisation** (Kafka partition per SKU, single consumer) | All buy requests become messages; one consumer processes in order | Perfect ordering, no locks | Async result → user waits/polls; extra latency; consumer is SPOF per SKU | ⚠️ alternative for 50×+ |
| F | Distributed lock (Redlock) | Lock around the decrement | — | Slow, complex, still single-file; correctness debated | ❌ |

**Selected: D + C (Redis Lua fast gate in front of an atomic conditional UPDATE).**
*Why:* D removes 99% of contention cheaply; C guarantees correctness with only ~100–150 transactions ever touching the hot row. Option E is our documented scale-out path (see ADR-003).

## 3. Data Model (concurrency-critical fields)

```sql
CREATE TABLE inventory (
  inventory_id        UUID PRIMARY KEY,
  product_id          UUID NOT NULL REFERENCES product(product_id),
  sale_id             UUID NOT NULL REFERENCES sale(sale_id),
  total_quantity      INT  NOT NULL CHECK (total_quantity > 0),
  available_quantity  INT  NOT NULL CHECK (available_quantity >= 0),
  reserved_quantity   INT  NOT NULL DEFAULT 0 CHECK (reserved_quantity >= 0),
  sold_quantity       INT  NOT NULL DEFAULT 0 CHECK (sold_quantity >= 0),
  version             BIGINT NOT NULL DEFAULT 0,
  updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
  CONSTRAINT uq_inventory_sale UNIQUE (product_id, sale_id),
  CONSTRAINT chk_conservation CHECK (available_quantity + reserved_quantity + sold_quantity = total_quantity)
);

CREATE TABLE inventory_reservation (
  reservation_id   UUID PRIMARY KEY,
  inventory_id     UUID NOT NULL REFERENCES inventory(inventory_id),
  sale_id          UUID NOT NULL,
  customer_id      UUID NOT NULL REFERENCES customer(customer_id),
  quantity         INT  NOT NULL CHECK (quantity > 0),
  status           VARCHAR(20) NOT NULL,
  idempotency_key  VARCHAR(64) NOT NULL UNIQUE,          -- duplicate request protection
  expires_at       TIMESTAMPTZ NOT NULL,
  created_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at       TIMESTAMPTZ NOT NULL DEFAULT now()
);
-- one active reservation per customer per sale (per-customer limit = 1)
CREATE UNIQUE INDEX uq_active_res_per_customer ON inventory_reservation (sale_id, customer_id)
  WHERE status IN ('RESERVED','PAYMENT_PENDING','CONFIRMED','SOLD');
-- expiry sweeper
CREATE INDEX ix_res_expiry ON inventory_reservation (status, expires_at)
  WHERE status IN ('RESERVED','PAYMENT_PENDING');
```

**`chk_conservation`** is our strongest invariant: units cannot appear or vanish.

## 4. Layer 1 – Redis Lua Gate

Keys (hash-tagged so they live on the same Redis Cluster slot):
- `stock:{sale42}` – integer, initialised to 100 at sale start
- `buyers:{sale42}` – set of customer IDs holding/bought a unit
- `idem:{sale42}:<key>` – idempotency record, TTL 15 min

```lua
-- reserve.lua   KEYS[1]=stock  KEYS[2]=buyers  KEYS[3]=idem
-- ARGV[1]=customerId  ARGV[2]=qty  ARGV[3]=ttlSeconds
local existing = redis.call('GET', KEYS[3])
if existing then return {'DUPLICATE', existing} end
if redis.call('SISMEMBER', KEYS[2], ARGV[1]) == 1 then return {'LIMIT_REACHED'} end
local stock = tonumber(redis.call('GET', KEYS[1]) or '0')
if stock < tonumber(ARGV[2]) then return {'SOLD_OUT'} end
redis.call('DECRBY', KEYS[1], ARGV[2])
redis.call('SADD', KEYS[2], ARGV[1])
redis.call('SET', KEYS[3], 'PENDING', 'EX', ARGV[3])
return {'OK'}
```

Redis executes a Lua script **atomically and single-threaded** → no two scripts interleave → the 101st caller always sees `stock = 0`.

## 5. Layer 2 – The ACID Reservation Transaction (transaction boundary)

```sql
BEGIN;  -- READ COMMITTED is sufficient: the UPDATE re-checks the predicate on the locked row
  UPDATE inventory
     SET available_quantity = available_quantity - :qty,
         reserved_quantity  = reserved_quantity  + :qty,
         version = version + 1, updated_at = now()
   WHERE sale_id = :sale AND available_quantity >= :qty;
  -- application: if rowcount = 0 → ROLLBACK, return SOLD_OUT

  INSERT INTO inventory_reservation (reservation_id, inventory_id, sale_id, customer_id, quantity,
                                     status, idempotency_key, expires_at)
  VALUES (:rid, :inv, :sale, :cust, :qty, 'RESERVED', :idemKey, now() + interval '10 minutes');
  -- unique violation on idempotency_key or (sale_id, customer_id) → ROLLBACK, return existing

  INSERT INTO outbox_event (event_id, aggregate_type, aggregate_id, event_type, payload)
  VALUES (:eid, 'Reservation', :rid, 'reservation.created', :json);

  INSERT INTO audit_log (...) VALUES (...);
COMMIT;
```

**Transaction boundary = one Inventory DB, one transaction.** Stock decrement, reservation row, outbox event and audit row commit or roll back together. No distributed transaction is ever needed for reservation.

**If DB fails after Redis said OK:** compensate Redis (`INCRBY stock`, `SREM buyers`, `DEL idem`). If the compensation itself is lost (pod crash), Redis shows *fewer* units than DB → the system **undersells temporarily** (safe direction), and the `StockReconciler` (every 10 s) resets `stock:{sale}` = DB `available_quantity`.

> **Design principle:** every failure mode errs toward **underselling** (recoverable) never **overselling** (unrecoverable).

## 6. Reservation Lifecycle Operations

| Operation | Trigger | DB statement (guarded) | Redis |
|---|---|---|---|
| **Create** | `POST /reservations` | see §5 | Lua gate |
| **Mark payment pending** | Checkout | `UPDATE reservation SET status='PAYMENT_PENDING' WHERE id=? AND status='RESERVED' AND expires_at > now()` | – |
| **Confirm → Sold** | `payment.succeeded` event | `UPDATE reservation SET status='SOLD' WHERE id=? AND status IN ('RESERVED','PAYMENT_PENDING')` + `UPDATE inventory SET reserved=reserved-1, sold=sold+1` (same txn) | – |
| **Release** | `payment.failed`, expiry, cancel | `UPDATE reservation SET status='RELEASED' WHERE id=? AND status IN ('RESERVED','PAYMENT_PENDING')` → if 1 row: `UPDATE inventory SET reserved=reserved-1, available=available+1` | `INCRBY stock`, `SREM buyers` |
| **Expire** | `ReservationExpiryJob` every 5 s | `SELECT … WHERE status IN (…) AND expires_at < now() FOR UPDATE SKIP LOCKED LIMIT 100` → release each | same |

The `WHERE status = <expected>` guard makes every transition **idempotent and race-safe**: if a duplicate `payment.failed` event or the expiry job and a late payment race, only one UPDATE matches; the loser affects 0 rows and does nothing.

**Expiry vs late payment race:** the expiry job does **not** release a `PAYMENT_PENDING` reservation whose payment is `PENDING/TIMED_OUT`; it asks Payment's reconciler first. If payment later succeeds for an already-released unit, the order is **refunded** (compensation) — see Payment & Order design.

## 7. Handling Duplicate Requests

| Duplicate type | Detection | Result |
|---|---|---|
| Same click retried (same `Idempotency-Key`) | Redis `idem:` key → DB `UNIQUE(idempotency_key)` | Returns **the same reservation** (200, replayed) |
| Same customer, new key (two tabs) | Redis `buyers` set → DB partial unique `(sale_id, customer_id)` | 409 `LIMIT_REACHED` |
| Two concurrent identical requests | Lua atomicity; DB unique constraint is the final arbiter | Exactly one reservation |

## 8. Trace: 10,000 Requests for 100 Units

| Step | Count |
|---|---|
| Requests hitting CDN at T0 | 10,000 (+ refreshes) |
| Dropped by WAF/rate limit (bots, > 5 req/s) | ~ several hundred |
| Admitted by waiting room in first second | up to 20,000/s → all 10k within ~0.5 s |
| Lua gate → `OK` | **100** |
| Lua gate → `SOLD_OUT` | ~9,700 |
| Lua gate → `DUPLICATE` / `LIMIT_REACHED` (≈2% duplicates) | ~200 |
| DB transactions on hot row | **≈100** (≈1 ms each → trivial) |
| Successful reservations | **100** |
| Payment failures (5%) → released units | ~5 → re-offered to waiting room → ~5 more reservations |
| **Final SOLD** | **100 (never more)** |

## 9. When Inventory Reaches Zero

1. Lua gate returns `SOLD_OUT` in < 1 ms for every request.
2. Inventory publishes `inventory.sold_out` → Sale Service sets sale status `SOLD_OUT`, CDN page badge purged/updated, **waiting room closes the queue** and tells queued users "Sold out – join waitlist".
3. If units are released later (payment failure/expiry), `inventory.restocked` reopens admission for that many units.

## 10. Consistency Guarantees Summary

| Data | Consistency | Mechanism |
|---|---|---|
| Inventory counts | **Strong (linearizable per SKU row)** | Single-row ACID update + CHECKs |
| Reservation status | Strong | Guarded UPDATEs |
| Redis stock counter | Eventually consistent with DB (≤ 10 s), **never higher than DB** | Compensation + reconciler |
| Displayed stock badge | Eventual (≤ 1 s) | Cached |
