# Week 16 — Distributed Systems Fundamentals

**Week of:** September 21, 2026
**Estimated study time:** ~2 hours
**Tags:** `distributed-systems` `consensus` `architecture`

---

## Overview

Distributed systems are the backbone of every modern, scalable application — and they introduce a category of problems that simply do not exist when everything runs on a single machine. The moment you split state across two nodes, you are confronted with questions about consistency, availability, and the reliability of the network between them. For an engineer working on a system like AESF, where a single business transaction must be coherently reflected in three separate systems — Salesforce, the FastAPI middleware's PostgreSQL database, and the Epic EHR backend — mastering this mental model is not optional. It is the core reasoning tool for every architectural decision you make.

This week covers the foundational theory that underpins virtually every distributed database, message broker, and cloud-native system you will encounter. We start with the CAP theorem and its more practical sibling PACELC, which give you a vocabulary for the trade-offs you are constantly negotiating. We then move into concrete consistency models — eventual, strong, and causal — and examine what guarantees each one actually makes to the application layer. From there, we study Raft consensus, the algorithm behind many modern distributed datastores, in enough depth to reason about its failure modes without getting lost in the proof.

The second half of the week focuses on distributed time. Clocks in distributed systems are fundamentally unreliable, and the tools we use to reason about ordering — Lamport timestamps and vector clocks — replace wall-clock time with logical causality. We then examine the failure modes that dominate operational concerns at scale: split brain and network partitions. Understanding how these manifest is what separates engineers who debug distributed failures quickly from those who spend hours in confusion.

Finally, we anchor all of this theory in the technologies you work with every day: PostgreSQL replication (including Cloud SQL's streaming replication and its lag characteristics) and Apache Kafka's partition model. By the end of the week, you should be able to look at the AESF three-way sync — Salesforce trigger fires, middleware enqueues, Epic BDE processes — and name exactly what consistency model it provides, where the partition risks live, and what failure modes require manual reconciliation.

---

## 1. CAP Theorem — The Fundamental Trade-off

CAP theorem, formalized by Eric Brewer and proved by Gilbert and Lynch, states that a distributed system can simultaneously guarantee at most two of three properties: **Consistency** (every read receives the most recent write or an error), **Availability** (every request receives a non-error response), and **Partition Tolerance** (the system continues to operate despite network partitions dropping or delaying messages between nodes).

The critical insight that most engineers miss on first encounter: **partition tolerance is not optional**. Networks partition. GCP's internal network, AWS VPCs, cross-zone links — all of them drop packets, introduce latency spikes, and occasionally segment. A system that cannot tolerate partitions is a system that stops working during any network interruption. So the real choice is not CAP as three options but rather: **given that partitions will happen, do you prefer CP (consistency over availability) or AP (availability over consistency)?**

```
                    CAP TRIANGLE
                    
                    Consistency
                         *
                        / \
                       /   \
                  CP  /     \  CA
                     /       \  (not partition-tolerant;
                    /         \  only valid for single-node
                   *-----------*  or LAN with no failures)
            Availability   Partition
                           Tolerance
                           
  CP examples: HBase, Zookeeper, etcd, MongoDB (w/ writeConcern:majority)
  AP examples: Cassandra, CouchDB, DynamoDB (default), DNS
```

**PACELC** extends CAP to cover the non-partition case, which is the common case. The full model says: if there is a **P**artition, choose between **A**vailability and **C**onsistency; **E**lse (when the system is running normally), choose between **L**atency and **C**onsistency. This is far more useful in practice because most of the time your system is not partitioned — but you are still making latency/consistency trade-offs on every write.

| System | Partition choice | Normal operation choice |
|---|---|---|
| PostgreSQL (sync replication) | CP | Low Latency sacrificed for Consistency |
| PostgreSQL (async replication) | AP | Low Latency preferred |
| Kafka (acks=all) | CP | Low Latency sacrificed |
| Kafka (acks=1) | AP | Low Latency preferred |
| Salesforce platform | AP | Low Latency preferred |

**AESF connection:** Salesforce is AP — it prioritizes availability and will process your Apex trigger even if its underlying infrastructure has internal replication lag. The AESF middleware PostgreSQL is configured for async replication in staging and sync in production. Epic BDE is a black box but behaves CP — it will reject duplicate or out-of-order writes. This means AESF spans all three positions on the CAP triangle simultaneously, which is exactly why manual reconciliation jobs exist.

> **Common mistake:** Treating CAP as a system-level label. In practice, *different operations within the same system* can have different CAP characteristics. PostgreSQL in async replication mode is AP for reads from the replica and CP for reads from the primary. Label your operations, not your systems.

---

## 2. Consistency Models — What Guarantees Does Your System Actually Make?

Consistency models form a spectrum from weakest (fastest, most available) to strongest (slowest, most consistent). Understanding where your system sits on this spectrum is the first step to reasoning about anomalies.

```
WEAKEST                                                    STRONGEST
    |                                                           |
Eventual --> Monotonic Read --> Read-Your-Writes --> Causal --> Sequential --> Linearizable
    |                                                           |
 Highest                                                   Lowest
availability                                             availability
& lowest                                                & highest
latency                                                 latency
```

**Eventual Consistency:** Given no new updates, all replicas will eventually converge to the same value. No guarantees about *when*. DNS is the canonical example — a record update propagates to resolvers within minutes, not milliseconds. Salesforce custom object replication across its global infrastructure is eventual. This means an Apex trigger that fires in NA1 may read a value that a trigger in another transaction just updated, and see the old value.

**Strong (Linearizable) Consistency:** Every operation appears to take effect atomically at a single point in time between its invocation and completion. Any read after a write sees that write. This is what PostgreSQL provides for reads and writes on the primary node, and what etcd/Zookeeper provide for their entire keyspace. It is the most intuitive model but requires coordination — every write must wait for acknowledgment from a quorum.

**Causal Consistency:** Operations that are causally related are seen by all nodes in the same order. Concurrent operations (no causal relationship) may be seen in different orders by different nodes. This is weaker than sequential consistency but stronger than eventual. MongoDB's causal sessions, and many distributed databases' default mode, provide this. It is often the right practical choice: you get more availability than linearizability while preventing the most confusing anomalies.

**Read-Your-Writes:** After a client performs a write, any subsequent reads by that same client will reflect that write. This is a session-level guarantee. PostgreSQL achieves this trivially if you always read from the primary. Cloud SQL achieves it for reads from the primary; reads from read replicas may violate it during replication lag.

**AESF connection:** When the AESF middleware writes a client record to PostgreSQL and then immediately queries it back to build the Epic BDE request payload, it relies on Read-Your-Writes consistency. If that read is accidentally routed to the read replica (which happens during connection pool misconfiguration or via SQLAlchemy's `read` bind), you can get a stale result, build the wrong Epic payload, and corrupt the Epic record. This is a real class of bug in the middleware.

> **Common mistake:** Conflating ACID isolation levels with distributed consistency models. PostgreSQL `SERIALIZABLE` isolation is a single-node guarantee about transaction anomalies. It says nothing about what a replica will return. Distributed consistency is about what multiple nodes agree on, which is orthogonal to isolation levels.

---

## 3. Raft Consensus — How Distributed Agreement Actually Works

Raft is a consensus algorithm designed to be understandable (unlike Paxos) while being provably correct. It is used in etcd (Kubernetes' backing store), CockroachDB, TiKV, and Consul. Understanding Raft helps you reason about what happens to GKE's control plane during a zone failure, and why certain distributed operations block until quorum is restored.

**Core roles:**

```
RAFT CLUSTER (5 nodes, quorum = 3)

  +----------+     +----------+     +----------+
  |  Node 1  |     |  Node 2  |     |  Node 3  |
  |  LEADER  |<--->| FOLLOWER |<--->| FOLLOWER |
  +----------+     +----------+     +----------+
       ^                                  ^
       |           +----------+           |
       +---------->|  Node 4  |<----------+
                   | FOLLOWER |
                   +----------+
                        ^
                   +----------+
                   |  Node 5  |
                   | FOLLOWER |
                   +----------+

Leader receives all writes, replicates to followers.
A write is committed once acknowledged by a MAJORITY (3/5).
```

**Leader election:** Nodes start as followers. If a follower does not hear from a leader within an election timeout (randomized 150-300ms), it becomes a candidate and requests votes. A node votes for a candidate only if the candidate's log is at least as up-to-date as its own. A candidate wins if it receives a majority of votes and becomes leader. Randomized timeouts prevent split votes from persisting.

**Log replication:** The leader receives a client write, appends it to its log as an uncommitted entry, sends `AppendEntries` RPC to all followers, waits for a majority to acknowledge, then commits the entry and responds to the client. Followers apply committed entries to their state machines in order.

**What "majority" buys you:** With 5 nodes, you can tolerate 2 simultaneous failures and still commit writes. With 3 nodes, you can tolerate 1 failure. This is why etcd in GKE is typically deployed as a 3-node cluster per region — you tolerate one node (zone) failure.

**Safety property:** Raft guarantees that committed entries are never lost. If an entry is committed (majority acknowledged), any future leader will have that entry. This is the property that makes it safe to build databases on top of Raft.

**AESF connection:** GKE's etcd cluster backs all Kubernetes API state. When a GKE zone fails during an AESF deployment, Kubernetes will stop scheduling new pods until etcd achieves quorum — typically within seconds as the other zones' nodes are still present. If two zones fail simultaneously in a 3-zone cluster, the entire Kubernetes control plane blocks until a zone recovers. Understanding this explains why AESF's GKE setup uses 3 zones (n=3, quorum=2, tolerates 1 failure) rather than 2 zones.

> **Common mistake:** Assuming Raft means zero data loss. Raft commits are durable only if a majority acknowledged. If a leader commits an entry and then all 5 nodes crash before the commit propagates to disk (power failure), the entry may be lost. Raft's durability depends on fsync, and many deployments disable fsync for performance — trading durability for speed.

---

## 4. Distributed Clocks — Lamport Timestamps and Vector Clocks

In a distributed system, there is no global clock. NTP synchronizes clocks to within a few milliseconds, but "a few milliseconds" is an eternity when events fire at microsecond intervals. You cannot use wall-clock time to determine event ordering reliably.

**Lamport Timestamps** provide a logical clock that captures causality. The rules are simple:
1. Each process maintains a counter, initialized to 0.
2. Before sending a message, a process increments its counter.
3. On receiving a message, a process sets its counter to `max(local, received) + 1`.

```
LAMPORT TIMESTAMP EXAMPLE

Process A:   1----2---------5----6
                   \       /
                    \     /
Process B:   1----2--3---4----------
                         \
                          \
Process C:   1----2--------5----6

A sends msg at t=2, B receives at t=3 (max(2,2)+1=3).
B sends msg at t=4, C receives at t=5 (max(4,4)+1=5).

If Lamport(e1) < Lamport(e2), then e1 MAY have caused e2.
If e1 caused e2, then Lamport(e1) < Lamport(e2). (guaranteed)
The inverse is NOT true — concurrent events can have any Lamport ordering.
```

**Limitation:** Lamport timestamps establish a partial order. If `L(A) < L(B)`, A happened-before B OR A and B are concurrent. You cannot distinguish the two cases.

**Vector Clocks** solve this by maintaining a counter per process:

```
VECTOR CLOCK EXAMPLE (3 processes: A, B, C)

A sends event:   A:[1,0,0] --> B receives: B:[1,1,0]
B sends event:   B:[1,2,0] --> C receives: C:[1,2,1]
A sends event:   A:[2,0,0] (concurrent with B's events)

To compare two vector clocks V1 and V2:
  V1 < V2 (V1 happened-before V2) iff every component of V1 <= V2
            AND at least one component is strictly less.
  Otherwise: CONCURRENT.

[1,2,0] vs [2,0,0]:
  Component 0: 1 <= 2 ✓
  Component 1: 2 <= 0 ✗
  --> CONCURRENT (neither happened-before the other)
```

Vector clocks are used in: DynamoDB (versioning), Riak, CRDTs (Conflict-free Replicated Data Types), and distributed debugging tools.

**AESF connection:** When Salesforce fires an Apex trigger that calls the AESF middleware, and simultaneously Epic BDE fires a webhook back to the middleware, both events carry a "last modified" timestamp from their respective systems. But Salesforce's clock and Epic's clock are not synchronized. Using wall-clock timestamps to determine which update "wins" is wrong — you need a causal ordering mechanism. The AESF middleware uses database sequence numbers (PostgreSQL sequences) as a Lamport-style logical clock for its sync queue, which is correct. A common bug would be to use `updated_at` timestamps from both systems to resolve conflicts — this will silently corrupt data during NTP drift or clock skew events.

> **Common mistake:** Thinking that because your cloud provider uses atomic clocks (Google TrueTime, AWS Time Sync Service), you can use wall-clock time for ordering. These reduce uncertainty but do not eliminate it. TrueTime gives you a confidence interval; you still need logic to handle cases where intervals overlap.

---

## 5. Failure Modes — Split Brain and Network Partitions

Two failure modes dominate distributed systems concerns: split brain and network partitions. They are related but distinct.

**Network Partition:** A network partition occurs when the communication link between two groups of nodes is severed, but both groups continue to operate. Neither group has failed — they are both healthy, they just cannot talk to each other.

```
NETWORK PARTITION SCENARIO

BEFORE PARTITION:
  [Node A] <---> [Node B] <---> [Node C]
    Primary        Replica       Replica

DURING PARTITION (link between A and B-C severed):
  [Node A]        [Node B] <---> [Node C]
    Primary         ?               ?
  (still alive,   (can they see a
   serving writes) primary? No!)

AFTER PARTITION HEALS:
  Both sides may have accepted writes.
  Which writes win? How do you merge?
```

**Split Brain:** Split brain is the specific scenario where two nodes both believe they are the primary/leader simultaneously. This typically occurs after a network partition when the replica side promotes a new primary (because it cannot see the old one), and then the partition heals. Now both nodes have accepted writes that the other does not know about.

```
SPLIT BRAIN TIMELINE

t=0: A is primary, B is replica. Network severs.
t=1: B cannot reach A. B's election timeout fires. B promotes itself.
t=2: A is still accepting writes (sees no problem from its side).
     B is also accepting writes.
t=3: Network heals.
t=4: Both A and B claim to be primary with diverged logs.
     SPLIT BRAIN.
```

**Prevention strategies:**
- **STONITH (Shoot The Other Node In The Head):** Before a node promotes itself, it fences the other node — forcibly powers it off or removes its network access. Ensures only one primary can exist.
- **Quorum-based promotion:** A replica only promotes itself if it can contact a majority of nodes. In a 3-node cluster, a single replica cannot form a majority alone, so it will wait rather than promote.
- **Fencing tokens:** Every primary holds a fencing token (monotonically increasing integer). Any operation to shared storage must include the token. Storage rejects operations with stale tokens.

**AESF connection:** PostgreSQL's Cloud SQL high-availability setup uses a quorum-based failover with Google's internal Paxos implementation. If the primary zone fails, the standby promotes only after confirming the primary is inaccessible — preventing split brain. However, if a GKE pod loses connectivity to Cloud SQL for 30 seconds (a network partition), the pod's SQLAlchemy connection pool will have stale connections. Those connections will error on next use, not silently serve stale data — this is the safe behavior. The dangerous scenario is if a caching layer (Redis, or even SQLAlchemy's in-process cache) serves stale reads after a partition heals, which is a real source of AESF consistency bugs.

> **Common mistake:** Assuming that because you use a managed database (Cloud SQL, RDS), split brain is someone else's problem. The database itself may avoid split brain, but your application layer — connection pools, ORM-level caches, application-level write buffers — can create application-level split brain where different pods have different views of reality.

---

## 6. PostgreSQL Replication and Distributed Systems Theory

PostgreSQL's replication system is a direct application of the distributed systems concepts above. Understanding the mapping makes operational reasoning much faster.

**Streaming replication architecture:**

```
PostgreSQL Streaming Replication

  +------------------+          WAL stream          +------------------+
  |    PRIMARY        |  ========================>  |    STANDBY        |
  |  (Cloud SQL)      |  Write-Ahead Log records    |  (Cloud SQL HA)   |
  |                   |                             |                   |
  |  pg_wal/          |                             |  Apply WAL        |
  |  LSN: 0/1A2B3C4D  |                             |  LSN: 0/1A2B3C40  |
  +------------------+                             +------------------+
                                                    Replication lag:
                                                    ~0ms (sync) or
                                                    up to seconds (async)
```

**LSN (Log Sequence Number)** is PostgreSQL's Lamport timestamp. Every WAL record has a monotonically increasing LSN. The replication lag is expressed as the difference between the primary's current LSN and the LSN the standby has applied. You can query this: `SELECT pg_current_wal_lsn() - sent_lsn FROM pg_stat_replication;`

**Synchronous vs. asynchronous replication (PACELC in action):**

| Mode | `synchronous_commit` | Write commits when | Consistency | Latency |
|---|---|---|---|---|
| Async | `off` | WAL flushed to primary | AP (eventual) | Low |
| Local | `local` | WAL flushed to primary | AP | Low |
| Remote write | `remote_write` | Standby received (not flushed) | Between AP/CP | Medium |
| Remote apply | `remote_apply` | Standby applied | CP (strong) | High |
| On (default sync) | `on` | Standby flushed to disk | CP | High |

**Read replicas and consistency:**

Cloud SQL read replicas serve reads with potential lag. If AESF middleware reads from a read replica immediately after writing to the primary, it may observe:
- **Stale reads:** The write has not yet propagated.
- **Non-monotonic reads:** Two sequential reads of the same row, both from replicas, may return newer-then-older values if the reads hit replicas with different lag.

The safe pattern for AESF: write-then-read operations that must be consistent (e.g., write to sync queue, then immediately read back to build Epic request) must read from the primary. SQLAlchemy's `execution_options(postgresql_readonly=True)` or the use of a separate read engine should be applied carefully.

**AESF connection:** The AESF middleware's `app/core/config.py` configures a single `DATABASE_URL` (primary) and an optional `DATABASE_READONLY_URL` (replica). Queries that are safe for replica lag (reporting, list endpoints, admin panel reads) should use the readonly engine. Queries that must reflect the most recent write (sync queue processing, conflict resolution) must use the primary engine. Mixing these up is the root cause of a class of "ghost" sync failures where the middleware reads a record as "pending" even though it was just committed.

> **Common mistake:** Using `pg_sleep` or application-side retry loops to "wait for replication" instead of reading from the primary or using synchronous commit for critical writes. Sleep-based waiting is both fragile (how long is long enough?) and inefficient.

---

## 7. Kafka and Distributed Systems Theory

Kafka's architecture is a direct implementation of distributed log principles. Each Kafka topic partition is an append-only distributed log, and Kafka's consistency and availability characteristics are determined by its replication and acknowledgment configuration.

```
KAFKA PARTITION REPLICATION (replication factor = 3)

  Broker 1 (Leader)      Broker 2 (Follower)    Broker 3 (Follower)
  +---------------+       +---------------+       +---------------+
  | Partition 0   |  ---> | Partition 0   |  ---> | Partition 0   |
  | Offset: 1042  |       | Offset: 1041  |       | Offset: 1039  |
  | (ISR member)  |       | (ISR member)  |       | (ISR member)  |
  +---------------+       +---------------+       +---------------+
  
  ISR = In-Sync Replica set. A replica is "in-sync" if its lag
  is within replica.lag.time.max.ms of the leader.

Producer writes:
  acks=0  -> fire and forget (AP, lowest latency, potential data loss)
  acks=1  -> leader acknowledges (AP, low latency, leader failure = data loss)
  acks=-1 (all) -> all ISR replicas acknowledge (CP, higher latency, no data loss)
```

**Kafka's CAP position:** With `acks=all` and `min.insync.replicas=2` on a 3-replica topic, Kafka is CP during partitions — a partition that loses enough replicas to fall below `min.insync.replicas` will reject writes rather than risk data loss. With `acks=1`, Kafka is AP — it accepts writes even if replicas are lagging, risking data loss on leader failure.

**Consumer offsets and at-least-once delivery:**

Kafka consumers commit offsets to track their position in the log. The default is `enable.auto.commit=true`, which commits offsets periodically. If a consumer processes a message but crashes before the offset is committed, it will reprocess the message on restart — **at-least-once delivery**. This means Kafka consumers must be idempotent: processing the same message twice should produce the same result as processing it once.

**AESF connection:** If AESF were to introduce Kafka as a message bus between Salesforce webhook receivers and the Epic sync workers (a reasonable architecture for decoupling), the Epic sync worker must be idempotent. The Epic BDE API rejects duplicate requests with a specific error code — the sync worker can treat that error as success, achieving at-least-once delivery with idempotent semantics (effectively exactly-once from the business perspective). The sync queue table in the AESF PostgreSQL database already implements a version of this: the `status` column with states `pending → processing → completed/failed` is a manual Kafka-style offset mechanism.

> **Common mistake:** Assuming Kafka's log retention means "nothing is ever lost." Kafka log segments are deleted based on `retention.ms` or `retention.bytes`. If a consumer group falls so far behind that the segments it needs have been deleted, the consumer will see an `OffsetOutOfRangeException` and must reset — either losing messages or reprocessing from the beginning. Monitor consumer lag proactively.

---

## 8. The AESF Three-Way Sync — A Distributed Systems Case Study

The AESF integration is a three-way distributed system. Let us apply every concept from this week to reason about it systematically.

```
THE AESF DISTRIBUTED SYSTEM

+------------------+      Apex Trigger       +------------------+
|   Salesforce     |  ===================>  |  AESF Middleware  |
|  (AP system)     |  HTTP (sync, ~200ms)   |  FastAPI + PG     |
|  Eventual cons.  |                         |  (CP primary,     |
|  Apex triggers   |  <==================   |   AP replica)     |
|  Platform Events |  Webhook callbacks      +------------------+
+------------------+                               |
                                                   | Epic SDK
                                                   | HTTP (sync)
                                                   v
                                         +------------------+
                                         |   Epic BDE       |
                                         |  (CP system)     |
                                         |  Rejects dupes   |
                                         |  Rejects OOO*    |
                                         +------------------+
                                         *OOO = out-of-order

Consistency model of the overall system: CAUSAL AT BEST
- Salesforce → Middleware: at-least-once (Apex retries on failure)
- Middleware → Epic: at-most-once (no retry on Epic 200 OK lost)
- Epic → Middleware → Salesforce: eventual (polling-based sync)
```

**Failure scenarios by category:**

| Failure | CAP category | AESF manifestation | Resolution |
|---|---|---|---|
| Middleware pod crash mid-request | Network partition (partial) | Salesforce got 200 OK, Epic never received | Reconciliation job detects missing Epic record |
| PostgreSQL replica lag 30s | Consistency degradation | Middleware reads stale sync queue, skips ready records | Read from primary for sync queue queries |
| Epic BDE timeout | Availability failure | Middleware marks record as failed | Retry with exponential backoff |
| Salesforce trigger fires twice | At-least-once delivery | Duplicate Epic records | Epic dedup logic + middleware idempotency key |
| GKE zone failure | Network partition | Active middleware pods reduced, leader election delay | 3-zone deployment tolerates 1 zone failure |

**The fundamental consistency guarantee of AESF:** The system provides **causal consistency at the application level** with manual reconciliation to recover from failures. It is not linearizable (the three systems are too far apart in the CAP space), and it is not even eventual (without the reconciliation jobs, failures leave permanent inconsistencies). The reconciliation job is the consistency mechanism — without it, AESF is an eventually inconsistent system that does not converge automatically.

> **Common mistake:** Treating reconciliation jobs as "nice to have" cleanup tasks rather than mandatory correctness mechanisms. In a system that spans AP and CP components, reconciliation is the only way to achieve convergence. Disabling or delaying reconciliation degrades the system's consistency model from eventual to permanently inconsistent.

---

## 9. Key Concepts Summary

```
DISTRIBUTED SYSTEMS FUNDAMENTALS — CONCEPT MAP

Distributed Systems
├── Fundamental Trade-offs
│   ├── CAP Theorem
│   │   ├── Consistency (C) — every read sees latest write
│   │   ├── Availability (A) — every request gets a response
│   │   └── Partition Tolerance (P) — survives network splits
│   │       [Real choice: CP vs AP, P is mandatory]
│   └── PACELC — extends CAP to normal (non-partition) operation
│       └── Else: Latency vs Consistency trade-off
│
├── Consistency Models (weakest → strongest)
│   ├── Eventual — replicas converge "eventually"
│   ├── Read-Your-Writes — session guarantee
│   ├── Monotonic Read — never see older version twice
│   ├── Causal — causally related ops seen in order
│   ├── Sequential — all nodes see same op order
│   └── Linearizable — atomic, real-time ordering
│
├── Consensus
│   └── Raft
│       ├── Leader election (randomized timeouts, majority vote)
│       ├── Log replication (AppendEntries, commit on majority ack)
│       └── Safety (committed entries never lost)
│
├── Distributed Time
│   ├── Lamport Timestamps — logical clock, partial order
│   │   Rule: increment on send; max(local,recv)+1 on receive
│   └── Vector Clocks — one counter per process, detects concurrency
│       Rule: V1 < V2 iff all components ≤ and at least one <
│
├── Failure Modes
│   ├── Network Partition — communication severed, nodes healthy
│   └── Split Brain — two nodes both believe they are primary
│       Prevention: quorum, STONITH, fencing tokens
│
└── Applied Systems
    ├── PostgreSQL Replication
    │   ├── WAL streaming = distributed log
    │   ├── LSN = Lamport timestamp
    │   └── synchronous_commit = CP/AP dial
    └── Kafka
        ├── Partition = distributed append-only log
        ├── ISR + acks=all = CP mode
        ├── acks=1 = AP mode
        └── Consumer offsets = at-least-once delivery
```

---

## Quiz — 20 Questions

### Questions

**1.** CAP theorem states you can only guarantee two of three properties. Which of the three is effectively non-negotiable in any real distributed system, and why?

**2.** How does PACELC improve on CAP for reasoning about production system behavior?

**3.** A PostgreSQL Cloud SQL instance is configured with `synchronous_commit = off` for performance. A client writes a record and immediately reads it back from a read replica. What consistency model does this configuration provide, and what failure scenario does it create for AESF?

**4.** Explain the difference between linearizability and causal consistency. Give an example of an anomaly that causal consistency prevents but that eventual consistency does not.

**5.** In Raft, a 5-node cluster has nodes A (leader), B, C, D, E. Nodes D and E lose connectivity to A, B, C. Can D and E elect a new leader? Why or why not?

**6.** What is the "Log Matching Property" in Raft, and why is it critical for safety?

**7.** A Lamport timestamp of process A is 5, and a Lamport timestamp of process B is 3. Can you conclude that A's event happened after B's event? Explain.

**8.** Two events have vector clocks `[2, 1, 0]` and `[1, 2, 0]` in a 3-process system. What is their causal relationship?

**9.** Describe the split-brain scenario step by step. What is the primary danger, and name two mechanisms used to prevent it?

**10.** In the AESF middleware, a SQLAlchemy query is accidentally routed to the Cloud SQL read replica instead of the primary. An Apex trigger just wrote a record 50ms ago. What can go wrong, and what is the safe fix?

**11.** Kafka is configured with `replication.factor=3`, `min.insync.replicas=2`, and a producer with `acks=all`. One broker fails. Can the producer still write? What happens if two brokers fail?

**12.** What is "at-least-once delivery" in Kafka, and what requirement does it place on consumers?

**13.** Explain how PostgreSQL's LSN (Log Sequence Number) is conceptually equivalent to a Lamport timestamp.

**14.** Why is wall-clock time insufficient for determining event ordering in a distributed system like AESF, even with NTP synchronization?

**15.** A GKE cluster has 3 zones, each with an etcd node. One zone experiences a complete outage. What happens to Kubernetes control plane operations, and why?

**16.** What is the PACELC trade-off for Kafka with `acks=all` versus `acks=1`? Map each to the PACELC model.

**17.** The AESF reconciliation job runs every 15 minutes. What consistency model does the overall AESF system provide between reconciliation runs, and what model does it provide after a successful reconciliation run?

**18.** A fencing token is used to prevent split-brain in storage systems. Describe how it works and why incrementing the token on each leader election is important.

**19.** An Apex trigger in Salesforce calls the AESF middleware synchronously. The middleware times out after 10 seconds (Salesforce limit). The middleware has written the record to PostgreSQL but has not yet called Epic BDE when the timeout occurs. Salesforce marks the callout as failed and will retry. What distributed systems problem does this create, and how should the middleware handle it?

**20.** You are asked to design the AESF sync queue to be idempotent. Using distributed systems concepts from this week, describe the key properties the queue must have and how you would implement them.

---

### Answers

??? note "Reveal Answers"

    **1.** Partition Tolerance is non-negotiable. Networks in any distributed deployment — including cloud VPCs, cross-zone links, and even intra-datacenter networks — experience packet loss, latency spikes, and temporary segmentation. A system that cannot tolerate partitions stops functioning entirely when any such event occurs. Since production networks partition with non-zero probability, designing a system that assumes no partitions means designing a system that will fail in production. The real choice is always CP versus AP: what does your system do *when* a partition occurs?

    **2.** PACELC adds the "else" case: when the system is operating normally without a partition (the common case), it must still choose between lower latency (which often means weaker consistency, such as async replication) and stronger consistency (which requires coordination overhead and increases latency). CAP only addresses the partition scenario, which is rare. PACELC is more useful for daily engineering decisions because most of the time you are not in a partition — you are choosing how much latency to accept for a given consistency level on every single write.

    **3.** With `synchronous_commit = off` and reads from a read replica, the system provides eventual consistency with no session guarantees. For AESF, this creates a stale-read failure mode: the middleware writes an account record to the primary (marking it as "sync pending"), then reads from the replica which has not yet received the WAL record, finds no pending record, and skips the Epic sync. The record stays out of sync until the reconciliation job runs. The safe fix is to route all reads that are functionally coupled to a recent write (particularly sync queue reads) to the primary connection.

    **4.** Linearizability requires every operation to appear atomic at a single real-time point — all subsequent reads, from any node, reflect the write. Causal consistency only requires that causally related operations be seen in order; concurrent operations may be seen in different orders by different nodes. An example anomaly: User A posts a comment, then User B replies to it (causally dependent). Causal consistency guarantees all nodes see A's comment before B's reply. Eventual consistency allows a node to show B's reply without A's comment — a confusing state that causal consistency prevents. However, causal consistency allows two nodes to see two concurrent, unrelated posts in different orders.

    **5.** No, D and E cannot elect a new leader. Raft requires a majority (quorum) vote to elect a leader. With 5 nodes, the majority is 3. D and E together are only 2 nodes — they cannot form a majority. D and E will remain in a candidate state, repeatedly timing out and re-requesting votes, but will never succeed. This is the correct behavior: it prevents split brain by ensuring only the partition with the majority (A, B, C) can elect a leader and accept writes. D and E effectively become unavailable, which is the CP trade-off.

    **6.** The Log Matching Property states: if two log entries in different nodes have the same index and term, then they store the same command, and all preceding entries are also identical. This is enforced by the `AppendEntries` RPC consistency check — a follower rejects entries if its previous entry's index and term don't match. This property ensures that all nodes that commit an entry at a given index have identical logs up to that point, which is the foundation of Raft's safety guarantee that committed entries are never overwritten or lost.

    **7.** No. Lamport timestamps establish a partial order with one-directional implication: if event e1 happened-before e2, then L(e1) < L(e2). The inverse does not hold. L(A)=5 and L(B)=3 means either A happened after B in a causal chain, OR A and B are concurrent events that happened to receive those timestamp values. Without knowing the full message history, you cannot conclude causality from Lamport timestamps alone. Vector clocks are needed to distinguish "happened-before" from "concurrent."

    **8.** The events are concurrent — neither happened before the other. To compare `[2,1,0]` vs `[1,2,0]`: component 0: 2 > 1 (first is greater), component 1: 1 < 2 (first is less). Since neither vector is component-wise ≤ the other, neither happened before the other. They are concurrent events from processes that did not communicate before these events occurred. In a system like AESF, concurrent events from Salesforce and Epic happening simultaneously with no causal link would produce exactly this pattern.

    **9.** Split brain occurs when a network partition causes a replica to believe the primary has failed and self-promote, resulting in two active primaries. Step by step: (1) Primary A and Replica B are healthy; (2) Network severs between A and B; (3) B's health check timeout fires, B cannot reach A, B promotes itself; (4) A is still running and accepting writes; (5) Both A and B accept writes independently; (6) Network heals, both claim to be primary with diverged state. The primary danger is data divergence — writes accepted by A are unknown to B and vice versa, leading to data loss or corruption when reconciling. Prevention mechanisms: quorum-based promotion (B can only promote if it can reach a majority of nodes — impossible alone in a 2-node cluster), and STONITH/fencing (B forcibly powers off or network-isolates A before promoting, ensuring only one primary exists).

    **10.** The SQLAlchemy query hits the read replica, which may be 50ms to several seconds behind the primary. The record written by the Apex trigger may not yet have been applied to the replica. The middleware reads no record (or the old version), fails to build the Epic request, and the sync is silently skipped — no error is raised, the record is simply not processed. The safe fix is to use the primary connection for any query whose correctness depends on observing a write made within the last replication lag window. In SQLAlchemy, maintain two engine instances (primary and replica) and route sync-critical queries explicitly to the primary engine. Never route sync queue reads to the replica.

    **11.** With one broker failed (2 of 3 remaining, ISR has 2 members): yes, the producer can still write. The ISR has 2 members, which meets `min.insync.replicas=2`, so `acks=all` will succeed with 2 acknowledgments. If two brokers fail (1 of 3 remaining, ISR has 1 member): the producer cannot write. The ISR falls below `min.insync.replicas=2`, and Kafka will respond with `NotEnoughReplicasException`. The topic becomes read-only (not unavailable for reads, only for writes). This is the CP behavior: Kafka prefers to block writes rather than risk data loss below the minimum durability threshold.

    **12.** At-least-once delivery means a message may be delivered to a consumer more than once — specifically, if a consumer processes a message but crashes before committing its offset, the message will be redelivered on restart. The requirement this places on consumers is **idempotency**: processing the same message multiple times must produce the same result as processing it once. For AESF, an Epic sync consumer receiving the same "update account" message twice must detect the duplicate (via an idempotency key or by checking Epic's current state) and skip the second write rather than creating a duplicate record in Epic.

    **13.** PostgreSQL's LSN is a monotonically increasing 64-bit integer that identifies a specific position in the Write-Ahead Log. Every write to PostgreSQL advances the LSN. Like a Lamport timestamp, the LSN captures the happened-before relationship for writes to a single node: if write W1 has LSN X and write W2 has LSN Y, and X < Y, then W1 happened before W2. The standby's applied LSN is always ≤ the primary's current LSN, establishing the replication lag as a measure of how many "happened-before" steps the standby is behind. Also like a Lamport timestamp, comparing LSNs across completely independent PostgreSQL instances (no replication relationship) is meaningless.

    **14.** NTP synchronizes clocks to within approximately 1-10 milliseconds under good conditions, but this synchronization is imperfect and variable. Two events that occur within this uncertainty window cannot be reliably ordered by wall-clock timestamp — the clocks of the two machines may disagree about which happened first. In AESF, Salesforce's servers, the GKE pods, and Epic's servers all have independent clocks. A Salesforce event at T=100ms and an Epic event at T=101ms (as measured by their respective servers) could actually be concurrent or even reversed. Using `updated_at` timestamps from different systems to resolve write conflicts will produce incorrect results during any clock skew event, which occur regularly. Logical clocks (Lamport timestamps, sequence numbers, version vectors) are necessary for reliable ordering.

    **15.** With 3 etcd nodes across 3 zones and one zone failing, the remaining 2 nodes (in 2 zones) can form a quorum (majority of 3 = 2). Kubernetes control plane operations — API server writes, scheduler decisions, controller reconciliations — continue normally. The failed zone's etcd node is simply removed from the quorum until it recovers. Pod workloads in the failed zone are rescheduled to the remaining zones by the node controller after the node-not-ready timeout (~5 minutes by default). If two zones fail simultaneously (leaving 1 etcd node), quorum is lost and the Kubernetes API server becomes read-only — no new pods can be scheduled, no deployments can be updated, but existing running pods continue to run since they do not depend on etcd at runtime.

    **16.** Kafka `acks=all`: during a partition, if the ISR falls below `min.insync.replicas`, writes are rejected — this is **CP** (chooses consistency over availability). Else (normal operation), every write must be acknowledged by all ISR members before the producer receives success, adding latency — this is the PACELC **EC** (Else: Consistency over Latency). Kafka `acks=1`: during a partition, the leader alone acknowledges, writes succeed even if replicas cannot be reached — this is **AP** (chooses availability over consistency). Else (normal operation), the leader acknowledges immediately without waiting for follower replication, minimizing latency — this is the PACELC **EL** (Else: Latency over Consistency).

    **17.** Between reconciliation runs, the AESF system provides **no formal convergence guarantee** — it is inconsistent in the formal sense, because failures can leave records permanently diverged without automated correction. This is weaker than eventual consistency, which guarantees convergence given no new updates. After a successful reconciliation run, the system achieves a point-in-time consistent snapshot across all three systems — but this consistency is immediately violated by any new writes that arrive during or after reconciliation. The practical model is: AESF provides **periodic consistency** (consistent at reconciliation checkpoints) with causal consistency within each individual sync transaction. This is an honest characterization that should inform SLA commitments to end users.

    **18.** A fencing token is a monotonically increasing integer issued by a lock service (like ZooKeeper or etcd) each time a lock is granted. Every write to shared storage must include the current fencing token. The storage layer rejects any write with a token lower than the highest token it has seen. During split brain, the old primary holds token N. The new primary receives token N+1. When the old primary (believing it is still leader) attempts to write with token N, the storage rejects it because it has already seen N+1. The increment-on-election property is critical: if tokens were reused or not incremented, a deposed leader could forge a valid token and corrupt state. The monotonic increment ensures that any write from a previous epoch is detectable and rejectable.

    **19.** This is a classic **at-least-once delivery** problem creating a potential **duplicate write** to Epic. Salesforce marks the callout as failed and will retry (at-least-once from Salesforce's perspective). The middleware has already written the record to PostgreSQL — so on retry, a second middleware invocation will attempt to write the same record to Epic. If Epic is idempotent (deduplicates on an external key), this is safe. If Epic is not idempotent, the retry creates a duplicate record. The middleware must implement idempotency: assign a stable `idempotency_key` (e.g., a hash of the Salesforce record ID and version) to each Epic request, and Epic rejects duplicate keys with a known error. Additionally, the middleware should use the PostgreSQL sync queue as a transactional outbox — write to the queue in the same transaction as the business record, and have a separate worker process the queue, enabling the middleware to safely return a timeout to Salesforce without losing the work.

    **20.** An idempotent sync queue requires three key distributed systems properties. First, **stable identity**: each sync operation must have a unique, deterministic key derived from the source record (e.g., `sha256(object_type + salesforce_id + version_number)`), so that duplicate submissions produce the same key and can be detected. Second, **atomic state transitions**: queue entries must move through states (`pending → processing → completed`) using compare-and-swap semantics (PostgreSQL `UPDATE ... WHERE status = 'pending' RETURNING *`), ensuring only one worker processes each entry even with multiple concurrent workers — this is the distributed mutual exclusion property. Third, **at-least-once processing with idempotent effect**: the downstream Epic call must be retried until success, but Epic must accept the idempotency key and return success on duplicate submission. The combination of a stable key, atomic state machine, and idempotent downstream makes the overall queue exactly-once from a business semantics perspective, even though the delivery mechanism is at-least-once.
