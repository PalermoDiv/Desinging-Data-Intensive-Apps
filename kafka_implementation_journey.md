# The Average Kafka Implementation: From Planning to Production

This document walks through what a typical Kafka implementation looks like in practice — from deciding to use Kafka, to starting it, connecting your application, and running it day-to-day.

---

## 1. Before You Start: Planning

### Decide Why You Need Kafka

Most teams adopt Kafka for one of these reasons:

- Multiple services need to react to the same event.
- A service produces data faster than downstream consumers can process it.
- You need an audit trail or replay capability.
- You are building event-driven microservices or real-time analytics.
- You want to reduce direct coupling between services.

### Identify Your Events

Ask: what are the important things that happen in the system?

Examples:

- `order.created`
- `payment.processed`
- `user.signed_up`
- `inventory.updated`

Each event usually becomes a Kafka topic.

### Choose a Deployment Model

| Option | When to Use It |
|---|---|
| **Local / Docker** | Development, learning, testing |
| **Self-hosted cluster** | Full control, on-premise, cost optimization |
| **Managed Kafka (Confluent Cloud, AWS MSK, Aiven, Upstash)** | Faster time to production, less operational burden |

Most production implementations today use a managed service unless there is a strong reason to self-host.

### Plan the Cluster Size

For production, a minimal reliable Kafka cluster usually has:

- **3 brokers** (allows one broker to fail while maintaining quorum)
- **Replication factor of 3** for important topics
- **Multiple partitions** per topic for parallelism
- **ZooKeeper or KRaft** for cluster coordination (KRaft is preferred in modern versions)

### Define Message Contracts

Choose a serialization format and use a schema registry:

- **Avro**: compact, schema evolution friendly
- **Protobuf**: compact, strongly typed, widely used
- **JSON**: human-readable, easy to debug, less efficient

A schema registry prevents producers and consumers from breaking each other when message formats change.

---

## 2. Starting Kafka

### Local Development Example (Docker)

The most common way to experiment is with Docker Compose:

```yaml
version: '3'
services:
  zookeeper:
    image: confluentinc/cp-zookeeper:latest
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181

  kafka:
    image: confluentinc/cp-kafka:latest
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
```

Run it:

```bash
docker-compose up -d
```

### Create a Topic

```bash
kafka-topics --bootstrap-server localhost:9092 \
  --create \
  --topic order-events \
  --partitions 3 \
  --replication-factor 1
```

In production, `replication-factor` is usually 3.

### Verify the Cluster

List topics:

```bash
kafka-topics --bootstrap-server localhost:9092 --list
```

Describe a topic:

```bash
kafka-topics --bootstrap-server localhost:9092 --describe --topic order-events
```

---

## 3. Connecting Kafka to Your Application

Kafka connects to applications through **client libraries**. Official clients exist for Java, Python, Go, Node.js, .NET, C/C++, and more.

### Producer Side

Your application creates a producer, configures it with broker addresses, and sends events.

#### Java Example

```java
Properties props = new Properties();
props.put("bootstrap.servers", "localhost:9092");
props.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
props.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");

Producer<String, String> producer = new KafkaProducer<>(props);

producer.send(new ProducerRecord<>("order-events", "order-123", "{\"status\":\"created\"}"));
producer.close();
```

#### Python Example

```python
from kafka import KafkaProducer
import json

producer = KafkaProducer(
    bootstrap_servers=['localhost:9092'],
    value_serializer=lambda v: json.dumps(v).encode('utf-8')
)

producer.send('order-events', {'order_id': '123', 'status': 'created'})
producer.flush()
producer.close()
```

### Consumer Side

Your application creates a consumer, subscribes to topics, and polls for messages.

#### Java Example

```java
Properties props = new Properties();
props.put("bootstrap.servers", "localhost:9092");
props.put("group.id", "order-service-group");
props.put("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
props.put("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");

Consumer<String, String> consumer = new KafkaConsumer<>(props);
consumer.subscribe(Arrays.asList("order-events"));

while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
    for (ConsumerRecord<String, String> record : records) {
        System.out.printf("offset = %d, key = %s, value = %s%n",
            record.offset(), record.key(), record.value());
    }
}
```

#### Python Example

```python
from kafka import KafkaConsumer
import json

consumer = KafkaConsumer(
    'order-events',
    bootstrap_servers=['localhost:9092'],
    group_id='order-service-group',
    value_deserializer=lambda m: json.loads(m.decode('utf-8'))
)

for message in consumer:
    print(f"Received: {message.value}")
```

---

## 4. What Happens at Runtime

### Producing an Event

1. Your application calls `producer.send()`.
2. The Kafka client hashes the message key to pick a partition.
3. The client sends the event to the leader broker for that partition.
4. The broker appends the event to the partition log on disk.
5. Follower brokers replicate the event.
6. The broker acknowledges the write based on your `acks` setting.

### Consuming an Event

1. Your application calls `consumer.poll()`.
2. The consumer fetches events from the assigned partitions.
3. It processes the event.
4. It commits the offset to Kafka or an internal topic.
5. On restart, the consumer resumes from the last committed offset.

### Consumer Group Rebalancing

When a consumer joins or leaves a group, Kafka performs a **rebalance**:

- Partitions are redistributed among active consumers.
- Existing consumers may stop processing briefly.
- Too many rebalances can hurt performance.

---

## 5. Typical Application Architecture with Kafka

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│  Order Service  │     │ Payment Service │     │  User Service   │
│   (Producer)    │     │ (Producer +     │     │   (Producer)    │
└────────┬────────┘     │   Consumer)     │     └────────┬────────┘
         │              └────────┬────────┘              │
         │                       │                       │
         └───────────────────────┼───────────────────────┘
                                 │
                    ┌────────────▼────────────┐
                    │     Kafka Cluster       │
                    │  ┌─────┐ ┌─────┐ ┌────┐ │
                    │  │ B1  │ │ B2  │ │ B3 │ │
                    │  └─────┘ └─────┘ └────┘ │
                    └────────────┬────────────┘
                                 │
         ┌───────────────────────┼───────────────────────┐
         │                       │                       │
┌────────▼────────┐     ┌────────▼────────┐     ┌────────▼────────┐
│ Inventory       │     │ Analytics       │     │ Notification    │
│ Service         │     │ Pipeline        │     │ Service         │
│ (Consumer)      │     │ (Consumer)      │     │ (Consumer)      │
└─────────────────┘     └─────────────────┘     └─────────────────┘
```

In this architecture:

- Services publish events when something important happens.
- Other services subscribe to the events they care about.
- No service directly calls another; everything flows through Kafka.

---

## 6. Configuration You Usually Touch

### Producer

| Setting | Purpose |
|---|---|
| `bootstrap.servers` | Broker addresses |
| `acks` | Durability level (`0`, `1`, `all`) |
| `retries` | Retry count on transient failures |
| `linger.ms` | Batch messages before sending |
| `key.serializer` / `value.serializer` | How to serialize messages |

### Consumer

| Setting | Purpose |
|---|---|
| `bootstrap.servers` | Broker addresses |
| `group.id` | Consumer group name |
| `auto.offset.reset` | Start from `earliest` or `latest` |
| `enable.auto.commit` | Whether offsets are committed automatically |
| `max.poll.records` | Number of records per poll |

### Topic

| Setting | Purpose |
|---|---|
| `partitions` | Parallelism and throughput |
| `replication.factor` | Fault tolerance |
| `retention.ms` / `retention.bytes` | How long data is kept |
| `cleanup.policy` | `delete` or `compact` |

---

## 7. Day-to-Day Operations

### Monitoring

Track these metrics:

| Metric | Why It Matters |
|---|---|
| Consumer lag | How far behind consumers are |
| Broker CPU / memory / disk | Cluster health |
| Under-replicated partitions | Replication problems |
| Request latency | Performance |
| Messages in / out per second | Throughput |

### Common Tools

- **Kafka UI / AKHQ / Kowl**: Web interface to inspect topics, messages, consumer groups.
- **Prometheus + Grafana**: Metrics and dashboards.
- **Confluent Control Center**: Commercial management platform.

### Handling Failures

| Problem | Typical Response |
|---|---|
| Consumer lag grows | Scale consumers or increase partitions |
| Broker fails | Wait for leader election, replace broker |
| Message processing fails | Retry, dead-letter queue, or alert |
| Schema mismatch | Use schema registry, enforce compatibility |

---

## 8. Road to Production Checklist

- [ ] Define events, topics, and schemas
- [ ] Choose deployment model (managed vs self-hosted)
- [ ] Set replication factor and partitions for each topic
- [ ] Configure producer/consumer clients with retry and idempotency
- [ ] Use a consumer group for scalability
- [ ] Set up monitoring and alerting
- [ ] Handle errors with retries or dead-letter queues
- [ ] Test failure scenarios (broker failure, consumer restart)
- [ ] Document message contracts and topic ownership

---

## 9. Summary

A typical Kafka implementation follows this flow:

1. **Plan**: identify events, topics, schemas, deployment model, and cluster size.
2. **Start**: deploy brokers, create topics, verify the cluster.
3. **Connect**: add Kafka client libraries to your application.
4. **Produce**: publish events from services.
5. **Consume**: read and process events in consumer groups.
6. **Operate**: monitor lag, health, and failures; scale as needed.

Kafka acts as the nervous system of the architecture: services emit events, and other services react to them independently and asynchronously.