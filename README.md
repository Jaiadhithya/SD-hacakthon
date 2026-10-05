# SALESTORM – Flash-Sale System Design (SYSCRAFTERS 2026)

> *"When thousands of customers compete for limited stock, can your architecture keep every transaction correct?"* — **Yes. Here's how.**

## The Design in 30 Seconds

1. **Filter at the edge** – CDN/WAF serves cached pages and blocks bots; API Gateway authenticates and rate-limits; a **Virtual Waiting Room** admits buyers at a controlled rate.
2. **Control contention at exactly one place** – the **Inventory & Reservation Service**:
   - **Redis Lua atomic gate** rejects ~99% of requests in under a millisecond once the 100 units are gone.
   - **PostgreSQL conditional UPDATE + `CHECK (available_quantity >= 0)`** in one ACID transaction is the hard guarantee: overselling is physically impossible.
3. **Idempotency everywhere** – `Idempotency-Key` + unique constraints on reservations, payments and orders; the gateway is called with a unique merchant reference.
4. **Async, recoverable after payment** – Transactional **Outbox → Kafka → idempotent consumers**, with retries, DLQ and **reconciliation jobs**. Order Service down for 30 s ⇒ events wait in Kafka, zero lost orders.
5. **Reservations expire** after 10 min and units return to stock automatically.

## Deliverables Index (all 19 mandatory items)

| # | Deliverable | File(s) |
|---|---|---|
| 1 | Requirements & Assumptions | `01_Requirements/Requirements_and_Assumptions.md` |
| 2 | System Context Diagram | `02_HLD/System_Context.png` (`01_System_Context.puml`) |
| 3 | HLD Architecture | `02_HLD/HLD_Architecture.png` (`03_HLD_Architecture.puml`) + `02_HLD/HLD_Explanation.md` |
| 4 | Container Diagram | `02_HLD/Container_Diagram.png` (`02_Container_Diagram.puml`) |
| 5 | Component Diagram | `02_HLD/Component_Diagram.png` (`05_Component_Diagram.puml`) |
| 6 | Deployment Diagram | `02_HLD/Deployment_Diagram.png` (`04_Deployment_Diagram.puml`) |
| 7 | Database / ER Diagram | `04_Database/ER_Diagram.png` + `04_Database/Database_Design.md` (DDL, keys, indexes, constraints) |
| 8 | Class Diagram | `03_LLD/Class_Diagram.png` (`01_Class_Diagram.puml`) |
| 9 | Purchase / Reservation Sequence | `03_LLD/Sequence_Purchase_Reservation.png` |
| 10 | Payment Sequence | `03_LLD/Sequence_Payment.png` |
| 11 | Order Sequence (incl. recovery) | `03_LLD/Sequence_Order.png` |
| 12 | Order / Reservation State Diagrams | `03_LLD/State_Reservation.png`, `03_LLD/State_Order.png` |
| 13 | SOLID Mapping | `06_SOLID/SOLID_Mapping.md` |
| 14 | Design Pattern Mapping | `07_Design_Patterns/Design_Pattern_Mapping.md` |
| 15 | API Specification | `05_API/API_Specification.md` + `05_API/openapi.yaml` |
| 16 | Scalability & Reliability Design | `08_Scalability_Reliability/Scalability_and_Reliability.md` |
| 17 | Security & Observability Design | `09_Security_Observability/Security_and_Observability.md` |
| 18 | Architecture Decision Records | `10_ADR/Architecture_Decision_Records.md` (12 ADRs) |
| 19 | Final Presentation | `12_Presentation/SALESTORM_Pitch.pptx` + `Pitch_Script_and_Jury_QA.md` |

Supporting design docs (Stages 3 & 4): `03_LLD/Concurrency_and_Inventory_Design.md`, `03_LLD/Payment_and_Order_Design.md`.

## Where to Find Answers to the Practical Checks

| Check | Where |
|---|---|
| 10,000 concurrent attempts request path | HLD diagram, `Concurrency_and_Inventory_Design.md` §8, deck slide 5 |
| 100 units protected from overselling | `Concurrency_and_Inventory_Design.md` §1–5, purchase sequence, deck slide 6 |
| Reservation expiry & release | Reservation state diagram, `Concurrency_and_Inventory_Design.md` §6 |
| Payment failure & recovery | Payment sequence, `Payment_and_Order_Design.md` §2.3 |
| Payment success + Order Service failure | Order sequence, `Payment_and_Order_Design.md` §3.2 |
| Duplicate request handling | Purchase & payment sequences, `Concurrency_and_Inventory_Design.md` §7 |
| 50× traffic | `Scalability_and_Reliability.md` A2 |
| DB / payment gateway failure | `Scalability_and_Reliability.md` B2, B3 |

## Rendering Diagrams

All diagrams are PlantUML (`.puml`) sources; PNGs are already rendered. To regenerate (requires Java), run from this folder:

```bash
java -DPLANTUML_LIMIT_SIZE=8192 -jar ../tools/plantuml.jar -tpng -charset UTF-8 "**/*.puml"
```

## Team

| Role | Ownership |
|---|---|
| Student 1 – System Architect | HLD, architecture, scalability, deployment |
| Student 2 – LLD & Design Engineer | LLD, SOLID, UML, design patterns |
| Student 3 – Data & API Engineer | Database, APIs, events, integration |
| Student 4 – Reliability Engineer | Concurrency, failure handling, security, recovery |
