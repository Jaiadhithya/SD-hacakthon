# SALESTORM – Security & Observability Design

> Deliverable 17 · Phase V · Owner: Reliability Engineer

## Part A – Security

### A1. Controls by Requirement

| Requirement | Design |
|---|---|
| **Authentication** | OIDC/OAuth2 login via Identity Provider; short-lived JWT access tokens (15 min) + refresh tokens; gateway validates signature via cached JWKS (no IdP call per request). Login required before Buy Now. MFA for admin accounts. |
| **Authorization** | Role/scope based (`customer`, `admin`, `ops`); resource ownership checks in services (customer can only read own orders/reservations); service-to-service via **mTLS** (service mesh) + workload identity. |
| **HTTPS / secure comms** | TLS 1.2+ at CDN and LB; HSTS; mTLS inside cluster; private subnets for services & DBs; egress to gateways via NAT with allow-listed IPs. |
| **Rate limiting & abuse** | CDN/WAF bot management, IP reputation, OWASP rule set; gateway token buckets (per user, per IP, per device); waiting room with signed, single-use, short-TTL **admission tokens** (HMAC, bound to user ID) – cannot be shared or replayed; per-customer purchase limit (1) enforced in Redis **and** DB. |
| **Input validation** | Schema validation at gateway (OpenAPI), bean validation in services, quantity bounds (1..limit), UUID formats; parameterised SQL only (no injection); output encoding; size limits on bodies. |
| **Payment data (PCI-DSS)** | **We never see or store card numbers**: hosted checkout / gateway SDK tokenisation. We store only `gateway_txn_id`, amount, status. Webhooks verified by **HMAC signature + timestamp** (replay protection). Payment service in isolated namespace with restricted network policy. |
| **Secrets management** | Vault / cloud Secrets Manager; dynamic DB credentials; KMS-encrypted; no secrets in code, images or env files; automatic rotation. |
| **Data protection** | Encryption at rest (KMS) for DBs, Redis, Kafka, backups; PII (email, phone, address) minimised and masked in logs. |
| **Audit logging** | Append-only `audit_log` for every inventory/reservation/payment/order transition + admin actions (who, what, when, trace_id); shipped to immutable storage (WORM bucket) for 1 year. |
| **Supply chain / infra** | Signed container images, image scanning, least-privilege IAM, K8s network policies, Pod Security standards. |

### A2. Threats Specific to Flash Sales

| Threat | Mitigation |
|---|---|
| Bots/scalpers buying all stock | WAF bot score, device fingerprinting, login + account age rules, per-customer limit, admission tokens bound to user, randomised admission |
| Replay / duplicate requests | Idempotency keys; signed single-use admission tokens |
| Price tampering | Price computed server-side by `PricingStrategy`; client price ignored |
| Fake payment webhooks | HMAC signature verification + gateway status query before confirming |
| DDoS | CDN absorption, WAF, rate limits, autoscaling |
| Inventory manipulation by insiders | Admin actions audited, 4-eyes approval for stock changes during live sale |

## Part B – Observability

### B1. Key Metrics (Prometheus, RED + business)

| Metric | Type | Labels | Why |
|---|---|---|---|
| `http_requests_total` | counter | service, route, status | Request rate & errors |
| `http_request_duration_seconds` | histogram | service, route | p50/p95/p99 latency |
| `reservation_attempts_total` | counter | sale, result (`OK/SOLD_OUT/DUPLICATE/LIMIT`) | Funnel & contention |
| `reservation_failures_total` | counter | sale, reason (`db_error/gate_error`) | Inventory reservation failures |
| `inventory_available` / `_reserved` / `_sold` | gauge | sale | Live stock |
| **`inventory_invariant_violation`** | gauge | sale | `available+reserved+sold != total` OR `sold > total` OR Redis stock > DB available |
| `payment_attempts_total` | counter | provider, result | Payment failures |
| `payment_latency_seconds` | histogram | provider | Gateway health |
| `circuit_breaker_state` | gauge | dependency | OPEN/HALF_OPEN visibility |
| `orders_by_status` | gauge | status | Lifecycle health |
| `order_conversion_ratio` | gauge | sale | reservations → paid orders |
| `kafka_consumer_lag` | gauge | topic, group | Queue backlog |
| `outbox_unpublished_age_seconds` | gauge | service | Stuck events |
| `dlq_messages_total` | counter | topic | Poison messages |
| `reservations_expired_total` | counter | sale | Abandoned holds |
| `waiting_room_queue_length` | gauge | sale | User experience |

### B2. Structured Logs (JSON → Loki/ELK)

Every critical business event logs one JSON line:

```json
{ "ts":"2026-10-05T10:00:01.234Z", "level":"INFO", "service":"inventory-service",
  "event":"reservation.created", "traceId":"4bf92f35...", "spanId":"00f067aa...",
  "saleId":"s-42", "reservationId":"r-77", "customerIdHash":"c9f1…", "qty":1,
  "availableAfter":37, "latencyMs":4, "idempotencyKey":"k-123" }
```

Logged events: `reservation.created/rejected/released/expired`, `payment.initiated/succeeded/failed/timed_out/reconciled`, `order.state_changed`, `circuit.opened/closed`, `dlq.message`, `admin.stock_changed`. PII hashed or masked; card data never logged.

### B3. Distributed Tracing (OpenTelemetry → Tempo/Jaeger)

- Gateway creates/propagates W3C `traceparent`; every service instruments HTTP, gRPC, JDBC, Redis, Kafka.
- **Trace context is carried inside Kafka event envelopes** (`traceId`) so one trace spans: `POST /reservations → Lua → DB txn → checkout → payment → gateway → outbox → Kafka → order confirm → fulfilment → notification`.
- Sampling: 100% for checkout/payment/order paths; 1–5% for browse; 100% for errors (tail sampling).

### B4. Dashboards

1. **Flash Sale Live** – stock gauges, reservation attempts by result, waiting-room length, sell-through curve, conversion.
2. **Payments** – success/failure/timeout rates by provider, latency, breaker state, reconciler backlog.
3. **Orders & Events** – orders by status, Kafka lag, outbox age, DLQ count.
4. **Platform** – RED metrics per service, pod counts, DB/Redis CPU & connections.

### B5. Alerts (PagerDuty)

| Alert | Condition | Severity |
|---|---|---|
| **Inventory inconsistency** | `inventory_invariant_violation > 0` for 1 min, or `sold > total` (ever) | **P1 – page immediately** |
| Redis vs DB stock drift | Redis stock > DB available for > 30 s | P2 |
| Payment failure spike | failure rate > 15% over 5 min (baseline 5%) | P1 |
| Circuit breaker open | any payment breaker OPEN > 1 min | P1 |
| Queue backlog | `kafka_consumer_lag{group=order}` > 500 or growing for 2 min | P2 |
| Stuck outbox | `outbox_unpublished_age_seconds` > 60 | P2 |
| DLQ | any new DLQ message on payment/order topics | P2 |
| Paid-not-confirmed orders | payments SUCCEEDED with order not CONFIRMED > 5 min | P1 |
| Latency SLO burn | reserve p99 > 200 ms for 5 min | P2 |
| Error rate | 5xx > 2% on checkout | P1 |
