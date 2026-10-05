# Prompt: SALESTORM 9-Slide Pitch Deck

Copy everything below the line into your slide tool (Gamma, Canva, Beautiful.ai, Copilot, etc.).

---

Create a 9-slide, 16:9 pitch deck for a system design hackathon. The project is SALESTORM: the architecture for an e-commerce flash sale in which 10,000 customers try to buy 100 units at the same moment. The audience is a jury of engineering faculty and industry reviewers. The pitch lasts 5 minutes and is followed by technical questions.

You have full creative freedom over visual style, colour, fonts and layout. The content below defines what each slide must say. The quality constraints at the end define what "good" looks like and what to avoid. Follow both.

====================================================================
PART 1 — SLIDE CONTENT
====================================================================

For each slide you get a takeaway (the single message the slide must land), the content that has to appear, a suggested visual (adapt it freely), and speaker notes. Shorten the on-slide wording if needed, but keep every number and technical name exactly as written.

--------------------------------------------------------------------
Slide 1 — Title and team
--------------------------------------------------------------------
Takeaway: This team designed SALESTORM, a flash-sale architecture that keeps every transaction correct under extreme load.

On slide:
- Project name: SALESTORM
- One-line description: Flash-sale architecture that sells 100 units to 10,000 simultaneous buyers without overselling, double charging or losing an order
- Event: SYSCRAFTERS 2026 Design-First System Design Hackathon
- Team name: [Team Name]
- Team members and roles:
  - [Name 1], System Architect: HLD, scalability, deployment
  - [Name 2], LLD and Design Engineer: UML, SOLID, design patterns
  - [Name 3], Data and API Engineer: database, APIs, events
  - [Name 4], Reliability Engineer: concurrency, failure handling, security

Visual idea: The project name is the focal point. Present the team as a quiet, well-aligned group (for example names set in a clean typographic list or a simple row), not four identical decorated cards.

Speaker notes: We are [Team Name], and we acted as the architecture team for SALESTORM. The business runs limited-stock flash sales where thousands of people press Buy Now in the same second. Our design answers one question: can every transaction stay correct when that happens? Each of us owned one part of the design, but all four of us can explain the whole system.

--------------------------------------------------------------------
Slide 2 — The problem
--------------------------------------------------------------------
Takeaway: When 10,000 buyers compete for 100 units, a normal shop design oversells, double charges or loses paid orders.

On slide:
- The scenario: 10,000 simultaneous Buy Now requests for 100 units of one product
- Design target: up to 500,000 requests per second (50× the test case)
- The three failures we must prevent:
  - Overselling: unit 101 is sold, or stock goes negative
  - Duplicate transactions: a double click creates two reservations, two charges or two orders
  - Lost orders: the payment succeeds but no order exists after a service failure
- Practical test conditions: 95% of payments succeed, 5% fail, 2% of requests are duplicates, and the Order Service is down for 30 seconds

Visual idea: Show the imbalance itself, for example 10,000 against 100 as a visual ratio, with the three failures presented as consequences of that tension rather than three equal boxes.

Speaker notes: The difficulty is contention on a single resource rather than raw traffic volume. About 99 out of every 100 buyers will not get the product, and they need a fast, cheap "sold out" answer. The 100 who do win must be processed perfectly, even when a payment times out or a service crashes. The brief also gives us a hard test: some payments fail, some requests are duplicated, and the Order Service goes down for 30 seconds. The rest of the deck shows how the design handles each of these.

--------------------------------------------------------------------
Slide 3 — Requirements
--------------------------------------------------------------------
Takeaway: Some properties are guarantees the design makes impossible to break; the rest are targets we measure.

On slide:
- Strict guarantees:
  - At most 100 units sold; available stock never below 0
  - No duplicate reservation, payment or order for the same idempotency key
  - Every successful payment ends in a confirmed order or a refund
  - Card data is never stored
- Measured targets:
  - Reserve API p99 under 200 ms; sold-out reply p99 under 50 ms
  - 99.95% availability for reserve, checkout and payment
  - Expired holds released within 30 s
  - RPO 0 for payments and orders; RTO under 5 min
- Assumptions: login required, 1 unit per customer, 10-minute hold

Visual idea: A clear two-sided comparison (guarantees against targets) where the guarantees side carries more visual weight.

Speaker notes: We separated the brief into two kinds of requirement. Guarantees are enforced by the database and the protocol, so they cannot be violated even when something fails. Targets such as latency and availability are measured and alerted on, and we can miss them briefly without corrupting data. This split drives every later decision: whenever speed and correctness conflict, correctness wins.

--------------------------------------------------------------------
Slide 4 — High-level architecture
--------------------------------------------------------------------
Takeaway: Traffic is filtered at the edge, contention is controlled in one service, and everything after payment recovers asynchronously.

On slide (as a diagram, not a list):
- Edge layer: CDN and WAF, load balancer, API Gateway (JWT auth, rate limit of 5 requests per second per user), virtual waiting room (admits 20,000 buyers per second)
- Service layer: Product, Cart, Sale, Inventory and Reservation, Checkout, Payment, Order, Fulfilment, Notification
- Data layer: Redis (stock counter), PostgreSQL (source of truth, one database per service), Kafka (events and dead-letter queues)
- External systems: payment gateway, delivery partner, email/SMS/push providers
- Mark the Inventory and Reservation service as the single point where contention is controlled
- Distinguish synchronous calls (the user waits) from asynchronous Kafka events (retried, idempotent)

Visual idea: A layered architecture diagram. The Inventory and Reservation service is the one visual focal point.

Speaker notes: Requests pass through three filters before they reach a service. The CDN serves cached pages and blocks bots, the gateway checks the login token and rate-limits each user, and the waiting room admits buyers at a steady rate. All services are stateless and run on Kubernetes across three availability zones, pre-scaled 30 minutes before the sale. State lives only in Redis, PostgreSQL and Kafka. We call a service synchronously when the user needs an immediate answer, and use events when the work must survive failures.

--------------------------------------------------------------------
Slide 5 — How 10,000 requests become exactly 100 reservations
--------------------------------------------------------------------
Takeaway: A Redis gate rejects almost every request in under a millisecond, and one PostgreSQL statement guarantees no overselling.

On slide (as a narrowing sequence; numbering allowed here because these are real steps):
1. 10,000 requests arrive; bots and users over the rate limit are dropped at the edge
2. The waiting room admits buyers at 20,000 per second with signed admission tokens
3. Redis Lua gate (atomic, one script at a time): 100 pass, about 9,700 get SOLD_OUT in under 1 ms, about 200 duplicates get their original answer back
4. PostgreSQL transaction for each of the 100 winners:
   `UPDATE inventory SET available = available - 1 WHERE sale_id = :s AND available >= 1`
   backed by `CHECK (available >= 0)`
5. Result: 100 reservations, 0 oversold
- Last unit, two buyers: only one UPDATE matches; the other gets 409 SOLD_OUT

Visual idea: A funnel or narrowing flow from 10,000 to 100, with the SQL statement shown as real code. This is the most important slide; give it the strongest focal point in the deck.

Speaker notes: The exact place where consistency is guaranteed is the conditional UPDATE inside the reservation transaction, backed by the CHECK constraint. Redis only decides who is allowed to try, so the database row is touched about 100 times instead of 10,000. We compared five approaches. Optimistic locking causes a retry storm under this contention, and pessimistic locking makes every request queue on one row lock. If the database fails after Redis said yes, we return the unit to Redis, and a reconciler resyncs Redis from the database every 10 seconds, so every failure leads to underselling and never to overselling.

--------------------------------------------------------------------
Slide 6 — Payment and order reliability
--------------------------------------------------------------------
Takeaway: Every payment outcome has a defined path, and a paid order survives the Order Service being down.

On slide:
- Payment outcomes:
  - Succeeds (95%): order confirmed, unit marked sold
  - Fails (5%): unit released to the next buyer in the queue, order cancelled
  - Times out: status checked with the gateway; never charged a second time
  - Duplicate request (2%): the same idempotency key returns the stored payment
- When the Order Service is down for 30 seconds:
  - The payment and its event are saved in one database transaction (transactional outbox)
  - The event waits in Kafka until the Order Service recovers
  - The redelivered event is processed once, using its event ID
  - Safety net: 5 retries, then a dead-letter queue and an alert; a reconciler confirms the order or refunds it

Visual idea: Pair the four outcomes with a short timeline of the 30-second outage. Avoid four identical boxes; let the outage timeline be the focal point.

Speaker notes: After a timeout we never assume the payment failed, because the gateway may have charged the card. We query the payment status using the same merchant reference, so a second charge is impossible. The transactional outbox fixes the classic dual-write problem: either the payment and its event are both saved, or neither is. Kafka keeps three copies of every event, so the Order Service can be down for 30 seconds without losing anything. A circuit breaker protects the payment gateway: after 50% of calls fail, it stops calling the gateway for 30 seconds.

--------------------------------------------------------------------
Slide 7 — Low-level design
--------------------------------------------------------------------
Takeaway: The critical modules are built from small interfaces, so a new payment provider or pricing rule needs one new class.

On slide:
- SOLID in the actual classes:
  - Single responsibility: ReservationService, ReservationExpiryJob, PaymentReconciler
  - Open/closed: new PricingStrategy or PaymentGateway without editing core code
  - Liskov: any PaymentGateway, including the circuit-breaker wrapper, can be substituted
  - Interface segregation: StockGate, ReservationRepository, ProcessedEventStore
  - Dependency inversion: services depend on interfaces, not on Redis or Kafka classes
- Design patterns: Strategy (pricing), Factory and Adapter (payment providers), State (order lifecycle), Observer (order events), Facade (checkout), Circuit Breaker (gateway calls)
- Example: adding PayU means one PayUAdapter class and one factory entry

Visual idea: A small, simplified class diagram of the payment or reservation module that shows the interfaces and their implementations, with the patterns labelled on the diagram itself. Avoid a grid of six pattern tiles.

Speaker notes: We applied SOLID to the three modules the brief calls critical: Inventory, Payment and Order. ReservationService depends only on the StockGate interface, so in tests we replace Redis with an in-memory version. The order lifecycle uses the State pattern, so an illegal move such as delivered back to payment pending throws an error instead of corrupting data. The full class, sequence and state diagrams are in our submission folder.

--------------------------------------------------------------------
Slide 8 — Scale and failure readiness
--------------------------------------------------------------------
Takeaway: The design keeps its guarantees at 50× traffic and when any single component fails.

On slide:
- At 50× traffic (500,000 buyers):
  - Waiting room becomes mandatory
  - Stock split across 10 Redis keys on different shards
  - Capacity scaled up 30 minutes before the sale starts
  - The database still sees only about 100 winners
- Failure and recovery:
  - Database primary fails: synchronous standby takes over in about 30 s with no data loss
  - Redis fails: replica promoted, stock reloaded from the database
  - Payment gateway down: circuit breaker opens; reservations are held until expiry
  - One availability zone lost: traffic routed to the other two
- Trade-offs we accept: brief pauses in reservations during a database failover, and a few seconds of "confirming order" after payment

Visual idea: A failure-to-recovery table or map as the focal point, with the 50× changes as a smaller supporting element.

Speaker notes: The likely bottleneck is the single Redis key holding the stock count, so at 50× we split the stock across ten keys. Autoscaling takes too long for a spike that arrives within five seconds, so we scale up in advance from the sale's start time. We chose consistency over availability for inventory: it is better to pause reservations for 30 seconds than to sell a unit we do not have. Monitoring raises an alert on any inventory mismatch, payment failure spike or growing Kafka backlog.

--------------------------------------------------------------------
Slide 9 — Thank you
--------------------------------------------------------------------
Takeaway: The design sells exactly 100 units with no double charges and no lost orders.

On slide:
- Thank you
- Result line: Exactly 100 sold, no double charges, every paid order confirmed or refunded
- Questions welcome
- [Team Name], SYSCRAFTERS 2026

Visual idea: Calm and minimal, echoing the slide 1 design.

Speaker notes: To answer the final jury question in one breath: the edge and waiting room filter the crowd, the Redis gate picks 100 winners, one database transaction per winner makes overselling impossible, idempotent payments prevent double charges, and Kafka with reconcilers makes sure every paid order is confirmed. Thank you, we are happy to take questions.

====================================================================
PART 2 — QUALITY CONSTRAINTS
Purpose: keep the deck from looking generic, templated or AI-generated.
You keep full creative freedom over style, colour, fonts and layout.
These rules only define what "good" looks like and what to avoid.
Follow every rule. If two rules seem to conflict, choose the option that
makes the slide clearer for an engineering jury.
====================================================================

A. CONTENT AND STORY
A1. One idea per slide. Each slide communicates exactly the takeaway given in Part 1. Every element must support that takeaway. Don't add extra facts that don't serve it.
A2. Clear three-part narrative: (1) Problem: slide 2; (2) Solution and evidence: slides 3–7; (3) Impact and readiness: slide 8. Slides 1 and 9 open and close. Each slide should lead naturally into the next; the end of slide 2 should make the audience want slide 3.
A3. Exactly 9 slides. No agenda, overview, divider or extra "about us" slides. If a slide has too much content, cut it and move the detail into the speaker notes.
A4. Minimal text on slides. Aim for roughly 40 words of body text or less per slide. Slides 4, 5 and 7 may run longer only if the content is inside a diagram or code block. Use short phrases, not paragraphs.
A5. No invented facts. Use only the facts in Part 1. Do not add statistics, percentages, user numbers, costs, quotes or claims. Keep [__] placeholders such as [Team Name] and [Name 1] exactly as they are and make them visible.
A6. Keep the specifics. Keep real names and numbers: 10,000 requests, 100 units, 500,000 requests per second, 50×, 20,000 per second, 5 requests per second, under 1 ms, p99 200 ms / 50 ms, 99.95%, 10-minute hold, 30 seconds, RPO 0, RTO under 5 min, 95% / 5% / 2%, Redis Lua, PostgreSQL, CHECK constraint, Kafka, transactional outbox, idempotency key, circuit breaker (50% failures, 30 s), 10 Redis keys, three availability zones. Never replace them with vague phrases such as "high performance" or "advanced architecture".

B. LANGUAGE AND WRITING
B1. Titles state a point. Each content slide title (slides 2–8) is a short claim, not a topic label.
    Weak: "Problem Statement". Better: "10,000 buyers competing for 100 units break a normal design"
    Weak: "Architecture". Better: "Contention is controlled in exactly one service"
    Weak: "Concurrency". Better: "One SQL statement makes overselling impossible"
    Exceptions: slide 1 (SALESTORM) and slide 9 (Thank you).
B2. One title style. All content titles are statements in sentence case, roughly 4–10 words. Don't mix questions, statements and single nouns.
B3. No buzzwords: leverage, synergy, drive impact, move the needle, in today's fast-paced world, revolutionize, game-changer, seamless, cutting-edge, state-of-the-art, unlock, empower, harness, robust (as filler), next-generation, transform the way, at the intersection of, delve, bulletproof, blazing fast, world-class.
B4. No dramatic sentence formulas: "It's not X, it's Y." / "X isn't just Y, it's Z." / "Imagine a world where…" / "What if we told you…" / one-word dramatic sentences ("Fast. Correct. Reliable.").
B5. Natural spoken language. Write the way a confident engineering student would explain the design out loud: plain, direct, accurate, active voice, short sentences. No marketing tone, exclamation marks or rhetorical questions.
B6. Consistent terminology. Use one name for each thing throughout: always "reservation" (never "hold" on one slide and "lock" on another), always "Inventory and Reservation service", always "waiting room", always "idempotency key", always "SALESTORM".

C. VISUAL DIRECTION
C1. Commit to one clear direction that suits a technical systems-architecture project, and apply it to the whole deck. Don't mix aesthetics.
C2. One repeated visual motif, used consistently on every slide: for example one way of drawing diagram nodes, a consistent connector/flow style, or one way of framing code. Stripes, bars, underlines and accent lines are not allowed as the motif.
C3. Be bold in one place only. Each slide gets at most one strong focal point (for example the funnel on slide 5, the Inventory service on slide 4). Everything around it stays calm.
C4. Consistent quality. Slides 7 and 8 must get the same design care as slide 1.
C5. Professional quality bar: the restraint and clarity of Apple keynotes, Stripe, Linear or Swiss/International typographic style. No stock templates or default layouts left unchanged.

D. LAYOUT AND COMPOSITION
D1. Vary the layouts. No two consecutive content slides share the same layout. Suggested matches (adapt freely):
    Slide 2: the 10,000 vs 100 imbalance as tension, with the three failures as consequences
    Slide 3: a weighted two-sided comparison (guarantees vs targets)
    Slide 4: a layered architecture diagram
    Slide 5: a narrowing funnel or stepped flow with real SQL
    Slide 6: outcomes paired with a timeline of the 30-second outage
    Slide 7: a simplified class diagram with patterns labelled on it
    Slide 8: a failure-to-recovery table or map
D2. No repeated "three/four/six equal boxes in a row" on more than one slide.
D3. Every slide has a meaningful visual (diagram, flow, table, code or structured graphic) that explains something. No text-only slides, and no decorative-only visuals.
D4. Grid and alignment. All elements sit on one consistent grid. Titles sit in the same position on every content slide.
D5. Whitespace. Use only about 60–70% of the slide area for content. Don't fill every corner or shrink text to fit more in.
D6. Consistent outer margins and identical gaps between similar elements on every slide.
D7. Clear hierarchy: title first, then the main visual, then supporting text. Someone glancing for 3 seconds should read the title first.
D8. Left-align body text, lists and table text. Centre only titles, single short statements, or the title and closing slides.

E. TYPOGRAPHY
E1. Every title uses the same typeface and weight.
E2. No single-word emphasis in headlines (no italic, coloured, bold, underlined or highlighted words inside a title).
E3. Use about 4–5 font sizes across the whole deck and reuse them exactly.
E4. Readable from the back of a room: body text no smaller than about 18 pt; table and diagram text no smaller than about 14–16 pt.
E5. No eyebrow labels (no small all-caps or letter-spaced labels above headings such as "THE PROBLEM" or "STEP 01").
E6. At most two typefaces in the whole deck. Monospace only for real code or technical identifiers (the SQL statement, CHECK constraint, class names), never as decoration.
E7. Sentence case for titles and text. ALL CAPS only for established acronyms (API, SQL, JWT, CDN, WAF, HLD, LLD, SOLID, RPO, RTO, DLQ) and state names shown as code (SOLD_OUT).

F. COLOUR
F1. A purposeful palette chosen for this topic: one dominant colour (about 60–70% of visual weight), one or two supporting tones, and at most one accent colour used sparingly for emphasis.
F2. Avoid overused palettes: cream/beige with terracotta or muted red; near-black with neon green, acid green or bright red; generic default corporate blue; purple, violet or blue-to-purple gradients; rainbow or many-coloured palettes.
F3. No decorative gradients (no gradient backgrounds, gradient text or washes).
F4. Strong contrast: WCAG AA at least (4.5:1 normal text, 3:1 large text). No grey on grey, light on light or dark on dark.
F5. Colour never carries meaning alone. If colour marks something (for example a failed payment vs a successful one, or the contention point), also mark it with a word, label or shape.
F6. Consistent colour meaning. If the accent marks "where correctness is enforced", it means exactly that on every slide.

G. DECORATION TO AVOID
G1. No accent lines under titles.
G2. No coloured bars or stripes on the top, left or side of boxes, cards or text.
G3. No uniform card kit (identical rounded cards with the same radius and soft grey shadow everywhere). If containers are used, their style should reflect importance, or use no containers at all.
G4. No numbering (01 / 02 / 03, Step 1 / Step 2) except on slide 5, where the steps are a real sequence. Numbers that are data (10,000, 100, 95%) are fine.
G5. No template chrome: meta lines joined by dots ("Redis · Kafka · PostgreSQL"), labels written as "WORD — phrase", arrows "→" appended to text, decorative quotation marks, sparkle or star icons, emoji, lightning bolts.
G6. No decorative icons. An icon is allowed only if it makes the meaning clearer, and all icons share one style (same stroke weight, fill and size).
G7. No background clutter: no blobs, particles, circuit-board patterns, glowing network lines or random geometric shapes.

H. IMAGES, DIAGRAMS AND DATA VISUALS
H1. No stock imagery: no shopping carts, shopping bags, crowds rushing into stores, sale/discount tags, handshakes, servers glowing in the dark, robots, holograms, people pointing at screens, abstract blue network images.
H2. Prefer explanatory visuals that show how SALESTORM actually works: the layered architecture, the 10,000-to-100 funnel, the SQL statement, the outage timeline, the class diagram, the failure-to-recovery table, and the state flows (reservation: AVAILABLE, RESERVED, PAYMENT_PENDING, CONFIRMED, SOLD, or RELEASED on failure; order: CREATED, PAYMENT_PENDING, CONFIRMED, PROCESSING, SHIPPED, OUT_FOR_DELIVERY, DELIVERED).
H3. One point per visual. Every diagram, table or chart supports the one insight stated in the slide title. No dashboard-style rows of big-number tiles without a story.
H4. Readable diagrams and tables: clear labels, aligned columns, no 3D effects, no unnecessary gridlines, direct labels instead of legends.
H5. Consistent visual language: all diagrams use the same node shapes, line weights, arrow styles and labelling style. Show synchronous vs asynchronous calls the same way everywhere they appear (for example solid vs dashed lines, labelled once).

I. MOTION AND TRANSITIONS
I1. No entrance animations on every element and no flashy transitions (spin, bounce, zoom, 3D flip). At most one or two deliberate build animations in the whole deck, ideally the funnel on slide 5 narrowing step by step. Transitions: none or a subtle fade, the same on every slide.

J. SPEAKER NOTES
J1. Put the provided speaker notes on every slide (3–5 natural spoken sentences).
J2. Notes carry the reasoning removed from the slide (why optimistic locking fails here, why we never retry a payment blindly, why consistency beats availability for inventory).
J3. Notes follow the same honesty and language rules: no invented facts, no buzzwords, plain spoken language.

K. FINAL QUALITY CHECK (perform before finishing)
Go through every slide and confirm each item. Fix anything that fails.
K1.  No text overflows its box, is cut off, or overlaps another element.
K2.  No leftover template text or sample images. The [__] placeholders stay visible.
K3.  Titles sit in the same position on every content slide.
K4.  Margins, gaps and alignment are consistent on all 9 slides.
K5.  Only the planned 4–5 font sizes and at most two typefaces are used.
K6.  Every slide has one clear takeaway and one meaningful visual.
K7.  No banned decorations from section G and no banned imagery from H1.
K8.  No banned words or sentence formulas from section B.
K9.  All numbers and technical names match Part 1 exactly; nothing invented.
K10. Contrast is strong on every slide, including diagram labels, code and table text.
K11. Every slide has speaker notes.
K12. Final look: if any slide could have come from a generic AI or template deck, redesign that slide.
