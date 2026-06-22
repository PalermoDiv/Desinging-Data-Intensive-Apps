# Microservices Architecture

**Microservices** is an architectural style in which an application is built as a collection of small, independent services. Each service is responsible for a specific business capability, runs in its own process, and communicates with other services over a network.

---

## Concept

Instead of building one large application (a **monolith**), you split the system into smaller pieces. Each microservice:

- Owns a single business function (e.g., orders, payments, inventory, notifications).
- Has its own codebase, database, and deployment pipeline.
- Can be developed, deployed, and scaled independently.

```
┌──────────────────────────────────────────────────────┐
│                     API Gateway                       │
└───────┬───────────────┬───────────────┬───────────────┘
        │               │               │
   ┌────▼────┐     ┌────▼────┐    ┌────▼────┐
   │ Orders  │     │Payments │    │Inventory│
   │ Service │     │ Service │    │ Service │
   └───┬─────┘     └───┬─────┘    └───┬─────┘
       │               │              │
   ┌───▼────┐     ┌───▼────┐    ┌───▼────┐
   │ Orders │     │Payments│    │Inventory│
   │  DB    │     │  DB    │    │   DB    │
   └────────┘     └────────┘    └─────────┘
```

---

## Key Characteristics

| Characteristic | Meaning |
|----------------|---------|
| **Single responsibility** | Each service does one thing well. |
| **Independently deployable** | A change in one service does not require redeploying the whole system. |
| **Decentralized data** | Each service owns its own database; no shared database. |
| **Inter-service communication** | Services talk via HTTP/gRPC APIs, message queues, or event streams. |
| **Technology diversity** | Teams can choose the best language/database for each service. |
| **Organized around business domains** | Services map to business capabilities, not technical layers. |

---

## Use Cases

- Large e-commerce platforms (orders, payments, shipping, recommendations).
- Streaming services with separate services for user profiles, content delivery, billing, and recommendations.
- Financial systems where different teams own payments, fraud detection, and compliance.
- SaaS products that need to scale individual features independently.

---

## When to Use Microservices

Use microservices when:

- The application is **large and complex** enough to justify the overhead.
- Multiple teams need to **develop and deploy independently**.
- Different parts of the system have **different scaling requirements**.
- You want to use **different technologies** for different problems.
- The organization can handle **distributed systems complexity**.

Avoid microservices when:

- The team or application is small.
- You do not yet understand the business domain well.
- You cannot afford the operational complexity (monitoring, tracing, deployments).

---

## Microservices vs Monolith

| Aspect | Monolith | Microservices |
|--------|----------|---------------|
| **Codebase** | Single, unified codebase | Multiple, separate codebases |
| **Deployment** | Deploy everything together | Deploy each service independently |
| **Database** | Often one shared database | Each service has its own database |
| **Scaling** | Scale the whole app | Scale only the services that need it |
| **Communication** | In-process function calls | Network calls (HTTP, gRPC, messaging) |
| **Complexity** | Simpler to develop and test | More complex to operate and debug |
| **Team structure** | One large team | Multiple small, autonomous teams |
| **Failure isolation** | One bug can bring everything down | A failure in one service may not crash others |
| **Best for** | Small to medium applications, early-stage products | Large, evolving, multi-team systems |

---

## Common Communication Patterns

| Pattern | How it works | Best for |
|---------|--------------|----------|
| **Synchronous HTTP/REST** | Service A calls service B and waits for a response. | Simple request/response interactions. |
| **Synchronous RPC (gRPC)** | Strongly typed, fast binary calls between services. | Internal high-performance service-to-service calls. |
| **Asynchronous messaging** | Services publish events to a message broker (Kafka, RabbitMQ). | Decoupling services, handling backpressure, event-driven systems. |
| **Event sourcing** | State changes are stored as a sequence of events. | Audit trails, complex domains, eventual consistency. |

---

## Common Challenges

| Challenge | Description |
|-----------|-------------|
| **Distributed complexity** | Debugging, tracing requests, and understanding failures is harder. |
| **Data consistency** | Transactions that span multiple services are difficult; use **Saga pattern** or eventual consistency. |
| **Network latency** | Service calls over the network are slower than in-process calls. |
| **Operational overhead** | More services mean more deployments, monitoring, logging, and service discovery. |
| **Testing** | Integration testing across services is more complex. |

---

## Best Practices

- **One database per service**: avoid sharing databases so services remain independent.
- **Design for failure**: use timeouts, retries, circuit breakers, and graceful degradation.
- **API contracts**: keep service interfaces stable and versioned.
- **Observability**: use centralized logging, metrics, and distributed tracing.
- **Automated deployments**: CI/CD pipelines are essential when many services deploy independently.
- **Domain-driven design (DDD)**: define clear boundaries around business domains.

---

## When to Use Each Architecture

| If your situation is... | Choose |
|-------------------------|--------|
| Small team, simple product, rapid iteration | **Monolith** |
| Multiple teams, different release cadences | **Microservices** |
| Some features need 10x more scale than others | **Microservices** |
| Different teams need different tech stacks | **Microservices** |
| Tight budget, limited DevOps experience | **Monolith** |
| Complex domain with clear business boundaries | **Microservices** |

---

## Takeaway

Microservices trade **development simplicity** for **organizational and operational flexibility**. They are powerful for large, multi-team systems but add real complexity. A well-structured **monolith** is often the better starting point; move to microservices when the pain of a single codebase outweighs the cost of distribution.
