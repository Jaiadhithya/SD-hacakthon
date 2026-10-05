# SALESTORM – Database Design

> Deliverable 7 companion · Owner: Data & API Engineer · Diagram: `ER_Diagram.png`

## 1. Database Choice & Ownership

| Service | Store | Tables owned | Why |
|---|---|---|---|
| Catalog / Sale | PostgreSQL + OpenSearch (search) + Redis (cache) | PRODUCT, CATEGORY, SALE, DEAL, COUPON | Read-heavy, relational; search index for discovery |
| Cart | Redis (primary) + PostgreSQL snapshot | CART, CART_ITEM | Fast mutable state; loss-tolerant |
| **Inventory** | **PostgreSQL** + Redis gate | INVENTORY, INVENTORY_RESERVATION, OUTBOX_EVENT | **ACID + CHECK constraints** for correctness |
| Order | PostgreSQL | ORDER, ORDER_ITEM, PROCESSED_EVENT, OUTBOX_EVENT | Transactions, state machine |
| Payment | PostgreSQL | PAYMENT, PAYMENT_ATTEMPT, OUTBOX_EVENT | Money → strongest guarantees |
| Fulfilment | PostgreSQL | SHIPMENT | Relational |
| Notification | PostgreSQL / DynamoDB | NOTIFICATION | High write volume, simple access |
| All | Append-only | AUDIT_LOG | Traceability |

**Database-per-service**: no service reads another service's tables; cross-service references (e.g. `order.reservation_id`) are **logical FKs** validated via API/events, physical FKs only inside one service's DB.

## 2. DDL (core tables)

```sql
-- ===== Catalog =====
CREATE TABLE customer (
  customer_id UUID PRIMARY KEY,
  email       VARCHAR(255) NOT NULL UNIQUE,
  phone       VARCHAR(20)  NOT NULL,
  name        VARCHAR(120) NOT NULL,
  status      VARCHAR(10)  NOT NULL DEFAULT 'ACTIVE' CHECK (status IN ('ACTIVE','BLOCKED')),
  created_at  TIMESTAMPTZ  NOT NULL DEFAULT now()
);

CREATE TABLE category (
  category_id UUID PRIMARY KEY,
  parent_id   UUID REFERENCES category(category_id),
  name        VARCHAR(100) NOT NULL
);

CREATE TABLE product (
  product_id  UUID PRIMARY KEY,
  category_id UUID NOT NULL REFERENCES category(category_id),
  sku         VARCHAR(64) NOT NULL UNIQUE,
  name        VARCHAR(200) NOT NULL,
  base_price  NUMERIC(12,2) NOT NULL CHECK (base_price >= 0),
  status      VARCHAR(10) NOT NULL DEFAULT 'ACTIVE'
);
CREATE INDEX ix_product_category ON product(category_id);

CREATE TABLE sale (
  sale_id            UUID PRIMARY KEY,
  product_id         UUID NOT NULL REFERENCES product(product_id),
  sale_price         NUMERIC(12,2) NOT NULL CHECK (sale_price >= 0),
  total_stock        INT NOT NULL CHECK (total_stock > 0),
  per_customer_limit INT NOT NULL DEFAULT 1 CHECK (per_customer_limit > 0),
  hold_minutes       INT NOT NULL DEFAULT 10,
  starts_at          TIMESTAMPTZ NOT NULL,
  ends_at            TIMESTAMPTZ NOT NULL,
  status             VARCHAR(12) NOT NULL DEFAULT 'SCHEDULED',
  CHECK (ends_at > starts_at)
);
CREATE INDEX ix_sale_status_start ON sale(status, starts_at);

CREATE TABLE coupon (
  coupon_id     UUID PRIMARY KEY,
  sale_id       UUID REFERENCES sale(sale_id),
  code          VARCHAR(32) NOT NULL UNIQUE,
  discount_type VARCHAR(8) NOT NULL CHECK (discount_type IN ('PERCENT','FLAT')),
  value         NUMERIC(10,2) NOT NULL CHECK (value > 0),
  max_uses      INT NOT NULL,
  used_count    INT NOT NULL DEFAULT 0,
  CHECK (used_count <= max_uses)
);

-- ===== Inventory (see Concurrency_and_Inventory_Design.md for full detail) =====
-- inventory, inventory_reservation with CHECK (available >= 0) and conservation invariant

-- ===== Order =====
CREATE TABLE "order" (
  order_id        UUID PRIMARY KEY,
  customer_id     UUID NOT NULL,
  reservation_id  UUID NOT NULL UNIQUE,           -- one order per reservation
  status          VARCHAR(20) NOT NULL,
  total_amount    NUMERIC(12,2) NOT NULL CHECK (total_amount >= 0),
  currency        CHAR(3) NOT NULL DEFAULT 'INR',
  idempotency_key VARCHAR(64) NOT NULL UNIQUE,
  version         BIGINT NOT NULL DEFAULT 0,
  created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
  CHECK (status IN ('CREATED','PAYMENT_PENDING','CONFIRMED','PROCESSING',
                    'SHIPPED','OUT_FOR_DELIVERY','DELIVERED','CANCELLED'))
);
CREATE INDEX ix_order_customer ON "order"(customer_id, created_at DESC);
CREATE INDEX ix_order_status ON "order"(status, updated_at);   -- reconciler

CREATE TABLE order_item (
  order_item_id UUID PRIMARY KEY,
  order_id      UUID NOT NULL REFERENCES "order"(order_id) ON DELETE CASCADE,
  product_id    UUID NOT NULL,
  quantity      INT NOT NULL CHECK (quantity > 0),
  unit_price    NUMERIC(12,2) NOT NULL
);

CREATE TABLE processed_event (            -- idempotent consumers
  event_id     UUID PRIMARY KEY,
  consumer     VARCHAR(50) NOT NULL,
  processed_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ===== Payment =====
CREATE TABLE payment (
  payment_id       UUID PRIMARY KEY,
  order_id         UUID NOT NULL,
  idempotency_key  VARCHAR(64) NOT NULL UNIQUE,
  merchant_txn_ref VARCHAR(64) NOT NULL UNIQUE,
  gateway_txn_id   VARCHAR(64) UNIQUE,
  provider         VARCHAR(20) NOT NULL,
  amount           NUMERIC(12,2) NOT NULL CHECK (amount > 0),
  status           VARCHAR(12) NOT NULL,
  attempt_count    INT NOT NULL DEFAULT 0,
  created_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at       TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE UNIQUE INDEX uq_one_success_per_order ON payment(order_id) WHERE status = 'SUCCEEDED';
CREATE INDEX ix_payment_stuck ON payment(status, updated_at) WHERE status IN ('PENDING','TIMED_OUT');

-- ===== Fulfilment / Notification =====
CREATE TABLE shipment (
  shipment_id      UUID PRIMARY KEY,
  order_id         UUID NOT NULL UNIQUE,
  carrier          VARCHAR(30) NOT NULL,
  awb_number       VARCHAR(40) UNIQUE,
  status           VARCHAR(20) NOT NULL,
  address_snapshot JSONB NOT NULL
);

CREATE TABLE notification (
  notification_id UUID PRIMARY KEY,
  customer_id     UUID NOT NULL,
  event_id        UUID NOT NULL UNIQUE,     -- never send the same notification twice
  channel         VARCHAR(5) NOT NULL,
  template        VARCHAR(50) NOT NULL,
  status          VARCHAR(8) NOT NULL
);

-- ===== Cross-cutting =====
CREATE TABLE outbox_event (
  event_id       UUID PRIMARY KEY,
  aggregate_type VARCHAR(30) NOT NULL,
  aggregate_id   UUID NOT NULL,
  event_type     VARCHAR(50) NOT NULL,
  payload        JSONB NOT NULL,
  created_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
  published_at   TIMESTAMPTZ
);
CREATE INDEX ix_outbox_unpublished ON outbox_event(created_at) WHERE published_at IS NULL;

CREATE TABLE audit_log (
  audit_id    BIGSERIAL PRIMARY KEY,
  entity_type VARCHAR(30) NOT NULL,
  entity_id   UUID NOT NULL,
  from_state  VARCHAR(20),
  to_state    VARCHAR(20) NOT NULL,
  actor       VARCHAR(60) NOT NULL,
  trace_id    VARCHAR(40),
  at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX ix_audit_entity ON audit_log(entity_type, entity_id, at);
```

## 3. Design Checklist (brief requirement)

| Requirement | How addressed |
|---|---|
| **Primary keys** | UUID v7 (time-ordered → index-friendly, globally unique, generated by app → safe for idempotent inserts) |
| **Foreign keys** | Inside each service DB; logical references across services |
| **Indexes** | Lookup by customer, status+time for reconcilers/sweepers, partial indexes for hot subsets |
| **Constraints** | `CHECK` non-negative quantities, conservation invariant, enum checks, `UNIQUE` idempotency keys, partial unique (one active reservation per customer, one successful payment per order) |
| **Transaction boundaries** | One aggregate per transaction per service: (inventory row + reservation + outbox), (payment + outbox), (order + processed_event + outbox) |
| **Concurrency control** | Atomic conditional UPDATE (inventory), guarded status UPDATEs, optimistic `version` for orders, `SKIP LOCKED` for job workers |
| **Consistency** | Strong within a service; eventual across services via outbox + Kafka |
| **Audit** | `audit_log` row per state transition with `trace_id`; `created_at/updated_at` on all tables |

## 4. Scaling the Data Layer

- **Read replicas** for catalog/order history; **writes** to primary only.
- **Partitioning:** `order`, `payment`, `audit_log` range-partitioned by month; `outbox_event` purged after publish + 7 days.
- **Sharding (future):** inventory by `sale_id`/`sku_id`, orders by `customer_id` (Citus / Vitess-style).
- **Connection pooling:** PgBouncer (transaction mode) per DB.
- **HA:** primary + synchronous standby (RPO 0) in another AZ; automatic failover ≈ 30 s.

## 5. SQL vs NoSQL Decision (summary – see ADR-002)

Relational (PostgreSQL) for inventory/order/payment because we need **multi-row ACID transactions, CHECK and UNIQUE constraints** — exactly the tools that make overselling and duplicate payments *impossible* rather than *unlikely*. NoSQL/Redis used where we need speed and can tolerate loss (cache, cart, gate, rate limiting).
