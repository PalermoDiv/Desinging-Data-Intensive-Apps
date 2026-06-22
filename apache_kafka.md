# Apache Kafka

Apache Kafka is a **distributed event streaming platform** designed for high-throughput, fault-tolerant, real-time data pipelines and event-driven applications.

Originally built at LinkedIn and later open-sourced, Kafka is widely used to move data between systems, process streams of events, and build decoupled, scalable architectures.

---

## 1. What Kafka Really Is

At its core, Kafka is a **distributed commit log**:

- Producers append events to a durable log.
- Consumers read from that log at their own pace.
- Events are persisted on disk and can be replayed.
- It functions as a message broker, but optimized for high throughput and horizontal scalability.

Kafka is often compared to message queues (RabbitMQ, ActiveMQ) but differs in key ways: it is log-centric, durable by default, and designed for replay and stream processing.

---

## 2. Core Concepts

### Topic

A **topic** is a category or stream of related events.

Examples: `orders`, `user-signups`, `sensor-readings`, `payments`.

### Partition

A topic is split into **partitions**. Each partition is an ordered, immutable log of events.

- Partitions allow Kafka to scale horizontally.
- Each event inside a partition gets a unique, monotonically increasing **offset**.
- Ordering is guaranteed only within a partition, not across partitions.

### Broker

A **broker** is a Kafka server that stores data and serves client requests.

- A Kafka cluster is made of multiple brokers.
- Data is distributed across brokers via partitions.
- Brokers handle producers, consumers, and replication.

### Producer

A **producer** writes events to a topic.

- Producers can choose which partition to write to, often via a key or round-robin strategy.
- Writes are acknowledged based on a configured level of durability.

### Consumer

A **consumer** reads events from a topic.

- Consumers track their position via an **offset**.
- They can read from the beginning, the latest point, or a specific offset.

### Consumer Group

A **consumer group** is a set of consumers that work together to read from a topic.

- Partitions are divided among the consumers in the group.
- Each partition is consumed by only one consumer in the group.
- Adding more consumers can increase parallelism up to the number of partitions.

### Offset

An **offset** is a numeric identifier for the position of an event within a partition.

- Kafka stores committed offsets so consumers can resume after restarts.
- Consumers can commit offsets manually or automatically.

### Replication

Partitions are **replicated** across multiple brokers for fault tolerance.

- One broker acts as the **leader** for a partition.
- Other brokers act as **followers**, replicating the leader's log.
- If the leader fails, a follower is promoted to leader.

### ZooKeeper / KRaft

Older Kafka versions use **Apache ZooKeeper** to manage cluster metadata and leader election.

Newer versions (Kafka 3.0+) support **KRaft** (Kafka Raft), which removes the ZooKeeper dependency and simplifies operation.

---

## 3. How Kafka Works

```
Producer -> Topic/Partition -> Broker (Leader) -> Follower Brokers
                                     |
                                     v
                              Consumer Group
```

1. A producer sends an event to a topic.
2. Kafka assigns the event to a partition based on its key or a partitioner.
3. The leader broker appends the event to the partition log.
4. Follower brokers replicate the event.
5. Consumers in a consumer group poll the topic and process events.
6. Consumers periodically commit their offsets.

---

## 4. Log-Based Storage

Kafka treats topics as **append-only logs** stored on disk.

- Events are not deleted after being consumed.
- Retention is controlled by time (`log.retention.hours`) or size (`log.retention.bytes`).
- This makes Kafka ideal for replay, auditing, and reprocessing historical data.

### Example partition log

```
Offset 0: {"user_id": 1, "event": "login"}
Offset 1: {"user_id": 2, "event": "purchase"}
Offset 2: {"user_id": 1, "event": "logout"}
```

---

## 5. Delivery Semantics

Kafka supports three common delivery guarantees:

| Semantic | Meaning | Trade-off |
|---|---|---|
| **At-most-once** | Messages may be lost but are not duplicated | Fastest, least reliable |
| **At-least-once** | Messages are delivered but may be duplicated | Balanced reliability |
| **Exactly-once** | Messages are delivered once and only once | Slower, more complex (requires idempotence + transactions) |

Exactly-once semantics are achieved through idempotent producers and Kafka transactions.

---

## 6. Common Use Cases

| Use Case | Why Kafka Fits |
|---|---|
| **Event sourcing** | Durable, ordered log of all system events. |
| **Log aggregation** | Collect logs from many services into one stream. |
| **Metrics and monitoring** | Ingest high-volume telemetry data in real time. |
| **Stream processing** | Process events as they happen with Kafka Streams or ksqlDB. |
| **Data integration (ETL)** | Move data between databases, data lakes, and warehouses. |
| **Activity tracking** | Track user actions, clicks, and events at scale. |
| **Microservices communication** | Decouple services with async messaging. |

---

## 7. When to Use Kafka

Use Kafka when:

- You need high throughput and low-latency event streaming.
- Producers and consumers must be decoupled.
- You want durable, replayable message storage.
- Multiple consumers need to read the same data independently.
- You are building event-driven or stream-processing architectures.
- You can accept operational complexity in exchange for scalability.

Avoid Kafka when:

- You need simple request/response or RPC-style communication.
- Message volume is very low (a traditional queue may be easier).
- Strong ordering across all messages is required (ordering is per-partition only).
- Operational simplicity is more important than scale.

---

## 8. Kafka vs Traditional Message Queues

| Aspect | Kafka | RabbitMQ / Traditional Queue |
|---|---|---|
| Model | Distributed log | Queue with point-to-point or pub/sub |
| Durability | Events persisted and replayable | Usually removed after consumption |
| Throughput | Very high | Moderate to high |
| Ordering | Per partition | Per queue |
| Consumer scaling | Consumer groups share partitions | Multiple consumers compete for messages |
| Use case | Stream processing, event sourcing, logs | Task queues, RPC, simple async messaging |
| Operational complexity | Higher | Lower |

---

## 9. Kafka vs Other Streaming Systems

| Aspect | Apache Kafka | Apache Pulsar | AWS Kinesis | Apache Flink (as consumer) |
|---|---|---|---|---|
| Deployment | Self-hosted or managed | Self-hosted or managed | Fully managed cloud | Stream processing engine |
| Storage model | Distributed log | Tiered storage + log | Managed shard-based log | Does not store data itself |
| Geo-replication | MirrorMaker / cluster linking | Built-in geo-replication | Cross-region streams | N/A |
| Ecosystem | Very large, mature | Growing | AWS-integrated | Strong processing ecosystem |
| Best for | General-purpose streaming, event bus | Multi-tenant, geo-replicated streaming | Managed AWS environments | Complex stream processing |

---

## 10. Key Strengths and Weaknesses

### Strengths

- **High throughput**: Handles millions of events per second.
- **Scalability**: Add brokers and partitions to grow.
- **Durability**: Events are persisted and replicated.
- **Replayability**: Consumers can re-read historical events.
- **Decoupling**: Producers and consumers operate independently.
- **Ecosystem**: Rich tooling including Kafka Connect, Kafka Streams, and ksqlDB.

### Weaknesses

- **Operational complexity**: Requires careful cluster management.
- **Latency**: Optimized for throughput; not always ideal for sub-millisecond latency.
- **Partition limits**: Scaling beyond many partitions adds coordination cost.
- **Learning curve**: Concepts like offsets, replication, and consumer groups take time to master.

---

## 11. Kafka Ecosystem

| Component | Purpose |
|---|---|
| **Kafka Connect** | Integrate Kafka with external systems (databases, S3, Elasticsearch). |
| **Kafka Streams** | Build stream-processing applications using Java/Scala. |
| **ksqlDB** | Query Kafka streams with SQL-like syntax. |
| **Kafka REST Proxy** | Produce and consume messages over HTTP. |
| **Schema Registry** | Manage Avro, Protobuf, or JSON schemas for message contracts. |

---

## 12. Summary

- Kafka is a **distributed, durable, scalable event log**.
- It decouples producers and consumers while preserving events for replay.
- It is ideal for **high-throughput streaming, event sourcing, logs, metrics, and data pipelines**.
- It trades some operational simplicity for massive scale and flexibility.
- When choosing Kafka, consider whether your workload benefits from replay, parallel consumption, and long-term retention — if so, Kafka is often the right tool.