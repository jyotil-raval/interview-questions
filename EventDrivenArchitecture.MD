<!-- Generated: 2026-03-29 | Event-Driven Architecture + Microservices Patterns -->

# Event-Driven Architecture & Microservices Interview Preparation

## Tech Lead / Architect Level

> Patterns · Saga · CQRS · Event Sourcing · Outbox · Choreography · Orchestration
> Generated: 2026-03-29 | Career OS

---

## Scope & Relationship to Existing Files

This file covers **architectural patterns** — the layer above broker mechanics.

Broker mechanics (Kafka, RabbitMQ, SQS/SNS) are in:

- [Messaging](./Messaging.md) — Kafka internals, RabbitMQ AMQP, delivery semantics
- [AWS](./AWS.md) — SQS, SNS, EventBridge, fan-out, DLQ

This file covers what those files don't:

- **Why** event-driven over synchronous — the architectural motivation
- **Choreography vs Orchestration** — the fundamental EDA architectural choice
- **Saga pattern** — distributed transactions without a distributed transaction manager
- **CQRS** — Command Query Responsibility Segregation
- **Event Sourcing** — state as a sequence of events
- **Transactional Outbox** — solving dual-write without distributed transactions
- **Idempotency** — the non-negotiable consumer contract
- **Eventual consistency** — handling it in practice, not in theory
- **Domain events vs Integration events** — the boundary that matters
- **Event schema evolution** — versioning events across services
- **EDA failure modes** — what breaks in production that theory doesn't predict
- **Decision logic** — when EDA is wrong for a use case

At Architect level, interviewers don't ask "what is a message queue."
They ask: "Walk me through how you'd handle a distributed payment transaction
across 4 services without a 2PC. What happens when the inventory deduction
succeeds but the payment charge fails? How do you recover?"

That question lives in this file.

---

**L1–L6 Topics identified:**

- Why event-driven: coupling, scalability, resilience — the architectural motivation
- Synchronous vs Asynchronous — when each applies
- Choreography vs Orchestration — the fundamental structural choice
- Domain events vs Integration events vs Commands
- Saga pattern — choreography-based and orchestration-based
- CQRS — model separation, read/write scaling
- Event Sourcing — the event log as truth
- Transactional Outbox Pattern — solving dual-write
- Idempotency — consumer contract design
- Eventual consistency — practical handling
- Event schema evolution and CloudEvents
- EDA at scale — ordering, partitioning, backpressure
- When NOT to use EDA — the anti-patterns

---

## L1 — Why Event-Driven: The Architectural Motivation

---

### Coupling, Scalability & Resilience

- **Question:** What problem does event-driven architecture solve that synchronous REST/gRPC does not, and what are the three core architectural benefits?
- **Answer:**
  - **The core problem — temporal and behavioural coupling:** In synchronous REST, Service A directly calls Service B. This creates two coupling problems: (1) **Temporal coupling** — A cannot complete its work unless B is available right now. If B is down for 30 seconds, A fails for 30 seconds. (2) **Behavioural coupling** — A must know B exists, know B's API contract, and handle B's error responses. Every new consumer of A's data requires A to change or add new calls.
  - **EDA solution:** A emits an event (something happened — `OrderPlaced`). A doesn't know who consumes it. B, C, D can independently subscribe. A completes without waiting for anyone. If B is down, B catches up when it recovers. This is **temporal decoupling** (work happens at different times) and **behavioural decoupling** (A doesn't know consumers exist).
  - **Three core architectural benefits:**
    1. **Resilience:** Consumers fail independently. A broker (Kafka/SQS) buffers events — no data loss when a consumer is temporarily unavailable. A synchronous chain has cascading failure; an async chain has isolated failure.
    2. **Scalability:** Each consumer scales independently based on its own load. The producer doesn't need to know how many consumers exist or how fast they process. Adding a new consumer requires zero producer changes.
    3. **Auditability:** Every event in the log is an immutable record of what happened. The full system history is available for debugging, replay, compliance, and analytics without hitting production databases.
  - **The cost:** EDA introduces **eventual consistency** (consumers process at different rates), **operational complexity** (broker infrastructure, monitoring consumer lag, DLQ management), and **harder debugging** (no single call stack spans the entire operation).
- **Example:**

```
Synchronous chain — tight coupling, cascading failure:
OrderService → [sync call] → InventoryService → [sync call] → PaymentService
                                                                     ↑
                              PaymentService down 30s = OrderService fails 30s

Event-driven — decoupled, isolated failure:
OrderService → emits OrderPlaced → [Kafka topic]
                                         ↓               ↓              ↓
                                  InventoryService  PaymentService  AuditService
                                  (processes when     (processes     (processes
                                  it's ready)         when ready)    asynchronously)

PaymentService down 30s = events queue in Kafka, processed on recovery.
OrderService is unaffected. InventoryService is unaffected.
```

- **Technical Terms to Include:** temporal coupling, behavioural coupling, producer, consumer, broker, event, at-least-once delivery, eventual consistency, cascading failure, consumer lag, decoupling, fan-out
- **Gotcha:** EDA does not eliminate failures — it **isolates** them. A system where the payment service processes events 6 hours late because of a bug is not "working." It's failing in a way that's harder to observe than a synchronous timeout. Consumer lag monitoring, DLQ alerting, and per-consumer SLOs are what make EDA failures visible. Without them, EDA gives you invisible failure rather than isolated failure.
- **Follow-Up:** "Does event-driven architecture always outperform synchronous?" → No. For a single read request that needs an immediate answer (get user profile, calculate cart total), event-driven adds latency and complexity with no benefit. EDA is appropriate for: mutations that fan out to multiple consumers, workflows that are acceptable to complete asynchronously, systems where producer and consumer scale differently. Synchronous is appropriate for: queries requiring immediate consistent responses, simple CRUD, tightly coupled workflows where partial completion is not acceptable. → They're testing decision selectivity.
- **Conclusion:** EDA's value is decoupling in time and behaviour — it makes systems more resilient and scalable at the cost of eventual consistency and operational complexity; the architectural choice should be driven by whether the use case actually benefits from decoupling, not by a preference for modern patterns.

---

## L2 — Choreography vs Orchestration

---

### The Fundamental Structural Choice

- **Question:** What is the difference between choreography and orchestration in EDA, what are the trade-offs of each, and how do you choose between them?
- **Answer:**
  - **Choreography:** Services react to events without central coordination. When Service A emits `OrderPlaced`, Service B (Inventory) independently subscribes and reacts by emitting `StockReserved`. Service C (Payment) subscribes to `StockReserved` and emits `PaymentCharged`. No service knows about the overall workflow — each only knows its own local event-reaction rule.
  - **Orchestration:** A central orchestrator (saga orchestrator, workflow engine — Temporal, AWS Step Functions) explicitly directs each service: "Do step 1. Wait for result. Do step 2. If step 2 fails, compensate step 1." The orchestrator holds the workflow state and explicitly commands each participant.
  - **Choreography — advantages:** No single point of failure in the coordination layer. Services are fully decoupled — adding a new reaction to `OrderPlaced` requires no change to the producer. Scales naturally with service count.
  - **Choreography — disadvantages:** The workflow logic is implicit — distributed across N service subscriptions. Debugging requires reconstructing the flow from events across multiple services. Adding a new step requires touching multiple services. Very hard to determine "what state is this order in right now?"
  - **Orchestration — advantages:** Workflow is explicit and visible in one place. Error handling, compensation, timeouts are all in the orchestrator. Debugging: one place to look. Easy to add new steps without touching existing services. The orchestrator can show real-time workflow state.
  - **Orchestration — disadvantages:** The orchestrator is a central coupling point — services now know about (and are called by) the orchestrator. Orchestrator becomes a bottleneck and a SPoF if not designed for HA. More complex to build and operate than simple event subscriptions.
  - **Decision rule:** Choreography for simple, low-step fan-out workflows (notify multiple services of a fact). Orchestration for multi-step business processes with compensations and timeouts (order processing, onboarding, financial transactions).
- **Example:**

```
CHOREOGRAPHY — Order Fulfilment (3 services react independently)

OrderService:
  User places order → emits: OrderPlaced { orderId, items, userId }

InventoryService (subscribes to OrderPlaced):
  receives OrderPlaced → reserves stock → emits: StockReserved { orderId }
  OR stock unavailable → emits: StockReservationFailed { orderId, reason }

PaymentService (subscribes to StockReserved):
  receives StockReserved → charges card → emits: PaymentCharged { orderId }
  OR charge fails → emits: PaymentFailed { orderId, reason }

NotificationService (subscribes to PaymentCharged):
  sends confirmation email

Problem: To know the current state of order-42, you must query all 4 services
or reconstruct the event stream. There's no single "order status" source.

---

ORCHESTRATION — Same flow, visible state (Temporal / Step Functions)

OrderOrchestrator (saga):
  Step 1: Command InventoryService.Reserve(orderId, items)
  Wait for result (timeout: 30s)
    Success → Step 2
    Failure → emit OrderFailed, exit

  Step 2: Command PaymentService.Charge(orderId, amount)
  Wait for result (timeout: 60s)
    Success → Step 3
    Failure → compensate: Command InventoryService.Release(orderId) → emit OrderFailed

  Step 3: Command NotificationService.SendConfirmation(orderId)
  emit OrderCompleted

State visible at all times: "order-42 is at Step 2 — awaiting payment"
```

- **Technical Terms to Include:** choreography, orchestration, saga orchestrator, Temporal, AWS Step Functions, workflow state, compensation, fan-out, event subscription, coupling point, SPoF (Single Point of Failure), event-reaction rule, implicit vs explicit workflow
- **Gotcha:** In choreography, the **"what is the current state of this business process?"** question has no good answer. You either maintain a process-state projection (another service that listens to all events and maintains state — essentially building an implicit orchestrator), or you join events across multiple services to reconstruct state at query time. Teams that start with choreography frequently end up building this projection service and realising they built an orchestrator the hard way.
- **Follow-Up:** "Can you use both in the same system?" → Yes, and this is common. Orchestration for core business workflows (order processing, onboarding) where state visibility matters. Choreography for side effects that don't affect the core workflow (analytics, notifications, audit logging). The rule: if the workflow's correctness depends on the step, orchestrate it. If the step is a reaction that doesn't affect the workflow outcome, choreograph it. → They're testing hybrid architecture design.
- **Conclusion:** Choreography gives autonomy at the cost of observability; orchestration gives visibility at the cost of coupling — the choice should be made per workflow type, and most production systems use both: orchestration for the core transaction, choreography for notifications and side effects.

---

## L3 — Saga Pattern: Distributed Transactions Without 2PC

---

### Saga — Choreography-Based & Orchestration-Based

- **Question:** What is the Saga pattern, why does it replace 2-Phase Commit in microservices, and how do compensating transactions work?
- **Answer:**
  - **The problem:** In a monolith, a database transaction spans all operations — if any step fails, the entire transaction rolls back. In microservices, each service has its own database. A traditional distributed transaction (2PC — Two-Phase Commit) requires a coordinator that locks resources across all services while waiting for all to confirm. This creates: long lock hold times, high latency, cascading failure if the coordinator fails, and tight coupling. 2PC does not work in practice for internet-scale microservices.
  - **Saga:** A sequence of local transactions, each published as an event. Each local transaction updates one service's database and publishes an event triggering the next step. If any step fails, **compensating transactions** are executed in reverse order to undo the already-completed steps. There are no locks — each service is available throughout.
  - **Compensating transaction:** The semantic undo of a completed step. Not a rollback (the data was committed) — a new forward operation that negates the effect. `ReserveStock` is compensated by `ReleaseStock`. `ChargePayment` is compensated by `RefundPayment`. Important: compensations must be idempotent (can be retried safely) and must always succeed — a compensation that can fail requires a compensation-for-the-compensation.
  - **Two implementation styles:**
    - **Choreography-based Saga:** Services emit events; each service listens for the event that triggers its step and its compensation trigger. No central coordinator. Failure handling is event-driven (on `PaymentFailed`, InventoryService reacts with `StockRelease`). Harder to observe.
    - **Orchestration-based Saga:** A saga orchestrator explicitly calls each service and handles the failure path. More visible, easier to debug, more coupling to the orchestrator.
  - **Saga does not provide isolation:** Two concurrent sagas can see each other's intermediate state (no ACID isolation). Countermeasure: semantic locking (mark a record as "pending" during the saga, preventing other sagas from modifying it until this one completes or compensates).
- **Example:**

```
SAGA: Order Placement — Choreography-based

Step 1: OrderService creates Order in PENDING state → emits OrderPlaced
Step 2: InventoryService receives OrderPlaced → reserves stock → emits StockReserved
Step 3: PaymentService receives StockReserved → charges card → emits PaymentCharged
Step 4: OrderService receives PaymentCharged → marks Order as CONFIRMED → emits OrderConfirmed

--- FAILURE at Step 3 (payment declined): ---

PaymentService emits: PaymentFailed { orderId, reason: "insufficient_funds" }

Compensation chain (in reverse):
  InventoryService receives PaymentFailed → releases reserved stock → emits StockReleased
  OrderService receives StockReleased → marks Order as CANCELLED → emits OrderCancelled

--- The compensating transactions must be IDEMPOTENT ---
--- If PaymentFailed is delivered twice, StockReleased runs twice ---
--- InventoryService.release() must be safe to call N times ---

// Idempotent compensation — uses orderId as deduplication key
func (s *InventoryService) ReleaseStock(ctx context.Context, orderId string) error {
    reservation, err := s.db.GetReservation(ctx, orderId)
    if err != nil { return err }
    if reservation.Status == "RELEASED" {
        return nil // already released — idempotent, safe to return success
    }
    return s.db.UpdateReservation(ctx, orderId, "RELEASED")
}
```

```
SAGA: Order Placement — Orchestration-based (Temporal workflow)

func OrderFulfillmentSaga(ctx workflow.Context, order Order) error {
    // Step 1: Reserve inventory
    var reserved bool
    err := workflow.ExecuteActivity(ctx, InventoryActivity.Reserve, order.Items).Get(ctx, &reserved)
    if err != nil { return markOrderFailed(ctx, order.ID, err) }

    // Step 2: Charge payment
    var charged bool
    err = workflow.ExecuteActivity(ctx, PaymentActivity.Charge, order.Amount).Get(ctx, &charged)
    if err != nil {
        // Compensate: release the reservation
        _ = workflow.ExecuteActivity(ctx, InventoryActivity.Release, order.Items).Get(ctx, nil)
        return markOrderFailed(ctx, order.ID, err)
    }

    // Step 3: Confirm order
    return workflow.ExecuteActivity(ctx, OrderActivity.Confirm, order.ID).Get(ctx, nil)
}
// Temporal provides: automatic retries, durable state (survives process restart),
// activity timeouts, compensation on failure — all without writing infrastructure code
```

- **Technical Terms to Include:** Saga, 2PC (Two-Phase Commit), local transaction, compensating transaction, semantic lock, ACID isolation, eventual consistency, choreography-based saga, orchestration-based saga, Temporal, AWS Step Functions, idempotent compensation, forward recovery, backward recovery
- **Gotcha:** **Compensations are not rollbacks.** A compensation is a new transaction in the forward direction that semantically negates a prior action. If you charged €100 to a card, you can't "roll back" the charge — the bank has already processed it. The compensation is a €100 refund, which is a new credit transaction. This matters for design: compensations must be explicitly designed as first-class operations, not assumed to be automatic.
- **Follow-Up:** "What is the 'pivot transaction' in a saga?" → The pivot transaction is the point of no return — after it completes, the saga must complete forward (no compensation possible). All steps before the pivot can be compensated. All steps after must complete, retrying if necessary. Identifying the pivot determines where you place the most effort on resilience (the steps after the pivot must always succeed). In an order saga, the pivot is typically the payment charge — after money is taken, the saga must complete fulfilment. → They're testing saga design precision.
- **Conclusion:** The Saga pattern trades ACID atomicity for availability and performance — it replaces distributed locks with compensating transactions, requires explicitly designed undos for every step, and demands idempotency throughout; it is the standard distributed transaction pattern for internet-scale microservices and the correct answer to "how do you handle distributed transactions without 2PC."

---

## L4 — CQRS & Event Sourcing

---

### CQRS — Command Query Responsibility Segregation

- **Question:** What is CQRS, what problem does it solve, and what does the read/write model separation look like in practice?
- **Answer:**
  - **CQRS** separates the data model for writes (Commands — change state) from the data model for reads (Queries — retrieve state). In a traditional CRUD architecture, the same model serves both — you read and write from the same table with the same schema. This creates friction when the optimal write model differs from the optimal read model.
  - **The problem it solves:** (1) **Write models** are optimised for transactional integrity — normalised, constraint-enforced, ACID. (2) **Read models** are optimised for query performance — denormalised, pre-aggregated, shaped to the specific UI requirement. A single model can't be optimal for both simultaneously. At scale, complex JOIN queries on a normalised write model cause contention with writes. CQRS separates these concerns completely.
  - **How it works:** Commands go to the write side (Command Handler → validates → updates write store → emits domain event). Events are consumed by projectors that update the read store(s). Read queries go directly to the read store — no joins, pre-computed views, shaped exactly to the query. The read store can be a separate database, a different database type (relational write store, Elasticsearch/Redis read store), or both.
  - **CQRS is not always event-sourced:** CQRS and Event Sourcing are independent patterns. CQRS = separate models. Event Sourcing = state as event log. They complement each other but each can be used without the other.
  - **Trade-off:** Read stores are eventually consistent with the write store — the time between a command completing and the read projection updating is the propagation delay (milliseconds to seconds typically). UX must tolerate this (optimistic UI, "your changes are being processed").
- **Example:**

```
TRADITIONAL CRUD — one model, tension between reads and writes
Table: orders (id, user_id, status, created_at, total_amount, shipping_address, ...)
Write: INSERT / UPDATE on the same table
Read: SELECT with JOINs to users, order_items, products, addresses
→ Complex queries contend with writes at scale

CQRS — separate models

WRITE SIDE:
  Command: PlaceOrder { userId, items, shippingAddressId }
  CommandHandler validates → writes to orders_write (normalised, transactional)
  Emits: OrderPlaced { orderId, userId, items, timestamp }

READ SIDE (multiple projections from the same events):
  Projection 1 — OrderSummaryView (for dashboard):
    { orderId, status, createdAt, total, itemCount }
    Updated by: OrderPlaced, OrderShipped, OrderCancelled
    Stored in: PostgreSQL read-replica (fast SELECT, no JOINs)

  Projection 2 — UserOrderHistoryView (for profile page):
    { userId, orders: [{ orderId, date, status, total }] }
    Stored in: Redis hash per userId (sub-ms read)

  Projection 3 — FullOrderDetailView (for order detail page):
    Denormalised single document with all nested data
    Stored in: MongoDB document store

// Each query goes directly to its projection — no JOINs, pre-computed
GET /api/orders?userId=42          → reads UserOrderHistoryView from Redis
GET /api/orders/99                 → reads FullOrderDetailView from MongoDB
GET /api/admin/orders              → reads OrderSummaryView from PostgreSQL
```

- **Technical Terms to Include:** CQRS, Command, Query, Command Handler, projection, read model, write model, eventual consistency, projector, read store, write store, domain event, denormalisation, read-replica, optimistic UI, stale reads
- **Gotcha:** **CQRS doesn't mean you need a separate database.** For many applications, two tables in the same database (one for write, one for read projection) provides the separation benefits without the operational complexity of multiple data stores. Only introduce separate data stores when the query requirements genuinely demand it (full-text search → Elasticsearch, sub-ms lookup → Redis). Start simple: same DB, separate models. Add data stores only when the query demands it.
- **Follow-Up:** "How do you handle the case where a user performs a write and immediately queries the data, but the projection hasn't updated yet?" → Three approaches: (1) **Return the command result directly** — after a successful command, return the new state computed by the command handler (the write side knows what it wrote). No query needed immediately. (2) **Optimistic UI** — update the UI immediately assuming success, reconcile on the next query. (3) **Wait for projection** — poll or use a correlation ID to wait until the read store reflects the change (adds latency, complex). In practice: approach 1 or 2 for most UX flows. Approach 3 only for safety-critical reads. → They're testing eventual consistency UX handling.
- **Conclusion:** CQRS solves the read/write model tension at scale by making them first-class separate concerns — the read model is a pre-computed projection of events optimised for each specific query, eliminating JOIN contention and enabling per-query data store selection, at the cost of eventual consistency between the two sides.

---

### Event Sourcing — State as Event Log

- **Question:** What is Event Sourcing, how does it differ from storing current state, and what are the operational realities of a production event-sourced system?
- **Answer:**
  - **Traditional state storage:** The database stores the current state of an entity. When the entity changes, the old state is overwritten. History is lost (unless you build audit logging separately). The question "what was this order's status at 14:32 yesterday?" is unanswerable without an audit trail.
  - **Event Sourcing:** The database stores the **sequence of events** that happened to an entity. Current state is derived by replaying the event sequence. Nothing is overwritten — the log is append-only. The complete history is the data.
  - **Core operations:** **Append** — add a new event to the entity's stream. **Load** — read all events for an entity and replay them to reconstruct current state (apply each event's state change function). **Snapshot** — periodic snapshot of current state to avoid replaying the full event history on every load (for entities with thousands of events).
  - **Benefits:** (1) Full audit trail — every state change is recorded with timestamp and causation. (2) Temporal queries — "what was the state at time T?" (replay up to T). (3) Event replay — reprocess all events with new logic (correct a bug, build a new projection). (4) CQRS natural fit — events are the source for all read projections.
  - **Real operational costs:** (1) **Schema evolution** — old events must remain deserializeable forever. Upcasters transform old event formats to current version. (2) **Replay performance** — loading an entity with 50,000 events requires 50,000 state applications. Snapshots required. (3) **Debugging complexity** — current state is not directly visible in the database; must replay to see it. (4) **Event store operations** — standard databases are not designed for event store access patterns; specialised stores (EventStoreDB, Marten) or Kafka topics are common.
- **Example:**

```
TRADITIONAL: BankAccount table
  id: 42
  balance: 850  // current state — history gone

EVENT SOURCED: BankAccount event stream (id: 42)
  Event 1: AccountOpened    { initialBalance: 1000, timestamp: 2026-01-01 }
  Event 2: MoneyDeposited   { amount: 500,  timestamp: 2026-01-15 }
  Event 3: MoneyWithdrawn   { amount: 200,  timestamp: 2026-02-01 }
  Event 4: MoneyWithdrawn   { amount: 450,  timestamp: 2026-03-01 }

// Reconstruct current state by replaying:
func (a *BankAccount) Apply(event Event) {
    switch e := event.(type) {
    case AccountOpened:    a.Balance = e.InitialBalance
    case MoneyDeposited:   a.Balance += e.Amount
    case MoneyWithdrawn:   a.Balance -= e.Amount
    }
}
// Result: 1000 + 500 - 200 - 450 = 850 ✅

// Temporal query: balance at 2026-02-01 — replay up to that date: 1000 + 500 - 200 = 1300

// Snapshot — after N events, save current state to avoid full replay
type Snapshot struct {
    AggregateID string
    Version     int    // event index at time of snapshot
    State       []byte // serialised current state
    Timestamp   time.Time
}
// Load: find latest snapshot → replay only events after the snapshot version
```

- **Technical Terms to Include:** Event Sourcing, event stream, append-only, event replay, aggregate, apply function, snapshot, upcaster, temporal query, EventStoreDB, Marten, schema evolution, projection rebuild, event store, causation ID, correlation ID
- **Gotcha:** **GDPR and Event Sourcing are in tension.** GDPR requires the ability to delete a user's personal data ("right to erasure"). Event Sourcing's append-only log cannot delete events. Resolution: (1) **Crypto-shredding** — encrypt personal data in events with a per-user key stored separately. On deletion request, delete the encryption key. Old events become unreadable. (2) **Separate PII** — store personal data by reference (user ID) in events; the actual PII is in a deletable separate store. Know this before proposing Event Sourcing for any EU-facing application.
- **Follow-Up:** "How do you rebuild a read projection from scratch with Event Sourcing?" → Replay all events from the beginning of the event stream (or all streams) through the projection handler into a new empty projection store. This is the "killer feature" of event sourcing: you can fix a bug in a projection, drop the old projection, and rebuild it perfectly from the event history. The same events that built the flawed projection build the correct one. In practice: build the new projection in parallel against the event stream, then cut over once it's caught up. → They're testing the operational value proposition of event sourcing.
- **Conclusion:** Event Sourcing trades current-state simplicity for complete history — it is the correct choice when audit trails, temporal queries, and projection rebuilds are genuine requirements, but its operational complexity (schema evolution, GDPR compliance, snapshot management) means it should not be adopted by default — only where history is a first-class business requirement.

---

## L5 — Transactional Outbox & Idempotency

---

### Transactional Outbox Pattern — Solving Dual-Write

- **Question:** What is the dual-write problem in event-driven microservices, and how does the Transactional Outbox pattern solve it reliably?
- **Answer:**
  - **The dual-write problem:** A service needs to (1) update its database and (2) publish an event to a message broker. These are two separate I/O operations. If the service updates the database and then crashes before publishing the event, the database has the new state but no event was emitted — downstream services miss the update. Conversely, if the event is published first and then the database write fails, consumers react to an event for something that didn't actually happen.
  - **Why you can't just use a transaction:** A database transaction wraps the DB write. The message broker is not part of the transaction — you cannot atomically commit a DB write and a Kafka publish in one transaction (distributed transaction — the 2PC problem again).
  - **Transactional Outbox:** Instead of publishing to the broker directly, the service writes the event to an **outbox table** in the same database transaction as the state change. Both succeed atomically (same transaction) or both fail. A separate **relay process** reads unpublished events from the outbox table and publishes them to the broker. Once published, the event is marked as processed (or deleted).
  - **Relay implementations:** (1) **Polling publisher** — periodically queries the outbox table for unprocessed events. Simple, adds DB load, small latency (polling interval). (2) **CDC (Change Data Capture)** via Debezium — reads the database's transaction log (PostgreSQL WAL, MySQL binlog) and streams all changes, including outbox insertions, to Kafka. Zero additional DB queries, sub-second latency, requires Kafka Connect + Debezium deployment.
  - **At-least-once guarantee:** The relay may publish the same event more than once (failure after publish, before marking processed → re-publish on next poll). Consumers must be idempotent. This is the expected operating condition — design for it.
- **Example:**

```sql
-- Outbox table in the same service database
CREATE TABLE outbox_events (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_type  TEXT NOT NULL,          -- 'OrderPlaced', 'PaymentCharged'
    aggregate_id TEXT NOT NULL,         -- 'order-42'
    payload     JSONB NOT NULL,         -- full event data
    created_at  TIMESTAMPTZ DEFAULT NOW(),
    published_at TIMESTAMPTZ,           -- NULL = unpublished
    sequence_nr  BIGSERIAL              -- for ordering within aggregate
);
```

```go
// In the same database transaction: update state + write to outbox
func (s *OrderService) PlaceOrder(ctx context.Context, order Order) error {
    return s.db.WithTransaction(ctx, func(tx *sql.Tx) error {
        // Step 1: Insert the order
        if err := insertOrder(tx, order); err != nil { return err }

        // Step 2: Insert outbox event (SAME TRANSACTION — atomic)
        event := OutboxEvent{
            EventType:   "OrderPlaced",
            AggregateID: order.ID,
            Payload:     mustMarshal(OrderPlacedEvent{
                OrderID: order.ID, UserID: order.UserID, Items: order.Items,
            }),
        }
        if err := insertOutboxEvent(tx, event); err != nil { return err }

        return nil // commit: both order + outbox event written atomically
    })
}

// Relay process — reads outbox, publishes to Kafka, marks published
func (r *OutboxRelay) Run(ctx context.Context) {
    ticker := time.NewTicker(500 * time.Millisecond) // poll every 500ms
    for {
        select {
        case <-ticker.C:
            events, _ := r.db.GetUnpublishedEvents(ctx, 100)
            for _, event := range events {
                if err := r.kafka.Publish(ctx, event); err != nil {
                    log.Error("publish failed", "id", event.ID, "err", err)
                    continue // will retry on next poll
                }
                r.db.MarkPublished(ctx, event.ID) // safe even if called twice
            }
        case <-ctx.Done(): return
        }
    }
}
```

- **Technical Terms to Include:** dual-write problem, Transactional Outbox, outbox table, relay, CDC (Change Data Capture), Debezium, PostgreSQL WAL, polling publisher, at-least-once delivery, idempotent consumer, `published_at`, atomic commit, Kafka Connect
- **Gotcha:** The outbox table **grows indefinitely** if events are never cleaned up. A busy service with 10,000 orders/day adds 10,000 rows to the outbox daily. After marking an event as published, schedule deletion or archiving. Either: hard delete after publishing, move to an archive table (for audit), or use a retention policy. Without cleanup, the outbox table becomes a production performance problem within weeks.
- **Follow-Up:** "How does Debezium CDC compare to polling outbox in production?" → **Polling:** Simple to implement, adds periodic DB queries, small publish latency (equal to poll interval), requires the outbox table. Works with any database. **CDC (Debezium):** Reads the PostgreSQL WAL directly — no additional DB queries, sub-second publish latency, but requires Kafka Connect + Debezium connector deployment and operations. CDC is also more resilient to application crashes — the WAL is the source of truth, not in-memory state. Choose polling for simpler stacks. Choose Debezium when low publish latency matters or you already operate Kafka Connect. → They're testing relay implementation trade-off depth.
- **Conclusion:** The Transactional Outbox is the correct solution to the dual-write problem — writing the event to the database in the same transaction as the state change guarantees at-least-once event delivery, eliminates the gap between "database updated" and "event published," and is a prerequisite for any reliable event-driven microservice.

---

### Idempotency — The Non-Negotiable Consumer Contract

- **Question:** What is idempotency in the context of event-driven consumers, how do you implement it, and what are the patterns for detecting and handling duplicates?
- **Answer:**
  - **Why idempotency is non-negotiable:** Message brokers (Kafka, SQS, RabbitMQ) guarantee at-least-once delivery — not exactly-once. A consumer will receive the same event more than once in these scenarios: (1) Broker delivers, consumer processes, consumer crashes before ACK → redelivery. (2) Network timeout during ACK → broker retries delivery. (3) Kafka rebalance during processing → uncommitted offset reprocessed. (4) Outbox relay publishes twice (retry on publish failure). An event consumer that is not idempotent will double-charge customers, double-reserve inventory, or send duplicate emails.
  - **Idempotency key:** Every event must carry a stable, unique identifier — the `eventId` (UUID generated by the producer). The consumer uses this to detect duplicates.
  - **Idempotency implementation patterns:**
    1. **Idempotency table (processed events log):** Consumer writes the `eventId` to a `processed_events` table before processing (or atomically with processing). On receipt, check: if `eventId` already in table → skip. Requires one DB write per event. Deduplication window limited by table retention.
    2. **Natural idempotency (upsert):** The operation is naturally idempotent. `UPDATE orders SET status='CONFIRMED' WHERE id=? AND status='PENDING'` — running this twice is safe (second run finds no rows to update). No dedup table needed if the state machine enforces valid transitions.
    3. **Conditional write (optimistic):** Use the event's sequence number + aggregate version. Only apply if current version matches expected. Duplicate events have incorrect expected version → rejected without double-apply.
    4. **Exactly-once with Kafka transactions:** Kafka's transactional producer + `read_committed` consumer isolation provides exactly-once within Kafka (not end-to-end if the consumer writes to another system). Practical for Kafka Streams but not for consumers writing to external databases.
- **Example:**

```go
// Pattern 1 — Idempotency table
func (h *PaymentHandler) Handle(ctx context.Context, event OrderPlacedEvent) error {
    // Check if already processed — in same transaction as processing
    return h.db.WithTransaction(ctx, func(tx *sql.Tx) error {
        var exists bool
        tx.QueryRowContext(ctx,
            "SELECT EXISTS(SELECT 1 FROM processed_events WHERE event_id = $1)",
            event.EventID,
        ).Scan(&exists)

        if exists {
            return nil // already processed — safe to skip
        }

        // Process the event
        if err := chargePayment(tx, event.OrderID, event.Amount); err != nil {
            return err
        }

        // Record as processed (SAME TRANSACTION — atomic with the charge)
        _, err := tx.ExecContext(ctx,
            "INSERT INTO processed_events (event_id, processed_at) VALUES ($1, NOW())",
            event.EventID,
        )
        return err
    })
    // If commit fails: neither charge nor event record committed → retry is safe
    // If commit succeeds: idempotent — second delivery finds event_id in table
}

// Pattern 2 — Natural idempotency (state machine upsert)
func (h *OrderHandler) ConfirmOrder(ctx context.Context, event PaymentChargedEvent) error {
    result, err := h.db.ExecContext(ctx,
        "UPDATE orders SET status='CONFIRMED', confirmed_at=NOW() WHERE id=$1 AND status='PENDING'",
        event.OrderID,
    )
    rows, _ := result.RowsAffected()
    if rows == 0 {
        // Either: already confirmed (duplicate) or invalid state transition
        // Either way: safe to return nil — no double-confirm
    }
    return err
}
```

- **Technical Terms to Include:** idempotency, at-least-once delivery, `eventId`, idempotency key, deduplication, processed events table, natural idempotency, upsert, state machine transition, optimistic concurrency, `RowsAffected`, exactly-once semantics, Kafka transactions, `read_committed`
- **Gotcha:** Writing the `eventId` to the processed events table **before** processing (not after) is critical. If you process first and then record — a crash between the two leaves the event unrecorded, so the next delivery is processed again (duplicate processing). If you record first and then process — a crash between the two leaves the event recorded as processed but not actually processed (silent data loss). The only correct approach: **atomic transaction containing both the processing and the processed_events insert**. If either fails, both fail, and the retry is safe.
- **Follow-Up:** "How long do you retain the processed events table?" → Long enough to cover the maximum expected redelivery window. For SQS, messages are retained up to 14 days — your dedup window must cover at least 14 days. For Kafka, the consumer offset determines the redelivery risk — a rebalance can cause any uncommitted message to be redelivered, but won't go back further than the earliest uncommitted offset. Typically: retain processed event IDs for 30 days minimum, partition by day for efficient deletion. Use TTL or scheduled cleanup. → They're testing dedup retention architecture.
- **Conclusion:** Idempotency is not an optimisation — it is a correctness requirement for any event consumer in an at-least-once delivery system; the processed events table with atomic commit is the general-purpose solution, natural idempotency via state machine transitions is the elegant solution when the domain supports it.

---

## L6 — Event Schema Evolution, Failure Modes & When Not to Use EDA

---

### Event Schema Evolution & CloudEvents

- **Question:** How do you version events in a long-lived event-driven system, and what is the CloudEvents specification?
- **Answer:**
  - **The versioning problem:** An event emitted by OrderService v1 is consumed by InventoryService. Six months later, OrderService adds a `discountCode` field to the `OrderPlaced` event. InventoryService must continue to work with both old events (no `discountCode`) and new events (has `discountCode`). Kafka topics retain events for months or years — old events from before the schema change will be replayed.
  - **Versioning strategies:**
    1. **Additive changes only (preferred):** Only add new optional fields. Consumers that don't know about them ignore them. Consumers that need them check for presence. Never rename, remove, or change the type of a field.
    2. **Versioned event types:** `order.placed.v1`, `order.placed.v2`. Each version is a separate event type. Consumers subscribe to the versions they support. Old events stay on the v1 stream; new events go to v2. Producers emit to both during migration; consumers migrate to v2 at their own pace. Operationally complex but cleanly backward compatible.
    3. **Upcasting:** A transformation layer applied when reading old events — converts v1 format to v2 format before handing to the consumer. Used in Event Sourcing to maintain a single event format in application code despite schema changes in the store.
  - **CloudEvents:** A CNCF specification for a standard event envelope format. Defines common metadata fields every event should carry: `id` (unique event ID), `source` (where it originated), `type` (event type — `com.example.order.placed`), `specversion`, `time`, `datacontenttype`, `data` (the actual event payload). Solves: every team inventing their own envelope format, making cross-system event routing and filtering inconsistent. Now a de facto standard for AWS EventBridge events, Knative, Dapr.
- **Example:**

```json
// CloudEvents envelope — standard metadata wrapping any event payload
{
  "specversion": "1.0",
  "id": "550e8400-e29b-41d4-a716-446655440000", // unique event ID (idempotency key)
  "source": "https://order-service.internal/orders",
  "type": "com.company.order.placed.v2", // versioned type
  "time": "2026-03-29T10:00:00Z",
  "datacontenttype": "application/json",
  "data": {
    "orderId": "order-42",
    "userId": "user-99",
    "items": [{ "productId": "prod-1", "qty": 2 }],
    "discountCode": "SPRING25" // new in v2 — absent in v1 events
  }
}
```

```go
// Consumer — handles both v1 and v2 gracefully
type OrderPlacedEvent struct {
    OrderID      string `json:"orderId"`
    UserID       string `json:"userId"`
    Items        []Item `json:"items"`
    DiscountCode string `json:"discountCode,omitempty"` // optional — zero value if v1 event
}

func handleOrderPlaced(event OrderPlacedEvent) {
    // Works for v1 (DiscountCode is "") and v2 (DiscountCode is set)
    if event.DiscountCode != "" {
        applyDiscount(event.OrderID, event.DiscountCode)
    }
    reserveInventory(event.OrderID, event.Items)
}
```

- **Technical Terms to Include:** event versioning, additive schema changes, versioned event types, upcaster, schema registry (Confluent / AWS Glue), CloudEvents, `id`, `source`, `type`, `specversion`, backward compatibility, forward compatibility, consumer migration, Avro/Protobuf schema evolution, breaking change
- **Gotcha:** **Never use a topic name as an event version.** `orders-v2` is not event versioning — it's a new topic that producers write to and consumers read from. The old topic still has v1 events. You now have two topics to monitor, consume from, and manage. Event versioning belongs in the event's `type` field (CloudEvents) or the Protobuf/Avro schema registry, not in the topic/queue name.
- **Follow-Up:** "What is a schema registry and when is it mandatory?" → A schema registry (Confluent Schema Registry, AWS Glue Schema Registry) stores and version-controls event schemas (Avro, Protobuf, JSON Schema). Producers register the schema on first publish; consumers fetch the schema by ID embedded in the message header to deserialise. Mandatory when: (1) Using Avro (binary format, schema required for deserialisation). (2) Enforcing compatibility checks at publish time (block breaking changes before they reach consumers). (3) Multiple language implementations need consistent schema definitions. → They're testing schema governance depth.
- **Conclusion:** Event schema evolution is the long-term maintenance challenge of any event-driven system — additive-only changes plus CloudEvents standard metadata plus a schema registry is the combination that makes a high-volume event ecosystem governable without ceremony for every schema change.

---

### EDA Failure Modes & When NOT to Use EDA

- **Question:** What are the production failure modes specific to event-driven systems that are not present in synchronous architectures, and when is EDA the wrong choice?
- **Answer:**
  - **EDA-specific production failure modes:**
    1. **Consumer lag accumulation (silent backlog):** Consumers fall behind producers. Kafka lag grows. SQS queue depth grows. The system appears healthy (producer side is fine) but consumers are processing hours-old events. Alert: consumer lag > N messages for > M minutes. Without this alert, backlogs can reach millions of events before anyone notices.
    2. **Poison message loop:** A malformed event that always causes the consumer to fail. Consumer processes it, fails, nacks, requeues, gets it back, fails again. If max delivery count is not configured, this loops forever, consuming consumer capacity and blocking queue processing. Mitigation: DLQ with max delivery count.
    3. **Event ordering assumption violation:** A consumer assumes events arrive in order for a given entity. Kafka guarantees order within a partition — but if two updates to the same entity go to different partitions (wrong partition key), the consumer may process them out of order. Always use entity ID as the partition key for ordered entity events.
    4. **Schema mismatch silent failure:** A consumer receives a new event schema it doesn't understand. If it fails silently (logs and moves on), the update is lost. Always fail explicitly on unknown event types — dead-letter the event rather than swallowing the error.
    5. **Clock skew in event-time reasoning:** Events have a `timestamp` field. Distributed systems have clock drift. Two events with the same timestamp may have actually occurred in a different order. Use Kafka offsets (not timestamps) for ordering within a partition. Use logical clocks (sequence numbers, version numbers) for ordering within a domain.
    6. **Dual-write without outbox (data loss):** Service updates DB, broker publish fails, DB not rolled back. State is updated but no event emitted. Downstream forever out of sync. Fix: always use the Outbox pattern.
  - **When NOT to use EDA:**
    1. **Immediate consistency required:** A transfer between bank accounts must be atomic — the debit and credit happen together or not at all. EDA's eventual consistency introduces a window where one account is debited but the other not yet credited. For financial correctness, this requires compensating transactions (Saga) and careful design — not always appropriate.
    2. **Simple CRUD with no fan-out:** A settings update that one service writes and one service reads. Adding a broker adds latency, complexity, and infrastructure for no benefit.
    3. **Query responses (read path):** A client asking "what is the current inventory count?" should not wait for an event round-trip. Queries are synchronous by nature.
    4. **Team maturity is low:** EDA requires understanding of eventual consistency, idempotency, schema evolution, DLQ management. A team without this experience will produce an unreliable system. Sometimes synchronous with good retry and circuit breaker patterns is the right choice while the team matures.
- **Example:**

```
EDA ANTI-PATTERNS:

1. Using EDA for request-response:
   WRONG: Client → emits RequestEvent → waits for ResponseEvent
   This is request/response with extra steps. Use gRPC/REST directly.

2. Using EDA to avoid service dependencies (avoiding the real design problem):
   "We don't want to take a dependency on UserService, so we'll consume
   UserCreated events and replicate the user into our own DB."
   This creates a stale replica and eventual consistency where it's not needed.
   If you always need the latest user data, just call UserService directly.

3. Publishing internal implementation events as domain events:
   WRONG: emit DatabaseRowUpdated { table: "orders", rowId: 42, field: "status", value: "confirmed" }
   RIGHT: emit OrderConfirmed { orderId: 42, confirmedAt: ..., confirmedBy: ... }
   Events should be business facts, not database change records.
   Database implementation events create tight coupling to implementation details.
```

- **Technical Terms to Include:** consumer lag, poison message, DLQ, max delivery count, partition key, ordering guarantee, clock skew, schema mismatch, dual-write, silent failure, event-time vs processing-time, anti-pattern, request-response over EDA, data replication anti-pattern, business event vs technical event
- **Gotcha:** **Consumer lag is invisible without explicit monitoring.** Unlike a synchronous call that returns a 500, a Kafka consumer that is processing 2 hours behind produces no error — everything looks healthy at the API level. Users see stale data. The business experiences delayed processing. The only way to know: monitor `kafka_consumer_group_lag` in Prometheus, alert when lag exceeds a threshold. Without this, EDA gives you silent degradation instead of visible failure.
- **Follow-Up:** "A business stakeholder says 'I want real-time updates in the dashboard.' Does that mean we need event-driven architecture?" → Not necessarily. "Real-time" is often a business requirement for low latency (< 1s), not a technical requirement for event-driven architecture. The latency question is: how does the dashboard currently get its data? If it polls an API every 30s, moving to websockets or SSE can provide sub-second updates without any EDA. EDA is appropriate if the dashboard aggregates from multiple services (fan-in), not just because the business wants fast updates. → They're testing "real-time" requirement decomposition.
- **Conclusion:** EDA failure modes are silent and systemic rather than loud and immediate — consumer lag monitoring, DLQ alerting, and poison message handling are the operational layer that makes an event-driven system's failures visible; and the architectural discipline of not applying EDA to queries, simple CRUD, or immediate-consistency requirements is what keeps the system from becoming an over-engineered liability.

---

## Quick Reference — EDA Pattern Decision Logic

| Pattern                  | Use When                                                                          | Avoid When                                                                    |
| ------------------------ | --------------------------------------------------------------------------------- | ----------------------------------------------------------------------------- |
| **EDA over sync**        | Write propagates to N consumers; consumers can tolerate async                     | Query needs immediate consistent response                                     |
| **Choreography**         | Simple fan-out (notification, analytics, audit); low step count                   | Multi-step business process with compensation; workflow state needed          |
| **Orchestration**        | Multi-step transactions; compensation required; workflow state visibility         | Simple one-way notifications with no failure handling                         |
| **Saga**                 | Distributed transaction across services; no 2PC                                   | Single-service transaction (use local ACID)                                   |
| **CQRS**                 | Read model needs different shape/store than write model; scale read independently | Simple CRUD with no read/write tension                                        |
| **Event Sourcing**       | Audit trail is a first-class requirement; replay required; temporal queries       | GDPR with no crypto-shredding design; simple state management; no replay need |
| **Transactional Outbox** | Always — whenever a service writes to DB and must emit an event                   | Never skip — dual-write without outbox is data loss waiting to happen         |
| **Idempotent consumer**  | Always — any at-least-once consumer                                               | Never skip — duplicates happen in all real systems                            |
| **CloudEvents**          | Cross-system, multi-cloud, or public event bus                                    | Internal single-service events (overkill for local)                           |

## Quick Reference — Common Interview Traps

| Question                                                  | Trap                                 | Correct Answer                                                                                                            |
| --------------------------------------------------------- | ------------------------------------ | ------------------------------------------------------------------------------------------------------------------------- |
| "How do you ensure consistency across microservices?"     | "Use distributed transactions (2PC)" | Saga pattern with compensating transactions                                                                               |
| "What's the guarantee of your event broker?"              | "Exactly-once"                       | At-least-once; exactly-once is available only in specific configs (Kafka transactions) — design consumers for idempotency |
| "How do you handle a failed step in your saga?"           | "Roll back the database"             | Compensating transaction — the DB committed, run a forward semantic undo                                                  |
| "How do you test event-driven systems?"                   | "Integration test the whole flow"    | Unit test the event handler logic; contract test the event schema; integration test with an in-memory broker              |
| "Choreography or orchestration — which is better?"        | Pick one                             | Choreography for side effects; orchestration for core business workflows — use both                                       |
| "Can Event Sourcing and CQRS be used without each other?" | "No, they go together"               | Yes — CQRS is a model separation pattern; Event Sourcing is a state persistence pattern; independent                      |

---

_— End of EventDrivenArchitecture_Interview_QA.md —_
_Coverage: 10 QA units | Patterns: Choreography · Orchestration · Saga · CQRS · Event Sourcing · Outbox · Idempotency · Schema Evolution · Failure Modes_
_Calibrated for: Tech Lead / Architect level — patterns above broker mechanics_
_Brokers covered in: Messaging_Interview_QA.md (Kafka/RabbitMQ) + AWS_Interview_QA.md (SQS/SNS)_
