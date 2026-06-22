# Database Replication: An Overview

Replication means keeping a copy of the same data on multiple machines. It is one of the main tools for improving availability, fault tolerance, and read scalability.

This document explains the concepts, not how to implement them.

---

## Why Replicate?

- **High availability**: If one machine fails, another can take over.
- **Read scalability**: Spread read traffic across many copies.
- **Low latency**: Place copies geographically closer to users.
- **Disaster recovery**: Keep a backup copy in a different location.

---

## Basic Idea: Leader and Followers

The most common replication pattern is **leader-follower** (also called **master-slave** or **primary-replica**).

- **Leader (primary/master)**: Receives all writes.
- **Followers (replicas/slaves)**: Copy data from the leader and serve reads.

When a client writes data, it goes to the leader. The leader then forwards the change to its followers, which update their own copies.

---

## Synchronous Replication

In synchronous replication, the leader **waits** until at least one follower confirms it has received and persisted the write. Only then does the leader tell the client the write succeeded.

### Example
```
Client -> Write to Leader
Leader -> Forward to Follower
Follower -> Acknowledge
Leader -> Confirm success to Client
```

### Pros
- Strong consistency between leader and follower.
- No data loss if the leader fails right after the write.

### Cons
- Higher write latency — every write must travel to the follower and back.
- If the follower is slow or unreachable, writes slow down or fail.
- Less tolerant of network problems.

---

## Asynchronous Replication

In asynchronous replication, the leader writes locally and immediately confirms success to the client. The follower receives the change afterward, with a small delay.

### Example
```
Client -> Write to Leader
Leader -> Confirm success to Client
Leader -> Forward to Follower (in the background)
Follower -> Acknowledge later
```

### Pros
- Low write latency — the client does not wait for followers.
- Leader continues working even if followers are temporarily unreachable.
- Better availability and performance.

### Cons
- If the leader fails before the follower catches up, recent writes can be lost.
- Followers may lag behind, causing temporary inconsistencies.
- Clients reading from a follower might see stale data.

---

## Semi-Synchronous Replication

A middle ground: the leader waits for **at least one** follower to acknowledge the write, but not all of them. Other followers catch up asynchronously.

- Safer than fully asynchronous replication.
- Faster and more available than fully synchronous replication.
- Used when you want a balance between durability and performance.

---

## Sync vs Async Comparison

| Aspect | Synchronous | Asynchronous | Semi-Synchronous |
|---|---|---|---|
| Client wait time | Until follower confirms | Only leader confirms | Until one follower confirms |
| Durability | Strong | Possible data loss | Balanced |
| Latency | Higher | Lower | Medium |
| Availability | Lower if follower fails | Higher | Medium |
| Consistency | Strong | Eventual | Strong-ish |

---

## Replication Architectures

### 1. Single-leader (leader-follower)

One leader handles all writes. Multiple followers copy from the leader and serve reads.

- Simple and widely used.
- Examples: MySQL replication, PostgreSQL streaming replication, MongoDB replica sets.
- Risk: the leader is a single point of failure for writes.

### 2. Multi-leader

Multiple nodes can accept writes and replicate changes to each other.

- Useful for systems spread across multiple data centers.
- More complex because conflicting writes can happen.
- Examples: Cassandra, CockroachDB (in some modes), MySQL Group Replication.

### 3. Leaderless

Any node can accept reads and writes. Clients often write to several nodes and read from several nodes to ensure consistency.

- Used in some highly available distributed databases.
- Requires conflict resolution mechanisms.
- Examples: Amazon DynamoDB (inspired), Cassandra (can be configured), Riak, Voldemort.

---

## Replication Lag

In asynchronous systems, followers can fall behind the leader. This delay is called **replication lag**.

Causes:
- High write load on the leader.
- Slow network between leader and follower.
- Expensive queries running on the follower.

Effects:
- Users might read old data.
- Systems must decide whether to tolerate stale reads or wait for consistency.

---

## Common Examples

| System | Default replication style | Notes |
|---|---|---|
| PostgreSQL | Streaming replication (async by default) | Can be configured as sync |
| MySQL | Async replication | Supports semi-sync and group replication |
| MongoDB | Async replication in replica sets | Configurable write concern |
| Redis | Async replication | Can enable synchronous waits |
| Kafka | Replication within brokers | ISR (in-sync replicas) concept |
| Cassandra | Leaderless / multi-leader | Tunable consistency |
| DynamoDB | Leaderless-inspired | Eventually consistent by default |

---

## Summary

- **Replication** keeps copies of data on multiple machines.
- **Synchronous replication** is safer but slower and less available.
- **Asynchronous replication** is faster and more available but can lose data or serve stale reads.
- **Single-leader** is the simplest and most common architecture.
- **Multi-leader** and **leaderless** architectures are used when availability and geographic distribution are critical.

The right choice depends on how much consistency, availability, and latency your application needs.
