# Week 19 — Event-Driven Architecture & Kafka

**Week of:** October 12, 2026
**Estimated study time:** ~2 hours
**Tags:** `kafka` `event-driven` `architecture` `messaging`

---

## Overview

Event-driven architecture (EDA) is a design paradigm where system components communicate by producing and consuming events rather than by calling each other directly. Instead of Service A asking Service B "what is the current state of X?", Service A broadcasts "X changed" and any interested party reacts. This inversion of control leads to systems that are naturally decoupled, independently scalable, and resilient to partial failures — qualities that become critical as distributed systems grow.

Request-driven (synchronous) systems are simple and predictable: you call an endpoint, you get a response, you move on. The cost is tight coupling: the caller must know the callee's address, the callee must be available right now, and failure propagates instantly back to the caller. Most web APIs and microservice-to-microservice HTTP calls are request-driven. AESF's Apex triggers calling `aesf-py-middleware` REST endpoints are a textbook example.

Queue-table-based async patterns sit in between. AESF today stores work items in PostgreSQL tables (essentially a homegrown message queue), which decouples the producer from the consumer in time — the Apex trigger writes a row, the ETL polls and processes it later. This is battle-tested and requires no additional infrastructure. The trade-off is that it puts queue-management complexity (polling, locking, dead-letter handling, ordering, fan-out) onto application code that was never designed for it, and PostgreSQL's MVCC engine pays a real cost for high-frequency updates to the same rows.

Apache Kafka occupies the far end of the spectrum: a distributed commit log purpose-built for high-throughput, ordered, durable, replayable event streams. This week you will learn Kafka's internals deeply enough to reason about whether — and where — it would improve AESF, how exactly-once semantics work, and how event sourcing plus CQRS compose on top of Kafka to fundamentally change how you model state.

---

## 1. Event-Driven vs. Request-Driven: The Coupling Spectrum

The core question is: who knows about whom?

```
Request-Driven (tight coupling)
────────────────────────────────
  Apex Trigger ──HTTP POST──► aesf-py-middleware /sync
       ▲                              │
       └──────── 200 OK / 500 ────────┘
  Caller must know URL, must retry on failure, blocked until response.

Queue-Table Async (loose in time, tight in schema)
───────────────────────────────────────────────────
  Apex Trigger ──INSERT──► sync_queue (PostgreSQL)
                                  │
                          ETL polls SELECT ... FOR UPDATE SKIP LOCKED
                                  │
                          Processes, UPDATE status='done'

Event-Driven (loose in time AND in coupling)
─────────────────────────────────────────────
  Apex Change Event ──produce──► Kafka topic: aesf.account.changed
                                        │
                              ┌─────────┴──────────┐
                         BDE Sync Consumer    BQ Sync Consumer
```

In the queue-table model, the producer (middleware API) and consumer (ETL) share knowledge of the same PostgreSQL schema. Add a third consumer (say, a BigQuery sink) and you must either add another queue table or modify existing logic. In Kafka, any number of independent consumer groups can read the same topic with zero producer-side changes.

**Common Mistake:** Treating event-driven as "just async." The deeper value is not latency — it is the removal of producer-consumer knowledge. If your "event-driven" system still has one producer writing exclusively for one consumer, you have async RPC, not EDA.

---

## 2. Kafka Internals: Brokers, Topics, and Partitions

A Kafka **cluster** is a group of **brokers** — JVM processes that store and serve messages. A **topic** is a logical stream of records, physically split into **partitions**. Each partition is an append-only, ordered log stored on disk.

```
Topic: aesf.opportunity.events   (4 partitions)

Broker 1                   Broker 2                   Broker 3
┌──────────────┐           ┌──────────────┐           ┌──────────────┐
│ Partition 0  │ (leader)  │ Partition 1  │ (leader)  │ Partition 2  │ (leader)
│ offset 0─►n  │           │ offset 0─►n  │           │ offset 0─►n  │
│              │           │              │           │              │
│ Partition 3  │ (replica) │ Partition 0  │ (replica) │ Partition 1  │ (replica)
└──────────────┘           └──────────────┘           └──────────────┘
```

**Partitioning determines parallelism and ordering.** Records with the same key are always routed to the same partition (consistent hash of key mod num_partitions), which guarantees ordering per key. If you use `account_id` as the Kafka message key, all events for a given account arrive in order at a single partition — critical for bidirectional sync correctness.

```python
# confluent-kafka producer — keyed by account_id for ordering guarantee
from confluent_kafka import Producer
import json

producer = Producer({
    'bootstrap.servers': 'kafka-broker-1:9092,kafka-broker-2:9092',
    'acks': 'all',                  # wait for all ISR replicas
    'enable.idempotence': True,     # exactly-once at producer level
    'compression.type': 'lz4',
})

def publish_account_event(account_id: str, payload: dict):
    producer.produce(
        topic='aesf.account.events',
        key=account_id.encode('utf-8'),   # same account → same partition
        value=json.dumps(payload).encode('utf-8'),
        callback=delivery_report,
    )
    producer.poll(0)   # trigger async callbacks

def delivery_report(err, msg):
    if err:
        print(f"Delivery failed for {msg.key()}: {err}")
    else:
        print(f"Delivered to {msg.topic()} [{msg.partition()}] @ offset {msg.offset()}")
```

**Replication factor** (typically 3 in production) means each partition has one **leader** and N-1 **followers** in the **In-Sync Replica (ISR)** set. A write is acknowledged only after all ISR replicas confirm it when `acks=all`. This is where Kafka's durability guarantee lives.

**Common Mistake:** Setting `replication.factor=1` in development and forgetting to change it before production. A single broker failure with RF=1 means permanent data loss for that partition.

---

## 3. Consumer Groups and Offsets

A **consumer group** is a set of consumers that collectively read a topic. Kafka assigns each partition to exactly one consumer in the group — this is the **partition assignment**. Adding consumers to a group scales throughput; adding consumer groups adds independent readers without any producer changes.

```
Topic: aesf.account.events (3 partitions)

Consumer Group: bde-sync-workers (3 consumers)
  Consumer-0 ─► Partition 0
  Consumer-1 ─► Partition 1
  Consumer-2 ─► Partition 2

Consumer Group: bq-sink-workers (1 consumer)
  Consumer-0 ─► Partition 0, 1, 2  (one consumer handles all)

Both groups read independently; producer is unaware.
```

The **offset** is a monotonically increasing integer per partition that marks a consumer's read position. Unlike a traditional message queue that deletes consumed messages, Kafka retains messages for a configurable retention period (default 7 days). Consumers commit their offsets back to Kafka (stored in `__consumer_offsets` internal topic), so they can resume after restart without message loss.

```python
# confluent-kafka consumer — manual offset commit for control
from confluent_kafka import Consumer, KafkaException

consumer = Consumer({
    'bootstrap.servers': 'kafka-broker-1:9092',
    'group.id': 'bde-sync-workers',
    'auto.offset.reset': 'earliest',       # start from beginning if no committed offset
    'enable.auto.commit': False,           # we commit manually after processing
    'max.poll.interval.ms': 300000,        # 5 min: time allowed between polls
})

consumer.subscribe(['aesf.account.events'])

try:
    while True:
        msg = consumer.poll(timeout=1.0)
        if msg is None:
            continue
        if msg.error():
            raise KafkaException(msg.error())

        payload = json.loads(msg.value().decode('utf-8'))
        process_account_sync(payload)          # your business logic

        # Only commit AFTER successful processing
        consumer.commit(message=msg, asynchronous=False)
finally:
    consumer.close()
```

Contrast with the AESF queue-table pattern:

```sql
-- PostgreSQL queue-table equivalent: manual "offset" via status column
-- Producer inserts:
INSERT INTO sync_queue (object_type, record_id, payload, status, created_at)
VALUES ('Account', '001...', '{"Name": "Acme"}', 'pending', NOW());

-- Consumer claims work (advisory lock prevents double-processing):
SELECT id, record_id, payload
FROM sync_queue
WHERE status = 'pending' AND object_type = 'Account'
ORDER BY created_at
LIMIT 10
FOR UPDATE SKIP LOCKED;

-- After processing:
UPDATE sync_queue SET status = 'done', processed_at = NOW() WHERE id = $1;
```

The SQL approach requires a lock, a read, a write, and an index scan per message. Under heavy load this creates lock contention and table bloat (dead rows from frequent UPDATEs). Kafka has no equivalent contention — consumers read from an offset without affecting the log.

**Common Mistake:** Using `enable.auto.commit=True` with slow consumers. Auto-commit fires on a timer regardless of whether processing succeeded. If your process crashes between auto-commit and successful processing, messages are silently skipped.

---

## 4. Exactly-Once Semantics

Distributed systems offer three delivery guarantees:

| Guarantee | Description | Risk |
|-----------|-------------|------|
| At-most-once | Never duplicate, may lose | Message loss on crash |
| At-least-once | Never lose, may duplicate | Idempotent consumers required |
| Exactly-once | No loss, no duplicate | Highest complexity/cost |

Kafka achieves exactly-once via two mechanisms layered together:

**Idempotent Producer** (`enable.idempotence=True`): The broker assigns each producer a PID and sequence number. Duplicate produces from the same producer (e.g., after a network retry) are detected and deduplicated at the broker.

**Transactional API**: Groups produce + consumer offset commit into an atomic transaction across partitions. Either both succeed or both are rolled back. This enables the "read-process-write" pattern without duplicates.

```python
# Transactional producer for exactly-once read-process-write
producer = Producer({
    'bootstrap.servers': 'kafka-broker-1:9092',
    'enable.idempotence': True,
    'transactional.id': 'bde-sync-transactional-producer-1',  # unique per instance
})

producer.init_transactions()

# In your processing loop:
def process_with_exactly_once(consumer, msg):
    producer.begin_transaction()
    try:
        payload = json.loads(msg.value())
        transformed = transform_for_epic(payload)

        producer.produce(
            topic='aesf.epic.sync.commands',
            key=msg.key(),
            value=json.dumps(transformed).encode(),
        )

        # Commit the input offset atomically with the output produce
        producer.send_offsets_to_transaction(
            {TopicPartition(msg.topic(), msg.partition()): msg.offset() + 1},
            consumer.consumer_group_metadata(),
        )
        producer.commit_transaction()
    except Exception as e:
        producer.abort_transaction()
        raise
```

For AESF's Epic sync, exactly-once matters when an Epic API call is non-idempotent (e.g., creating a contact creates a duplicate if called twice). The current queue-table approach handles this via the `status` column plus application-level duplicate checks — functionally equivalent to at-least-once with idempotent consumers.

**Common Mistake:** Believing exactly-once applies end-to-end including your downstream API calls. Kafka's exactly-once guarantee covers Kafka-to-Kafka operations. Calling an external API (like Epic's REST endpoints) still requires idempotency keys or duplicate-check logic at the application layer.

---

## 5. Event Sourcing

**Event sourcing** is a persistence pattern where the system stores a sequence of immutable events — rather than current state — as the system of record. Current state is derived by replaying events.

```
Traditional (state-based):
  accounts table row:  {id: "001", name: "Acme", status: "active"}
  → One UPDATE overwrites history forever.

Event-sourced:
  account_events stream:
    {seq: 1, type: "AccountCreated",  data: {name: "Acme Corp"}}
    {seq: 2, type: "AccountRenamed",  data: {old: "Acme Corp", new: "Acme Inc"}}
    {seq: 3, type: "AccountActivated",data: {activated_by: "user@..."}}
  → Current state = fold(events, initial_state)
  → History is first-class, not reconstructed from audit logs.
```

Kafka is a natural event store: its append-only, ordered, durable log is semantically identical to an event stream. Setting `retention.ms=-1` (infinite retention) on a topic turns it into a permanent event log.

```python
# Reconstructing account state from event stream
from dataclasses import dataclass, field
from typing import List

@dataclass
class AccountState:
    id: str = ""
    name: str = ""
    status: str = "inactive"
    epic_client_id: str = ""

def apply_event(state: AccountState, event: dict) -> AccountState:
    etype = event["type"]
    data = event["data"]

    if etype == "AccountCreated":
        state.id = data["id"]
        state.name = data["name"]
    elif etype == "AccountRenamed":
        state.name = data["new_name"]
    elif etype == "EpicClientLinked":
        state.epic_client_id = data["epic_client_id"]
    elif etype == "AccountDeactivated":
        state.status = "inactive"
    return state

def rebuild_account(events: List[dict]) -> AccountState:
    state = AccountState()
    for event in events:
        state = apply_event(state, event)
    return state
```

For AESF, event sourcing would mean that every Salesforce change event (Account updated, Opportunity created) becomes an immutable record. The ETL never reads "current state" from Salesforce — it processes events. Debugging sync failures becomes trivial: replay the event stream from any point.

**Common Mistake:** Event sourcing every domain object by default. It adds complexity (snapshots for performance, schema evolution of old events, eventual consistency of read models). Reserve it for domains where audit history and replay are genuinely valuable, such as financial records or compliance-critical sync operations.

---

## 6. CQRS — Command Query Responsibility Segregation

CQRS separates write operations (**commands**) from read operations (**queries**) into independent models. Combined with event sourcing, this is the **CQRS + ES** pattern:

```
                     ┌─────────────┐
  HTTP POST ─────────► Command     │
  "Update Account"   │ Handler     │──── produces ────► Kafka topic
                     └─────────────┘                   aesf.account.events
                                                              │
                                              ┌───────────────┤
                                              ▼               ▼
                                    ┌──────────────┐  ┌───────────────┐
                                    │ BDE Read     │  │  BQ Read      │
                                    │ Model (PG)   │  │  Model (BQ)   │
                                    └──────────────┘  └───────────────┘
                                           │
  HTTP GET ──────────────────────────────► Query Handler (reads from PG read model)
  "Get Account"                            (fast, denormalized, optimized for reads)
```

The write side (command) normalizes for consistency. The read side materializes purpose-built projections. In AESF terms: the Apex trigger produces a `AccountChanged` command, middleware validates and publishes the event, and two separate consumers build their own read models — one for Epic BDE sync state, one for BigQuery analytics. Neither consumer blocks the other or requires schema agreement beyond the event contract.

**Common Mistake:** Implementing CQRS without event sourcing and ending up with two databases that drift out of sync. CQRS without ES requires explicit synchronization logic. With ES, the event log is the single source of truth and read models are always rebuildable.

---

## 7. Dead Letter Queues

A **Dead Letter Queue (DLQ)** is a destination for messages that cannot be processed successfully after a configured number of retries. Without a DLQ, a persistently failing message blocks the consumer (poison pill) or is silently dropped.

```
Normal Flow:
  aesf.account.events ──► Consumer ──► Epic API call ──► success ──► commit offset

Retry + DLQ Flow:
  aesf.account.events ──► Consumer ──► Epic API call ──► 500 error
                                               │
                                    retry (max_retries=3, backoff=exponential)
                                               │
                                    still failing after 3 attempts
                                               │
                                    ──► aesf.account.events.DLQ
                                          (original msg + error metadata + retry count)
                                          │
                                    ──► alert/monitoring
                                    ──► manual reprocess or discard
```

```python
# DLQ pattern with confluent-kafka
import json
from datetime import datetime, timezone

DLQ_TOPIC = 'aesf.account.events.DLQ'
MAX_RETRIES = 3

def process_with_dlq(consumer, producer, msg):
    payload = json.loads(msg.value())
    retry_count = payload.get('_retry_count', 0)

    try:
        sync_to_epic(payload)
        consumer.commit(message=msg, asynchronous=False)

    except EpicTransientError as e:
        # Retryable: re-enqueue with incremented retry count
        if retry_count < MAX_RETRIES:
            payload['_retry_count'] = retry_count + 1
            payload['_last_error'] = str(e)
            producer.produce(
                topic=msg.topic(),     # back to same topic
                key=msg.key(),
                value=json.dumps(payload).encode(),
            )
        else:
            # Exhausted retries → DLQ
            send_to_dlq(producer, msg, payload, str(e))
        consumer.commit(message=msg, asynchronous=False)

    except EpicPermanentError as e:
        # Non-retryable: straight to DLQ
        send_to_dlq(producer, msg, payload, str(e))
        consumer.commit(message=msg, asynchronous=False)

def send_to_dlq(producer, original_msg, payload, error: str):
    dlq_payload = {
        'original_topic': original_msg.topic(),
        'original_partition': original_msg.partition(),
        'original_offset': original_msg.offset(),
        'original_payload': payload,
        'error': error,
        'failed_at': datetime.now(timezone.utc).isoformat(),
    }
    producer.produce(
        topic=DLQ_TOPIC,
        key=original_msg.key(),
        value=json.dumps(dlq_payload).encode(),
    )
    producer.flush()
```

**AESF Comparison:** The current queue-table pattern handles this via a `status='failed'` column and `retry_count` field. Failed rows sit in the same `sync_queue` table. A DLQ in Kafka gives you separation of concerns: the main topic stays clean, failed messages have their own durable log with full metadata, and you can build a monitoring consumer on the DLQ topic without touching the happy path.

```sql
-- Current AESF DLQ equivalent in PostgreSQL
UPDATE sync_queue
SET status = 'dead',
    error_message = $1,
    retry_count = retry_count + 1,
    last_attempted_at = NOW()
WHERE id = $2;

-- "DLQ consumer": a cron job that SELECTs status='dead' for manual review
SELECT * FROM sync_queue WHERE status = 'dead' ORDER BY last_attempted_at DESC;
```

**Common Mistake:** Not monitoring the DLQ. A DLQ without alerts is a silent graveyard. Every message in a DLQ represents a failed business operation. In AESF's context, a DLQ message means an Epic sync did not happen — which leads to data drift between Salesforce and Epic BDE.

---

## 8. AESF Migration Analysis: PostgreSQL Queue Tables vs. Kafka

This is the practical question: should AESF migrate from queue tables to Kafka?

**Current AESF queue-table characteristics:**

| Property | PostgreSQL Queue Table | Kafka |
|----------|----------------------|-------|
| Throughput | ~1,000–5,000 msg/s (limited by MVCC write amplification) | 1M+ msg/s per broker |
| Ordering | Per-query ORDER BY (expensive under load) | Per-partition, guaranteed |
| Fan-out | Requires multiple tables or JOIN logic | Inherent: multiple consumer groups |
| Replay | Lost on DELETE; requires archive table | Native: seek to any offset |
| Schema | PostgreSQL schema, tightly coupled | Schema Registry (Avro/Protobuf) |
| Operational complexity | Zero (already running PostgreSQL) | High: ZooKeeper/KRaft, brokers, monitoring |
| Exactly-once | Application-level (SELECT FOR UPDATE + status) | Native transactional API |
| Message retention | Until deleted by ETL | Configurable retention period |

**When to migrate — signals to watch for:**

```
Migrate if:
  ✓ sync_queue table has > 100k rows regularly (autovacuum pressure)
  ✓ ETL poll cycle > 10s due to lock contention under load
  ✓ Need > 2 independent consumers of the same events (fan-out)
  ✓ Audit/replay of all historical sync events is required
  ✓ Team has Kafka operational expertise

Stay with queue tables if:
  ✓ Current throughput is adequate (< 5,000 sync events/day)
  ✓ Team is small and operational overhead matters
  ✓ Deployment constraints prevent adding Kafka to GKE cluster
  ✓ The "exactly-one consumer" model is permanent
```

**Hybrid migration path** (minimal risk):

```
Phase 1 (now):     Apex → PostgreSQL queue table → ETL (current)
Phase 2 (bridge):  Apex → PostgreSQL queue table → CDC (Debezium) → Kafka topic
                                                                          │
                                                               New consumers read Kafka
                                                               ETL still reads queue table
Phase 3 (cutover): Remove queue table; ETL reads Kafka directly
```

Debezium is a CDC (Change Data Capture) tool that tails the PostgreSQL WAL and produces Kafka events for every table change — effectively turning your existing queue table into a Kafka producer without changing any application code.

**Common Mistake:** Migrating to Kafka because it is modern, not because current pain points require it. For AESF's current scale (thousands of Epic sync events per day, not millions), PostgreSQL queue tables are a reasonable choice. The migration overhead — Kafka cluster operations, Schema Registry, consumer group management — is real and non-trivial on a small team.

---

## 9. Kafka Schema Registry and Message Evolution

In a queue-table model, the "schema" is the PostgreSQL column definitions known implicitly to both producer and consumer. In Kafka, producers and consumers are independent — a consumer deployed months later must still be able to read messages produced today.

**Schema Registry** (Confluent or AWS Glue) solves this by storing Avro/Protobuf/JSON Schema definitions versioned by subject (topic name). Producers register a schema before producing; consumers fetch the schema to deserialize.

```python
# confluent-kafka with Schema Registry (Avro)
from confluent_kafka.schema_registry import SchemaRegistryClient
from confluent_kafka.schema_registry.avro import AvroSerializer, AvroDeserializer
from confluent_kafka.serialization import SerializationContext, MessageField

schema_registry_client = SchemaRegistryClient({'url': 'http://schema-registry:8081'})

account_event_schema = """
{
  "type": "record",
  "name": "AccountEvent",
  "namespace": "com.aesf.events",
  "fields": [
    {"name": "account_id",      "type": "string"},
    {"name": "event_type",      "type": "string"},
    {"name": "name",            "type": "string"},
    {"name": "epic_client_id",  "type": ["null", "string"], "default": null},
    {"name": "occurred_at",     "type": "string"}
  ]
}
"""

avro_serializer = AvroSerializer(schema_registry_client, account_event_schema)

# Producer uses schema-aware serializer
producer.produce(
    topic='aesf.account.events',
    key=account_id.encode(),
    value=avro_serializer(
        {"account_id": account_id, "event_type": "AccountRenamed",
         "name": "New Name", "epic_client_id": None,
         "occurred_at": "2026-10-12T09:00:00Z"},
        SerializationContext('aesf.account.events', MessageField.VALUE)
    )
)
```

Schema evolution rules under Avro backward compatibility: adding a field with a default is safe; removing a field is safe if consumers handle missing fields; changing a field type is a breaking change.

**Common Mistake:** Skipping Schema Registry in development ("we'll add it later") and shipping consumers that hardcode field names. When the schema evolves without a registry, you discover the incompatibility at runtime in production.

---

## 10. Key Concepts Summary

```
Event-Driven Architecture
├── Coupling Models
│   ├── Request-Driven (sync HTTP)       ← AESF Apex→Middleware API calls
│   ├── Queue-Table Async                ← AESF current ETL pattern
│   └── Event-Driven (Kafka)             ← target architecture for scale
│
├── Kafka Internals
│   ├── Broker                           ← stores partition replicas
│   ├── Topic                            ← logical stream, split into partitions
│   ├── Partition                        ← ordered append-only log, unit of parallelism
│   ├── ISR (In-Sync Replicas)           ← durability guarantee
│   ├── Consumer Group                   ← each partition → one consumer, scales horizontally
│   └── Offset                           ← cursor per partition per consumer group
│
├── Delivery Semantics
│   ├── At-most-once                     ← auto-commit before processing
│   ├── At-least-once                    ← commit after processing (default safe choice)
│   └── Exactly-once                     ← idempotent producer + transactional API
│
├── Architectural Patterns
│   ├── Event Sourcing                   ← events as system of record, state = replay
│   ├── CQRS                             ← separate write model (commands) from read model
│   └── Dead Letter Queue               ← isolate poison pills, enable monitoring
│
└── AESF Migration Considerations
    ├── Current scale: queue tables are adequate
    ├── Fan-out need → strongest signal to adopt Kafka
    ├── Replay/audit → event sourcing adds value for Epic sync history
    ├── Hybrid path: Debezium CDC bridges PostgreSQL → Kafka
    └── Operational cost: Kafka requires mature DevOps to justify
```

---

## Quiz — 20 Questions

### Questions

**1.** What is the fundamental difference between request-driven and event-driven architecture in terms of coupling?

**2.** In Kafka, what guarantees that all events for a specific `account_id` arrive at the same partition and in order?

**3.** What is the In-Sync Replica (ISR) set, and why does `acks=all` matter for durability?

**4.** Two independent services need to consume the same Kafka topic without interfering with each other. What Kafka concept enables this, and how does it work?

**5.** What is the difference between a Kafka offset commit and deleting a message from a traditional message queue?

**6.** Why does `enable.auto.commit=True` create a risk of message loss in a slow consumer?

**7.** What two Kafka features must be combined to achieve exactly-once semantics in a read-process-write pipeline?

**8.** Exactly-once in Kafka covers Kafka-to-Kafka operations. What additional pattern is required when the downstream operation is a REST API call to an external system like Epic?

**9.** What is event sourcing, and how does it differ from storing current state in a relational database row?

**10.** What is a "poison pill" message in the context of a Kafka consumer, and how does a Dead Letter Queue address it?

**11.** In the current AESF PostgreSQL queue-table pattern, what SQL clause prevents two ETL workers from claiming the same row simultaneously, and what is the Kafka equivalent?

**12.** What is CQRS, and why is it particularly powerful when combined with event sourcing?

**13.** What does the Kafka Schema Registry do, and what problem does it solve that does not exist in the PostgreSQL queue-table pattern?

**14.** You observe that `sync_queue` in PostgreSQL is frequently over 100,000 rows and autovacuum is running constantly. What does this indicate, and what Kafka feature directly addresses the underlying cause?

**15.** What is Debezium, and how would it enable a low-risk migration from AESF's queue-table pattern to Kafka?

**16.** A Kafka topic has 4 partitions and a consumer group has 6 consumers. How many consumers will be idle, and why?

**17.** What is the difference between Kafka's `retention.ms` setting and the queue-table pattern's `DELETE FROM sync_queue WHERE status='done'`?

**18.** In event sourcing, how is current state derived, and what is a "snapshot" used for?

**19.** A new requirement asks that every Epic sync attempt — including retries and failures — be auditable for 2 years. Which approach (queue table or Kafka with event sourcing) better supports this, and why?

**20.** Name the three strongest signals that AESF should migrate from PostgreSQL queue tables to Kafka, and one strong reason to stay with the current approach.

---

### Answers

??? note "Reveal Answers"

    **1.** Request-driven architecture requires the caller to know the callee's address and availability — if Service B is down, Service A fails immediately. Event-driven architecture inverts this: the producer emits an event to a broker without knowing who will consume it or when. This removes temporal and spatial coupling, enabling producers and consumers to evolve, scale, and fail independently. AESF's Apex trigger calling the middleware REST API is request-driven; a trigger publishing to a Kafka topic would be event-driven.

    **2.** Kafka uses consistent hashing of the message key modulo the number of partitions to determine which partition a record is routed to. Because this hash is deterministic, all messages with the same key (e.g., `account_id = "001ABC"`) always map to the same partition. Within a partition, records are strictly ordered by insertion time. This is why choosing the right partition key — one that co-locates related events — is critical for correctness in systems like AESF where account events must be processed in sequence.

    **3.** The ISR set is the group of partition replicas that are fully caught up with the partition leader. When `acks=all` is set on the producer, the broker only acknowledges a write after every replica in the ISR has persisted the message to disk. This means even if the leader broker fails immediately after the acknowledgment, at least one ISR follower has the data and will be elected as the new leader without data loss. Without `acks=all` (e.g., `acks=1`), acknowledgment comes from the leader alone, and a leader crash before replication completes results in lost messages.

    **4.** Kafka **consumer groups** enable independent consumers of the same topic. Each consumer group maintains its own committed offsets for every partition it reads. When Group A commits offset 500 on Partition 0, Group B's position on that same partition is completely unaffected. You can have any number of consumer groups — e.g., `bde-sync-workers` and `bq-sink-workers` — reading the `aesf.account.events` topic simultaneously, each at their own pace, with zero producer-side changes.

    **5.** Committing a Kafka offset advances a pointer that says "this consumer group has processed up to offset N on this partition" — the message itself remains in the log until the topic's retention period expires. In a traditional message queue (or AESF's `sync_queue` table), consuming a message typically deletes or marks it, making it unavailable to any future reader. Kafka's design means any consumer group can reset to an earlier offset and reprocess historical messages, which is the foundation for replay, backfill, and disaster recovery.

    **6.** With `enable.auto.commit=True`, Kafka commits the offset on a timer (default every 5 seconds), regardless of whether your application has finished processing. If a slow or failing consumer crashes between the auto-commit and the completion of actual processing, Kafka believes those messages were handled and will not re-deliver them. The result is silent message loss — particularly dangerous for AESF sync events where a missed commit means an Epic record is never updated. Manual commit after confirmed processing eliminates this risk.

    **7.** Exactly-once semantics require the **idempotent producer** (`enable.idempotence=True`) combined with the **transactional API** (`transactional.id` + `begin_transaction` / `commit_transaction`). The idempotent producer deduplicates retried produces at the broker using a PID and sequence number. The transactional API atomically groups a produce and a consumer offset commit so that either both succeed or both roll back. Together, they ensure each input message produces exactly one output, even across consumer restarts and network failures.

    **8.** When calling an external REST API like Epic's endpoints, exactly-once requires **idempotency keys** at the application layer. You pass a unique request ID (e.g., derived from the Kafka message's topic + partition + offset) in each API call. If the Epic server has already processed a request with that ID, it returns the previous result without executing the operation again. This is the only way to prevent duplicate Epic records when the Kafka consumer retries after an uncertain API response (e.g., a network timeout where you don't know if the server processed the request).

    **9.** Event sourcing stores the complete history of changes as an immutable sequence of events rather than overwriting a single mutable row. Current state is derived by replaying all events through a fold function. In a relational database, an `UPDATE accounts SET name='Acme Inc'` permanently loses the old name unless you maintain an audit table. With event sourcing, the `AccountRenamed` event is permanently recorded, the previous name is always recoverable, and you can reconstruct the state of any record at any point in time by replaying up to a specific event sequence number.

    **10.** A poison pill is a message that consistently causes consumer processing to fail — perhaps due to malformed data, an unexpected schema, or a bug triggered by a specific value. Without a DLQ, the consumer retries indefinitely, blocking all subsequent messages in that partition forever (since Kafka guarantees order within a partition). A Dead Letter Queue routes the problematic message to a separate topic after N failed retries, allowing the consumer to skip it and continue processing. The DLQ message retains full metadata (original topic, partition, offset, error) so engineers can diagnose and optionally reprocess it later.

    **11.** AESF's queue table uses `SELECT ... FOR UPDATE SKIP LOCKED` — this acquires a row-level lock on matching rows and skips rows already locked by other transactions, allowing multiple ETL workers to each claim distinct batches without double-processing. In Kafka, this problem does not exist in the same form: Kafka's consumer group protocol assigns each partition to exactly one consumer in the group, so two consumers in the same group never read from the same partition simultaneously. The coordination is handled by the Kafka broker (via the group coordinator), not by application-level locking.

    **12.** CQRS (Command Query Responsibility Segregation) separates the write model (commands that change state) from the read model (queries that return state). Combined with event sourcing, the write model produces events rather than directly updating a database, and multiple independent read models (projections) subscribe to those events to build purpose-optimized views. This is powerful because each read model can be denormalized for its specific query patterns, new read models can be added without changing the write path, and any read model can be rebuilt from scratch by replaying the event log — making the system auditable, debuggable, and evolvable.

    **13.** The Schema Registry is a versioned store for the message schemas (Avro, Protobuf, or JSON Schema) used by Kafka producers and consumers. It solves schema evolution: when a producer adds a new field to an event, the registry ensures backward compatibility and consumers using older schema versions can still deserialize the message correctly. In the PostgreSQL queue-table pattern, the "schema" is implicit — both producer and consumer know the table columns because they share the same database. This works when producer and consumer are deployed together but breaks down when they evolve independently, as Kafka consumers in separate services naturally do.

    **14.** Over 100,000 rows with constant autovacuum indicates high write amplification from frequent status column UPDATEs (`pending` → `processing` → `done`). PostgreSQL's MVCC engine creates dead row versions on every UPDATE; autovacuum must reclaim these continuously. Under heavy load, autovacuum cannot keep up, the table bloats, and query performance degrades. Kafka's append-only log design has no equivalent: consumers do not modify the log at all; they only advance a pointer (the offset). There are no dead rows, no vacuum cycles, and throughput scales with disk write bandwidth rather than lock contention.

    **15.** Debezium is an open-source CDC (Change Data Capture) platform that tails the PostgreSQL Write-Ahead Log (WAL) and emits a Kafka event for every INSERT, UPDATE, or DELETE on a watched table. For AESF migration, this means the existing Apex trigger → middleware → queue table INSERT code requires zero changes. Debezium would watch `sync_queue` and produce every new row as a Kafka event automatically. New consumers (BQ sink, audit service) read from Kafka while the existing ETL continues reading PostgreSQL — a true strangler-fig migration that can be reversed at any point before the final cutover.

    **16.** With 4 partitions and 6 consumers in the same group, exactly 2 consumers will be idle. Kafka's partition assignment protocol guarantees each partition is assigned to at most one consumer in a group. With 4 partitions, only 4 consumers can be active; the remaining 2 sit in standby. They serve as automatic failover — if one active consumer dies, a rebalance immediately assigns its partition to a standby consumer. Adding more consumers than partitions provides resilience but not additional parallelism; to increase parallelism you must increase the number of partitions.

    **17.** Kafka's `retention.ms` is a time-based or size-based policy applied by the broker automatically to all messages across all consumers — messages older than the retention window are deleted from disk regardless of whether anyone has read them. This is a background infrastructure operation. AESF's `DELETE FROM sync_queue WHERE status='done'` is an application-driven operation that runs explicitly, only deletes rows the application marked as processed, and requires a WHERE clause index scan. Kafka's retention is operationally simpler (no application code needed) but means consumers that fall behind by more than the retention window will miss messages permanently.

    **18.** In event sourcing, current state is derived by taking an empty initial state and applying every event in sequence through a pure function (`state = fold(events, initial_state)`). For domains with long event histories (millions of events per entity), replaying from the beginning on every read is prohibitively slow. A **snapshot** is a periodic checkpoint that captures the fully-reduced state at a specific event sequence number. On a read, the system loads the most recent snapshot and only replays events that occurred after the snapshot's sequence number, dramatically reducing replay time while maintaining correctness.

    **19.** Kafka with event sourcing is significantly better suited for a 2-year audit requirement. By setting `retention.ms=-1` (infinite) or `retention.bytes` to a large value on the topic, every Epic sync attempt — including retries with full metadata — is permanently stored in order, queryable by replaying the partition. The queue-table approach would require a separate `sync_audit` table with explicit INSERT logic for every retry, careful schema management, and growth management strategies for a 2-year accumulation. More importantly, with event sourcing you can replay any time window to reconstruct exactly what sync attempts were made for any account, which is exactly what a compliance audit requires.

    **20.** The three strongest signals to migrate to Kafka are: (1) **Fan-out** — a second independent consumer (e.g., a new analytics service) needs to read the same sync events without modifying the queue-table schema or ETL code; (2) **Scale** — the `sync_queue` table shows autovacuum pressure, growing bloat, or poll latency exceeding SLA thresholds under normal load; (3) **Replay/audit** — a requirement to reconstruct the complete history of all sync operations or to replay events from a specific point in time. The strongest reason to stay is **operational cost**: Kafka requires a dedicated cluster (or managed service like Confluent Cloud / GCP Pub/Sub), Schema Registry, consumer group monitoring, and a team experienced in distributed broker operations — all of which represent substantial ongoing overhead for a system that currently works correctly at AESF's scale.
