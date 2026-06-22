# Transactions (DDIA Chapter 7)

A **transaction** is a logical unit of work that groups several reads and writes into a single operation. The goal is to provide **safety guarantees** even when errors, crashes, or concurrent access occur.

The most common guarantees are summarized by the acronym **ACID**.

---

## ACID Properties

| Property | Meaning | Why it matters |
|----------|---------|----------------|
| **Atomicity** | All operations in a transaction complete, or none do. | If a crash happens halfway, the database is not left in a half-updated state. |
| **Consistency** | The database moves from one valid state to another. | Enforces invariants (e.g., account balances stay correct). Note: this is partly the application's responsibility. |
| **Isolation** | Concurrent transactions do not interfere with each other. | Each transaction behaves as if it ran alone, even when many run at the same time. |
| **Durability** | Once committed, data is safe even after a crash. | Data is written to disk or replicated so it survives failures. |

> **BASE** is the alternative often mentioned for distributed NoSQL systems: **Basically Available, Soft state, Eventually consistent**. It trades strong guarantees for availability and partition tolerance.

---

## 1. Read Committed

### Concept
The simplest isolation level. It guarantees:

1. You will not read data that has been **written by a transaction that has not yet committed** (no dirty reads).
2. When you read data, you will see only **committed values**.
3. When you write data, you will **overwrite only committed values**.

### Use cases
- Default isolation level in many databases (PostgreSQL, MySQL InnoDB, SQL Server).
- General-purpose OLTP applications.

### When to use
- You want a simple, predictable default.
- Your application does not have complex concurrent write patterns.

### What it does NOT prevent
- **Non-repeatable reads**: reading the same row twice in one transaction can give different values if another transaction committed in between.

---

## 2. Snapshot Isolation

### Concept
Each transaction reads from a **consistent snapshot** of the database as it existed at the start of the transaction. Writes from other transactions do not affect what the current transaction sees until it commits.

This is usually implemented with **Multi-Version Concurrency Control (MVCC)**: the database keeps multiple versions of a row, and each transaction sees the version that was current when it started.

### Use cases
- Long-running read-only queries that should not see partial updates.
- Backups and analytics queries running on an OLTP database.
- Applications where consistent reads are important.

### When to use
- You need **repeatable reads** within a transaction.
- You have long read queries mixed with write-heavy workloads.
- You want reads to not block writes, and writes to not block reads.

### What it does NOT prevent
- **Write skew**: two transactions read the same data, each modifies something based on what it read, and together they violate a constraint.

---

## 3. Repeatable Read

### Concept
Similar to snapshot isolation in many databases. The transaction sees a consistent snapshot and repeatable reads are guaranteed.

In some databases, this level also protects against **phantom reads** (depending on implementation).

### Use cases
- Applications that read related rows multiple times and need them to stay consistent.

---

## 4. Serializability

### Concept
The strongest isolation level. Transactions are executed as if they ran **one after another**, not concurrently. This prevents all concurrency anomalies.

### How it can be implemented

| Technique | How it works |
|-----------|--------------|
| **Two-Phase Locking (2PL)** | Readers and writers acquire shared/exclusive locks on rows. Locks are held until the transaction ends. |
| **Serializable Snapshot Isolation (SSI)** | Uses snapshots plus detection of write-write and read-write conflicts; aborts transactions that could violate serializability. |
| **Actual serial execution** | Run one transaction at a time. Practical only for very high-performance in-memory systems. |

### Use cases
- Financial systems with strict invariants.
- Applications where concurrency bugs are unacceptable.
- Situations where write skew or phantom reads would cause real problems.

### When to use
- Safety is more important than raw throughput.
- You have complex transactions with overlapping data.

---

## Common Concurrency Problems

| Problem | Description | Prevented by |
|---------|-------------|--------------|
| **Dirty read** | Reading uncommitted data from another transaction. | Read committed |
| **Dirty write** | Overwriting uncommitted data from another transaction. | Read committed |
| **Non-repeatable read** | Reading the same row twice gives different results. | Snapshot isolation / repeatable read |
| **Phantom read** | A query returns different rows when run twice in the same transaction. | Serializable isolation |
| **Lost update** | Two transactions read-modify-write the same data; one update is lost. | Serializable isolation or atomic operations |
| **Write skew** | Two transactions read overlapping data and make decisions that together violate a constraint. | Serializable isolation |

---

## Isolation Levels Comparison

| Isolation level | Dirty reads | Non-repeatable reads | Phantom reads | Write skew | Notes |
|-----------------|-------------|----------------------|---------------|------------|-------|
| **Read uncommitted** | Possible | Possible | Possible | Possible | Rarely used |
| **Read committed** | Prevented | Possible | Possible | Possible | Default in many DBs |
| **Snapshot isolation** | Prevented | Prevented | Possible | Possible | Good balance of safety and performance |
| **Repeatable read** | Prevented | Prevented | DB-dependent | DB-dependent | Behavior varies by database |
| **Serializable** | Prevented | Prevented | Prevented | Prevented | Strongest, slowest |

> Many databases do not follow the SQL standard exactly, so check your specific database's documentation.

---

## When to Use Each Isolation Level

| If your situation is... | Choose |
|-------------------------|--------|
| Simple OLTP with mostly independent transactions | **Read committed** |
| Long reads mixed with writes; need consistent snapshots | **Snapshot isolation** |
| Strict financial or inventory invariants | **Serializable** |
| High contention with many conflicting writes | **Serializable Snapshot Isolation** (where available) |
| You need the simplest possible model and can tolerate lower throughput | **Serializable** |

---

## Two-Phase Locking vs MVCC

| Aspect | Two-Phase Locking (2PL) | MVCC (Snapshot Isolation) |
|--------|-------------------------|---------------------------|
| **Reads block writes?** | Yes, if a writer needs a lock held by a reader. | No |
| **Writes block reads?** | Yes, if a reader needs a lock held by a writer. | No |
| **Performance** | Lower throughput under contention | Higher throughput under read-heavy loads |
| **Anomalies prevented** | All serializable anomalies | Dirty reads, non-repeatable reads; not write skew |
| **Typical use** | Systems needing strict serializability | Systems needing consistent snapshots without blocking |

---

## Takeaway

Transactions are the database's way of pretending that concurrency and crashes do not exist. The stronger the isolation level, the more the database does for you — but usually at the cost of performance. **Snapshot isolation** is a practical sweet spot for many applications: it gives consistent reads without the heavy locking of full serializability, but you must still watch out for **write skew** if your application enforces multi-row invariants.
