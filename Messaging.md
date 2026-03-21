<!-- Generated: 2026-03-12 | Apache Kafka 4.0 | RabbitMQ 4.2.4 -->

# Messaging Brokers Interview Preparation — Tech Lead / Architect Level

> Apache Kafka 4.0 · RabbitMQ 4.2.4 | Generated: 2026-03-12 | Career OS

---

## Stack Overview

Kafka and RabbitMQ solve the same broad problem — decoupling producers from consumers — but with fundamentally different architectures and trade-offs. Interviewers at architect level will compare the two, test your decision logic for choosing between them, and probe deep on one or both depending on your resume.

**Key contrast axis:** Kafka is a distributed commit log (messages persist, consumers replay). RabbitMQ is a message broker (messages are consumed and acknowledged, then gone). This distinction drives every architectural decision.

---

# SECTION 1 — Apache Kafka 4.0

---

**L1–L5 Topics identified:**

- What is Kafka / Log-based messaging model
- Core Concepts — Topic / Partition / Offset / Consumer Group
- Producers — partitioning strategy, acknowledgements, idempotence
- Consumers — consumer groups, offset management, rebalancing
- KRaft — ZooKeeper removal (Kafka 4.0)
- Delivery Guarantees — at-most-once / at-least-once / exactly-once
- Kafka Streams & ksqlDB
- Performance & Retention tuning
- When to use Kafka vs RabbitMQ

---

### Core Architecture

#### What is Kafka / The Log Model

- **Question:** What is Apache Kafka, how does the distributed log model work, and why is it architecturally different from traditional message queues?
- **Answer:**
  - Kafka is a distributed event streaming platform. Its core abstraction is an **append-only, partitioned, replicated log**. Producers append events to the end of the log; consumers read from a position (offset) in the log. Unlike a queue, reading an event does not remove it — messages are retained for a configurable period (hours, days, indefinitely).
  - **Distributed log vs queue:** In a traditional queue (RabbitMQ), a message is consumed and acknowledged — it's gone. Multiple consumers receive different messages (competing consumers). In Kafka's log, every message persists. Multiple consumer groups can independently read the same messages — each group has its own offset. This enables:
    (1) Replay — reprocess historical events.
    (2) Multiple independent consumers of the same stream.
    (3) Event sourcing and audit trails.
  - **Core structure:** A **topic** is a named category of events. Topics are split into **partitions** — the unit of parallelism and ordering. Partitions are distributed across **brokers** (servers). Partitions are replicated — one leader, N followers. Producers write to the partition leader; consumers read from the leader (by default in Kafka 4.0).
  - **Kafka 4.0 (2024):** ZooKeeper fully removed — Kafka now uses **KRaft** (Kafka Raft Metadata) for cluster coordination. The controller is now built into Kafka itself. Simpler deployment, faster failover, higher partition counts supported. Also: Queue-style consumption (consumer groups with a new sharing API) as a first-class feature.
- **Example:**

```
Topic: order-events
  Partition 0: [offset 0: order-created] [offset 1: order-paid] [offset 3: order-shipped]
  Partition 1: [offset 0: order-created] [offset 1: order-cancelled]
  Partition 2: [offset 0: order-created] [offset 1: order-paid]

Consumer Group A (order-processor):
  reads Partition 0 at offset 3 (has processed all events)
  reads Partition 1 at offset 0 (only processed first event)

Consumer Group B (audit-logger):
  reads Partition 0 at offset 1 (independent position — its own lag)
  // Both groups consume the SAME events independently
```

```bash
# Kafka CLI — produce and consume
kafka-console-producer.sh --topic order-events --bootstrap-server kafka:9092
kafka-console-consumer.sh --topic order-events --from-beginning --group audit-group
# --from-beginning replays ALL historical events
```

- **Technical Terms to Include:** topic, partition, offset, broker, leader, follower, consumer group, log, append-only, retention, KRaft, ZooKeeper removal, event streaming, commit log
- **Gotcha:** **Ordering is per-partition, not per-topic.** If you need strict ordering across all events of a type, all those events must go to the same partition. Kafka guarantees ordering within a partition — not across partitions. Using a key-based partitioner (events with the same key go to the same partition) is the standard pattern for per-entity ordering (all events for order-id-42 go to the same partition).
- **Follow-Up:** "What changed with the removal of ZooKeeper in Kafka 4.0?" → ZooKeeper was Kafka's external dependency for cluster metadata management (broker registration, leader election, topic config). KRaft (Kafka Raft Metadata) moves this into Kafka itself — the controller quorum uses the Raft consensus protocol. Benefits: fewer moving parts to operate, faster controller failover (sub-second vs several seconds), support for millions of partitions (ZooKeeper was a bottleneck at 200K+), simpler deployment and security model. → They're testing awareness of the most significant recent Kafka architectural change.
- **Conclusion:** Kafka's log model — persistent, replayable, partitioned, independently consumed — is architecturally different from queues in a way that determines the entire system design: it enables event sourcing, audit trails, stream processing, and multi-consumer fan-out that queue-based systems handle awkwardly or not at all.

---

#### Producers — Partitioning, Acks, Idempotence

- **Question:** How does a Kafka producer work, what are the acknowledgement modes, and what is idempotent production?
- **Answer:**
  - A Kafka producer sends records to a topic. Each record has a key (optional), value, and optional headers. The producer determines which partition to send to via the **partitioner**:
    (1) If a key is provided: `hash(key) % num_partitions` — consistent routing (same key → same partition).
    (2) If no key: Kafka 3.x+ uses sticky partitioning (batch to one partition, then round-robin) — maximises batching efficiency.
  - **Acknowledgement modes (`acks`):**
    (1) `acks=0` — fire and forget. No confirmation. Lowest latency, highest data loss risk.
    (2) `acks=1` — leader acknowledgement. The leader writes to disk and confirms. Fast, but if the leader fails before replication, data is lost.
    (3) `acks=all` (or `-1`) — all in-sync replicas (ISR) confirm. Highest durability. Pair with `min.insync.replicas=2` to require at least 2 replicas to confirm.
  - **Idempotent producer (`enable.idempotence=true`):** Assigns a producer ID and sequence number per partition. If the broker receives a duplicate (network retry), it deduplicates using the sequence number. Guarantees exactly-once delivery within a single partition, within a single producer session. Enabled by default in Kafka 3.0+.
  - **Transactions:** For exactly-once across multiple partitions or topics — wrap produce + consumer-offset commit in a transaction. Used by Kafka Streams for stateful processing.
- **Example:**

```java
// Producer config — high durability
Properties props = new Properties();
props.put("bootstrap.servers", "kafka1:9092,kafka2:9092");
props.put("acks", "all");                        // wait for all ISR
props.put("enable.idempotence", "true");          // deduplicate retries
props.put("retries", Integer.MAX_VALUE);           // retry on transient failures
props.put("max.in.flight.requests.per.connection", "5"); // with idempotence, up to 5
props.put("linger.ms", "5");                       // batch for 5ms to improve throughput
props.put("batch.size", "16384");                  // 16KB batch size

KafkaProducer<String, String> producer = new KafkaProducer<>(props);

// Send with key — ensures order per key (e.g. per order-id)
ProducerRecord<String, String> record = new ProducerRecord<>(
    "order-events",
    "order-42",           // key — routes all order-42 events to same partition
    orderJson             // value
);

producer.send(record, (metadata, exception) -> {
    if (exception != null) handleError(exception);
    else log("Written to partition " + metadata.partition() + " offset " + metadata.offset());
});
```

- **Technical Terms to Include:** producer, partitioner, key-based partitioning, `acks`, in-sync replica (ISR), `min.insync.replicas`, idempotent producer, producer ID, sequence number, transaction, `linger.ms`, `batch.size`
- **Gotcha:** `acks=all` alone does NOT guarantee no data loss if `min.insync.replicas=1`. If only the leader is in the ISR, `acks=all` is satisfied by just the leader — same as `acks=1`. Set `min.insync.replicas=2` (or `N/2+1`) on the broker/topic and `acks=all` on the producer together. The combination is the durability guarantee.
- **Follow-Up:** "What is the difference between idempotent production and transactional production?" → Idempotent: deduplicates retries to a single partition from a single producer session. Does not span partitions or topics. Transactional: atomically writes to multiple partitions and commits consumer offsets together — all succeed or all fail. Transactions enable exactly-once stream processing (read-process-write) in Kafka Streams. Idempotence is the prerequisite for transactions. → They're testing exactly-once semantics depth.
- **Conclusion:** Kafka's producer configuration — `acks=all`, `min.insync.replicas=2`, idempotent production — forms the durability layer that determines whether messages are lost in broker failures; getting this wrong means production data loss that is invisible until you notice missing events in your audit log.

---

#### Consumers — Consumer Groups, Offset Management, Rebalancing

- **Question:** How do Kafka consumer groups work, how is offset management handled, and what causes consumer group rebalancing?
- **Answer:**
  - A **consumer group** is a set of consumer instances sharing the same `group.id`. Kafka distributes partitions across consumers in the group — each partition is consumed by exactly one consumer in the group (no two consumers in the same group read the same partition). Scaling: add consumers up to the number of partitions. More consumers than partitions → some are idle.
  - **Offset management:** Each consumer group tracks its offset per partition in the `__consumer_offsets` internal topic. `auto.offset.reset=earliest` (start from beginning) or `latest` (start from now) when no committed offset exists. `enable.auto.commit=true` commits offsets automatically every `auto.commit.interval.ms` — risks processing the same message twice if the consumer dies between processing and commit (at-least-once). Manual commit (`commitSync()`/`commitAsync()`) after processing — explicit control.
  - **Rebalancing:** Triggered when: a consumer joins/leaves the group, a consumer exceeds `session.timeout.ms` without heartbeat, or partitions are added to the topic. During rebalancing: ALL consumers in the group pause — a stop-the-world event. **Cooperative incremental rebalancing** (Kafka 2.4+) minimises disruption by only reassigning partitions that need to move.
  - **`max.poll.interval.ms`:** If a consumer takes longer than this between `poll()` calls, Kafka considers it dead and triggers a rebalance. Long processing times require increasing this or processing async outside the poll loop.
- **Example:**

```java
Properties props = new Properties();
props.put("group.id", "order-processor");
props.put("auto.offset.reset", "earliest");
props.put("enable.auto.commit", "false");  // manual commit — explicit control
props.put("max.poll.records", "100");       // process 100 records per poll
props.put("max.poll.interval.ms", "300000"); // 5 min max between polls

KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);
consumer.subscribe(List.of("order-events"));

while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
    for (ConsumerRecord<String, String> record : records) {
        try {
            processOrder(record.value()); // your business logic
        } catch (RetryableException e) {
            // Don't commit — retry on next poll
            break;
        }
    }
    consumer.commitSync(); // commit only after successful processing
    // commitAsync() for higher throughput — risk: commit failure not retried
}
```

- **Technical Terms to Include:** consumer group, group.id, partition assignment, offset, `__consumer_offsets`, auto commit, manual commit, rebalancing, `session.timeout.ms`, `max.poll.interval.ms`, cooperative incremental rebalancing, `ConsumerRebalanceListener`, lag, consumer group coordinator
- **Gotcha:** Consumer lag is the key production metric for Kafka consumers — `consumer_lag = log_end_offset - consumer_offset`. Lag growing means the consumer is falling behind producers. Monitor via `kafka-consumer-groups.sh --describe` or Prometheus JMX exporter + `kafka_consumer_group_lag` metric. Alert when lag grows continuously for more than N minutes — indicates consumer throughput is insufficient.
- **Follow-Up:** "What is the difference between `commitSync()` and `commitAsync()`?" → `commitSync()` blocks until the commit succeeds or throws after retries — reliable but slower. `commitAsync()` doesn't block — fires and forgets. If it fails, there's no automatic retry (a retry could commit a stale offset after a newer one was already committed). Use `commitSync()` for shutdown sequences. Use `commitAsync()` for in-loop commits with a retry callback. Use both: `commitAsync()` in the poll loop, `commitSync()` in the finally block. → They're testing offset commit strategy depth.
- **Conclusion:** Consumer group mechanics — partition assignment, offset tracking, and rebalancing triggers — determine the fault tolerance and throughput characteristics of a Kafka-based system; understanding that rebalancing is a stop-the-world event and that cooperative incremental rebalancing minimises it is the distinction between a default setup and a production-tuned one.

---

#### Delivery Guarantees & Kafka vs RabbitMQ Decision

- **Question:** What are Kafka's three delivery semantics, and how do you decide between Kafka and RabbitMQ for a given use case?
- **Answer:**
  - **At-most-once:** Producer `acks=0`, no retries. Consumer commits before processing. Message may be lost; never duplicated. Use when loss is acceptable and throughput is paramount (metrics sampling, non-critical telemetry).
  - **At-least-once:** Producer retries on failure. Consumer commits after processing. Message is never lost but may be delivered multiple times (on consumer failure between process and commit). Requires idempotent consumers. The most common production default.
  - **Exactly-once:** Idempotent producer + transactional producer + `read_committed` consumer isolation level. Never lost, never duplicated — within a Kafka cluster. Significant overhead. Use for financial transactions, inventory updates, anything where duplicates cause business errors.
  - **Kafka vs RabbitMQ decision matrix:**

| Decision Factor                | Choose Kafka                  | Choose RabbitMQ                            |
| ------------------------------ | ----------------------------- | ------------------------------------------ |
| Message retention & replay     | ✅ Log persists               | ❌ Messages consumed and gone              |
| Multiple independent consumers | ✅ Consumer groups            | ❌ Each message to one consumer            |
| Throughput requirements        | ✅ Millions/sec               | ✅ Thousands–hundreds of thousands/sec     |
| Message routing complexity     | ❌ Topic + partition only     | ✅ Exchanges (direct/fanout/topic/headers) |
| Message priority queues        | ❌ No native support          | ✅ Native priority queue                   |
| Complex routing / dead-letter  | ❌ Manual DLQ topics          | ✅ Native DLX, TTL, per-queue policies     |
| Stream processing              | ✅ Kafka Streams, ksqlDB      | ❌ Not designed for it                     |
| Event sourcing                 | ✅ Ideal                      | ❌ Not suitable                            |
| Simple task queues             | ❌ Overkill                   | ✅ Simpler to operate                      |
| Per-message TTL & expiry       | ❌ Topic-level retention only | ✅ Per-message TTL                         |

- **Technical Terms to Include:** at-most-once, at-least-once, exactly-once, idempotent producer, transactional API, `read_committed`, consumer isolation, DLQ, Kafka Streams, event sourcing, log compaction
- **Gotcha:** Exactly-once semantics in Kafka means exactly-once **within Kafka** — not exactly-once end-to-end with external systems (databases, APIs). If you process a Kafka message, write to a database, and the database write succeeds but Kafka commit fails → the message is redelivered and processed again → duplicate database write. Exactly-once requires the external system to participate in the transaction (idempotent writes, transactional outbox pattern).
- **Follow-Up:** "What is log compaction in Kafka and when do you use it?" → Log compaction retains only the **last message per key** — earlier messages with the same key are garbage collected. Turns Kafka from an event stream into a compacted snapshot: the latest state of each entity. Use for: change data capture (CDC) streams, configuration topics (latest config for each service), event-sourced read models. A compacted topic with key=entity-id gives you the current state of every entity efficiently. → They're testing log compaction architectural use case awareness.
- **Conclusion:** The Kafka vs RabbitMQ decision is architectural, not preferential — Kafka for event streaming, replay, fan-out, and stream processing; RabbitMQ for complex routing, priority queues, per-message policies, and simple task queue patterns; misapplying Kafka as a task queue wastes its strengths while exposing its operational complexity.

---

# SECTION 2 — RabbitMQ 4.2.4

---

**L1–L4 Topics identified:**

- What is RabbitMQ / AMQP model
- Core Concepts — Exchange / Queue / Binding / Routing Key
- Exchange Types — direct / fanout / topic / headers
- Acknowledgements, Prefetch, Dead Letter Exchange
- Reliability — durable queues, persistent messages, publisher confirms
- Quorum Queues (RabbitMQ 4.x default)
- When to use RabbitMQ

---

### Core Architecture

#### What is RabbitMQ / AMQP Model

- **Question:** What is RabbitMQ, how does the AMQP routing model work, and why is it more flexible for routing than Kafka?
- **Answer:**
  - RabbitMQ is a message broker implementing the AMQP (Advanced Message Queuing Protocol) model. Producers send messages to **exchanges**; exchanges route messages to **queues** based on **binding keys** and **routing keys**; consumers subscribe to queues.
  - **AMQP routing model:** The exchange is the routing intelligence. The producer specifies a **routing key** on the message. The exchange compares the routing key to its **bindings** (queue subscriptions with binding keys) and delivers to matching queues. The producer has no knowledge of queues — it only knows the exchange.
  - **This enables:**
    (1) Multiple queues receiving copies of the same message (fanout).
    (2) Selective routing based on routing key patterns (topic exchange).
    (3) Content-based routing via message headers.
    (4) Dynamic reconfiguration of routing without changing producers or consumers.
  - **RabbitMQ 4.x:** Quorum queues are now the default queue type (replacing classic mirrored queues). Khepri metadata store replaces Mnesia for improved consistency. Native MQTT 5.0 support. Streams (Kafka-like append-only log) available as a queue type for replay use cases.
- **Example:**

```
AMQP Model:

Producer → [Exchange: "order-exchange" (topic)] → Binding: "order.#" → [Queue: all-orders]
                                                 → Binding: "order.paid" → [Queue: payment-processor]
                                                 → Binding: "order.*.uk"  → [Queue: uk-orders]

Message with routing_key="order.paid.uk":
  Matches "order.#"    → delivered to all-orders ✅
  Matches "order.paid" → NOT matched (must match exactly) ❌
  Matches "order.*.uk" → delivered to uk-orders ✅
  Result: message delivered to all-orders AND uk-orders
```

```python
import pika

connection = pika.BlockingConnection(pika.ConnectionParameters('rabbitmq'))
channel = connection.channel()

# Declare exchange
channel.exchange_declare(exchange='order-exchange', exchange_type='topic')

# Publish — producer sends to exchange with routing key
channel.basic_publish(
    exchange='order-exchange',
    routing_key='order.paid.uk',
    body=json.dumps(order),
    properties=pika.BasicProperties(
        delivery_mode=2,  # persistent — survive broker restart
        content_type='application/json'
    )
)
```

- **Technical Terms to Include:** AMQP, exchange, queue, binding, routing key, binding key, direct exchange, fanout exchange, topic exchange, headers exchange, default exchange, virtual host (vhost), message properties, delivery mode
- **Gotcha:** Messages sent to a **direct exchange** with a routing key that matches no binding are silently **dropped**. There is no error — the producer doesn't know the message was lost. Use the `mandatory` flag on publish — if no queue is bound that matches, the broker returns the message to the producer via the `return` callback. Or bind a fallback queue. Silent message loss from unbound routing keys is one of the most common RabbitMQ production bugs.
- **Follow-Up:** "What is the default exchange in RabbitMQ?" → Every queue is automatically bound to the default exchange with a binding key equal to the queue name. Publishing to the empty exchange (`""`) with routing key `"my-queue"` delivers directly to `my-queue`. This is how most tutorial examples work — they skip declaring an exchange entirely and publish directly to a queue by name. It's a convenience, not a routing pattern to use in production. → They're testing AMQP model depth.
- **Conclusion:** RabbitMQ's exchange-based routing model gives producers and consumers complete decoupling with sophisticated routing logic — the producer knows only the exchange and routing key; the binding configuration determines delivery, enabling runtime routing changes without code deployment.

---

#### Exchange Types & Dead Letter Exchange

- **Question:** Explain the four RabbitMQ exchange types and how the Dead Letter Exchange pattern enables reliable error handling.
- **Answer:**
  - **Direct exchange:** Routes messages to queues whose binding key exactly matches the routing key. One-to-one or one-to-many (multiple queues with same binding key). Use for: task distribution where the destination is known.
  - **Fanout exchange:** Routes to all bound queues — ignores routing key. Use for: broadcast (push notification to all consumers, cache invalidation, audit logging). Every bound queue receives every message.
  - **Topic exchange:** Routes based on routing key pattern matching. `*` matches one word; `#` matches zero or more words. `order.paid` matches `order.*` and `order.#` but not `*.paid`. Use for: event categorisation, selective subscription by category, multi-tenant routing.
  - **Headers exchange:** Routes based on message header key-value pairs (not routing key). Flexible but rare — use when routing logic is based on message content rather than a routing key string.
  - **Dead Letter Exchange (DLX):** When a message cannot be delivered (queue full, message TTL expired, consumer `nack` with `requeue=false`), it is forwarded to the configured DLX with an optional routing key. The DLX routes to a Dead Letter Queue (DLQ). Engineers can inspect, replay, or alert on dead-lettered messages. Essential for production reliability — without DLX, undeliverable messages are silently dropped.
- **Example:**

```python
# Topic exchange — order routing by region and type
channel.exchange_declare(exchange='orders', exchange_type='topic', durable=True)

# Queues and bindings
channel.queue_declare(queue='all-orders', durable=True,
    arguments={'x-dead-letter-exchange': 'orders-dlx',  # DLX config
               'x-message-ttl': 300000})                # 5 min TTL
channel.queue_bind('all-orders', 'orders', routing_key='order.#')

channel.queue_declare(queue='uk-payments', durable=True,
    arguments={'x-dead-letter-exchange': 'orders-dlx'})
channel.queue_bind('uk-payments', 'orders', routing_key='order.paid.uk')

# DLX setup — catches all dead letters
channel.exchange_declare(exchange='orders-dlx', exchange_type='fanout', durable=True)
channel.queue_declare(queue='dead-letters', durable=True)
channel.queue_bind('dead-letters', 'orders-dlx')

# Consumer — reject without requeue → goes to DLX
def on_message(ch, method, properties, body):
    try:
        process(body)
        ch.basic_ack(delivery_tag=method.delivery_tag)
    except NonRetryableError:
        ch.basic_nack(delivery_tag=method.delivery_tag, requeue=False)
        # → message sent to orders-dlx → dead-letters queue
```

- **Technical Terms to Include:** direct exchange, fanout exchange, topic exchange, headers exchange, Dead Letter Exchange (DLX), Dead Letter Queue (DLQ), `x-dead-letter-exchange`, `x-message-ttl`, `nack`, `requeue=false`, routing key pattern, durable exchange
- **Gotcha:** A Dead Letter Exchange must be bound to a queue — declaring the DLX without a bound queue causes dead-lettered messages to be silently dropped (same as having no DLX). Always declare the DLX exchange + DLQ queue + binding together. Monitor the DLQ depth as a first-class metric — growing DLQ means your consumers are failing.
- **Follow-Up:** "How do you implement a retry mechanism with exponential backoff in RabbitMQ?" → Use multiple DLQs with increasing TTLs: DLQ-1 (30s TTL), DLQ-2 (5m TTL), DLQ-3 (1h TTL). Each DLQ has the original exchange as its own DLX. When a message expires in DLQ-1, it goes back to the original exchange → requeued → if it fails again → DLQ-2. After max retries, move to a permanent DLQ for human inspection. Each retry attempt decrements a header counter. → They're testing production RabbitMQ retry architecture depth.
- **Conclusion:** RabbitMQ's exchange type flexibility enables routing patterns that Kafka cannot express natively — fanout for broadcast, topic for selective subscription, DLX for graceful error handling — making it the more operationally flexible choice for complex routing requirements at the cost of a more complex mental model.

---

#### Reliability — Quorum Queues, Durability, Publisher Confirms

- **Question:** What makes a RabbitMQ setup reliable, what are quorum queues, and how do publisher confirms work?
- **Answer:**
  - **Durability trilogy:** (1) **Durable exchange** — survives broker restart. (2) **Durable queue** — survives restart (declared with `durable=True`). (3) **Persistent message** — `delivery_mode=2` — written to disk, survives restart. All three must be set for true durability — any one missing and messages are lost on restart.
  - **Quorum queues (default in RabbitMQ 4.x):** Replicated queues using the Raft consensus protocol. A quorum (majority) of replicas must confirm a write before it's considered durable. Replace the older classic mirrored queues (which had split-brain risks). Support: per-message TTL, DLX, consumer priorities. Require an odd number of nodes (3, 5) for quorum. Prefer quorum queues for all new deployments.
  - **Publisher confirms:** The broker sends an `ack` to the producer after the message is durably stored (for quorum queues: after Raft consensus). Enables the producer to know messages are safe. Without confirms: fire-and-forget — messages may be lost between producer and broker. With confirms: at-least-once between producer and broker.
  - **Consumer acknowledgements:** `basic_ack` — message processed, removed from queue. `basic_nack` with `requeue=True` — requeue immediately (risk: poison message loop). `basic_nack` with `requeue=False` — dead-letter if DLX configured, or drop.
- **Example:**

```python
# Full durability setup
channel.exchange_declare(exchange='tasks', exchange_type='direct', durable=True)

channel.queue_declare(
    queue='task-queue',
    durable=True,                    # queue survives restart
    arguments={
        'x-queue-type': 'quorum',    # Raft-based replication (RabbitMQ 4.x default)
        'x-dead-letter-exchange': 'tasks-dlx',
        'x-delivery-limit': 3,       # quorum queue: max 3 deliveries before DLQ
    }
)

# Publisher confirms — know when message is safe
channel.confirm_delivery()
try:
    channel.basic_publish(
        exchange='tasks',
        routing_key='task-queue',
        body=task_json,
        properties=pika.BasicProperties(delivery_mode=2)  # persistent
    )
    # Raises UnroutableError or NackError if not confirmed
except pika.exceptions.NackError:
    handle_not_confirmed()  # retry or log

# Consumer — manual ack, prefetch to avoid overwhelming slow consumer
channel.basic_qos(prefetch_count=10)  # receive max 10 unacked messages
def process(ch, method, properties, body):
    do_work(body)
    ch.basic_ack(delivery_tag=method.delivery_tag)

channel.basic_consume(queue='task-queue', on_message_callback=process)
```

- **Technical Terms to Include:** durable exchange, durable queue, persistent message, quorum queue, classic mirrored queue, Raft consensus, publisher confirms, `basic_ack`, `basic_nack`, `prefetch_count`, `x-delivery-limit`, at-least-once, poison message
- **Gotcha:** **Prefetch count (`basic_qos`)** is critical for consumer performance. Without it, RabbitMQ sends all available messages to the consumer at once — a slow consumer can be overwhelmed with messages it can't process, holding all of them unacked. This blocks other consumers from receiving work. Set `prefetch_count` to match the consumer's concurrency capacity (e.g., `10` for a consumer with 10 worker threads). This is the most common RabbitMQ consumer performance issue.
- **Follow-Up:** "What is the `x-delivery-limit` on quorum queues?" → A quorum queue–specific feature that limits how many times a message can be delivered (attempted). After exceeding the limit, the message is dead-lettered to the DLX. This is the quorum queue equivalent of a maximum retry count — prevents poison messages from looping forever. Classic queues had no built-in delivery limit; quorum queues make this a first-class configuration. → They're testing RabbitMQ 4.x quorum queue feature awareness.
- **Conclusion:** RabbitMQ reliability requires the durability trilogy (durable exchange + durable queue + persistent message) plus quorum queues for replication, publisher confirms for producer assurance, and prefetch count control for consumer stability — each layer independently necessary, none sufficient alone.

---

_— End of Messaging_Interview_QA.md —_
_Coverage: Kafka (L1–L5, 4 QA units) | RabbitMQ (L1–L4, 3 QA units)_
_Calibrated for: Tech Lead / Architect level_
