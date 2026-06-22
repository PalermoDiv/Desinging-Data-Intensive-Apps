# Replication Architectures (DDIA Chapter 5)

Chapter 5 of *Designing Data-Intensive Applications* focuses on **replication**: keeping copies of the same data on multiple machines. The main reason is **fault tolerance / high availability**: if one node goes down, others still have the data.

There are three main replication architectures:

1. Single-Leader (Primary / Master-Replica)
2. Multi-Leader (Multi-Master)
3. Leaderless

---

## 1. Single-Leader Replication

### Concept
One node is designated as the **leader** (also called primary or master). All writes go to the leader. The leader then copies the changes to one or more **followers** (replicas / secondaries). Reads can be served by either the leader or the followers.

```
┌─────────────┐       ┌─────────────┐
│   Client    │──────▶│   Leader    │
└─────────────┘       └──────┬──────┘
                             │ replication stream
              ┌──────────────┼──────────────┐
              ▼              ▼              ▼
        ┌───────────┐  ┌───────────┐  ┌───────────┐
        │ Follower 1│  │ Follower 2│  │ Follower N│
        └───────────┘  └───────────┘  └───────────┘
```

### How writes propagate
- **Synchronous replication**: leader waits until at least one follower confirms it received the write. Safer, but slower.
- **Asynchronous replication**: leader does not wait. Faster, but followers may lag behind, causing stale reads.
- **Semi-synchronous**: one follower is synchronous, the rest are asynchronous (a common compromise).

### Use cases
- Standard relational databases: PostgreSQL, MySQL, SQL Server with replication.
- Read-heavy workloads where you want to scale reads by adding followers.
- When strong consistency for writes is important.

### When to use
- You have many more reads than writes.
- You can tolerate a small delay between writes appearing on followers.
- You want a simple mental model: one place for all writes.

---

## 2. Multi-Leader Replication

### Concept
Multiple nodes are designated as leaders. Writes can go to **any leader**, and each leader forwards changes to the other leaders and to their own followers.

```
        ┌─────────────┐              ┌─────────────┐
        │  Leader A   │◀────────────▶│  Leader B   │
        └──────┬──────┘              └──────┬──────┘
               │                            │
          ┌────┴────┐                  ┌────┴────┐
          ▼         ▼                  ▼         ▼
    ┌──────────┐ ┌──────────┐   ┌──────────┐ ┌──────────┐
    │Follower A1│ │Follower A2│   │Follower B1│ │Follower B2│
    └──────────┘ └──────────┘   └──────────┘ └──────────┘
```

### Use cases
- **Multi-datacenter / geo-distributed deployments**: each region has its own leader, so local writes do not have to cross the ocean.
- **Offline / collaborative editing**: each user device acts like a leader and syncs changes when online (CouchDB, CalDAV, etc.).
- **Application-level conflict resolution** is acceptable.

### When to use
- You need low-latency writes from multiple geographic locations.
- You can accept that simultaneous writes to different leaders may conflict.
- Your application has a natural way to merge conflicting updates.

### Key challenge: conflicts
Because writes can happen independently on multiple leaders, the same record may be modified at the same time, producing a **conflict**.

Examples of conflict resolution:
- **Last-write-wins (LWW)**: keep the write with the newest timestamp; simple but may silently drop data.
- **Per-user / per-datacenter wins**: one source is declared the winner.
- **Application-specific merge**: ask the user or use business logic to combine the values.
- **Conflict-free replicated data types (CRDTs)**: data structures designed to be merged automatically.
- **Version vectors**: track which node has seen which versions.

---

## 3. Leaderless Replication

### Concept
There is no leader at all. Every replica can accept reads and writes. Clients typically write to several replicas in parallel and read from several replicas, using **quorum** logic to ensure consistency.

Popularized by Dynamo-style systems (Amazon Dynamo, Cassandra, Riak, Voldemort).

### Quorums
If there are `n` replicas:
- A write is considered successful if it reaches at least `w` replicas.
- A read is successful if it contacts at least `r` replicas.
- If `w + r > n`, the read and write are guaranteed to overlap on at least one replica (read-after-write consistency in the simple case).

### Use cases
- High-write, always-available systems where any node can accept traffic.
- Shopping carts, session stores, and other eventually-consistent use cases.
- Situations where network partitions are common and you prefer availability over strong consistency.

### When to use
- You need very high availability and can tolerate stale data.
- You can design your application to handle concurrent writes (e.g., using vector clocks or last-write-wins).
- You want to avoid the complexity of leader failover.

---

## Comparison Table

| Aspect | Single-Leader | Multi-Leader | Leaderless |
|--------|---------------|--------------|------------|
| **Number of write nodes** | One | Multiple | Any / all replicas |
| **Write latency** | Depends on leader location; may be high for distant clients | Low for local leaders; leaders sync in background | Low; writes can go to closest replicas |
| **Read scaling** | Easy: add followers | Easy: add followers per leader | Moderate: reads contact multiple replicas |
| **Consistency** | Stronger; one source of truth | Eventual; conflicts possible | Eventual / quorum-based |
| **Conflict handling** | None (one writer) | Required; complex | Required; often LWW or vector clocks |
| **Failure handling** | Needs leader failover | Survives leader failure better | No failover needed |
| **Typical databases** | PostgreSQL, MySQL, MongoDB (default) | MySQL MM, PostgreSQL BDR, CouchDB, CockroachDB | DynamoDB, Cassandra, Riak, Voldemort |
| **Best for** | Read-heavy apps, strong consistency | Geo-distributed writes, offline-first apps | High availability, massive scale, partition tolerance |

---

## When Should You Use Each?

| If your situation is... | Choose |
|-------------------------|--------|
| Mostly reads, one datacenter, want simplicity | **Single-Leader** |
| Multiple regions need to write locally | **Multi-Leader** |
| Users work offline and sync later | **Multi-Leader** |
| Cannot tolerate leader failover downtime | **Leaderless** |
| Need high availability over strong consistency | **Leaderless** |
| Need strong consistency and simple conflict model | **Single-Leader** |

---

## Takeaway

There is no single best architecture. The choice depends on your **consistency needs**, **latency requirements**, **geography**, and **tolerance for conflict resolution complexity**. Most traditional applications start with **single-leader replication** because it is the easiest to reason about. Move to **multi-leader** or **leaderless** only when the scale or geography of the system truly demands it.
