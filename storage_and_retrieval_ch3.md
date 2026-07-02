# Storage and Retrieval (DDIA Chapter 3)

Chapter 3 of *Designing Data-Intensive Applications* explains how databases store data and retrieve it efficiently. The central idea is that a database's storage engine is built around trade-offs: optimizing writes often makes reads more expensive, and optimizing reads usually requires additional work during writes.

This chapter focuses on two major families of storage engines—**log-structured storage** and **page-oriented storage**—and then examines how analytical systems store data differently from transactional systems.

---

## Table of Contents

1. [The Simplest Database](#1-the-simplest-database)
2. [Indexes and Their Trade-offs](#2-indexes-and-their-trade-offs)
3. [Hash Indexes](#3-hash-indexes)
4. [SSTables and LSM-Trees](#4-sstables-and-lsm-trees)
5. [B-Trees](#5-b-trees)
6. [Comparing B-Trees and LSM-Trees](#6-comparing-b-trees-and-lsm-trees)
7. [Other Indexing Structures](#7-other-indexing-structures)
8. [Transaction Processing and Analytics](#8-transaction-processing-and-analytics)
9. [Data Warehousing](#9-data-warehousing)
10. [Column-Oriented Storage](#10-column-oriented-storage)
11. [Compression, Sort Order, and Writing](#11-compression-sort-order-and-writing)
12. [Aggregation and Materialized Views](#12-aggregation-and-materialized-views)
13. [Choosing a Storage Model](#13-choosing-a-storage-model)
14. [Key Takeaways](#14-key-takeaways)

---

## 1. The Simplest Database

A minimal key-value database can be implemented with two operations:

```bash
db_set 123456 '{"name":"Ana","city":"Monterrey"}'
db_get 123456
```

The write operation can append each key-value pair to a file. This file is an **append-only log**: new records are added at the end instead of overwriting old records in place.

```text
123456,{"name":"Ana","city":"Monterrey"}
987654,{"name":"Luis","city":"Guadalajara"}
123456,{"name":"Ana","city":"Saltillo"}
```

If a key is written more than once, its latest value wins. Appending is efficient because sequential writes are generally fast, but reading is slow if the database must scan the entire file to find the latest value.

This reveals the basic purpose of an **index**: it is an additional data structure that helps locate records without scanning all the data.

---

## 2. Indexes and Their Trade-offs

An index is derived from the primary data. It does not change the logical contents of the database; it changes how quickly those contents can be found.

Indexes improve reads but add costs:

- They consume additional storage.
- Every write may also require an index update.
- They add implementation and maintenance complexity.
- More indexes usually mean slower writes.

This is an important systems principle:

> A well-chosen index accelerates the application's real access patterns. Indexing everything wastes space and write throughput.

The storage engine therefore needs to know what kind of queries matter: point lookups, range scans, full-text searches, geospatial queries, or large aggregations.

---

## 3. Hash Indexes

A **hash index** keeps an in-memory hash map from each key to the byte offset of its latest value in the data file.

```text
In-memory index                 Append-only data file

123456 ──────────────────────▶ offset 142: {"city":"Saltillo"}
987654 ───────────────▶         offset 58:  {"city":"Guadalajara"}
```

When a key is updated, the database appends the new value and updates the hash map to point to the new offset. This works well when:

- Keys fit in memory.
- Values are larger than keys.
- Queries mostly request one exact key.
- The workload has frequent updates to a limited set of keys.

### Segments and compaction

An append-only file grows forever unless old versions are removed. Storage engines solve this by splitting the log into **segments**. Once a segment reaches a size limit, it becomes immutable and writes continue in a new segment.

Background work can then:

1. **Compact** a segment by keeping only the newest value for each key.
2. **Merge** several compacted segments into one.
3. Delete old segments after active readers finish using them.

```text
Segment A          Segment B          Merged segment
k1 = old           k1 = new           k1 = new
k2 = value    +    k3 = value   ───▶  k2 = value
                                        k3 = value
```

### Practical details

A real log-based engine also needs to handle:

- **Record format:** binary encodings are usually easier to parse reliably than ad hoc text.
- **Deletion:** a special deletion record called a **tombstone** marks a key as removed.
- **Crash recovery:** the in-memory index must be rebuilt from segment files or restored from a snapshot.
- **Partial writes:** checksums help detect records damaged by a crash.
- **Concurrency:** one writer is common, while multiple readers can safely use immutable segments.

### Limitations

- The entire key set usually needs to fit in memory.
- Hash indexes do not support efficient range queries. Finding every key from `1000` to `2000` requires individual lookups or a scan.

---

## 4. SSTables and LSM-Trees

A **Sorted String Table (SSTable)** is a segment whose key-value pairs are sorted by key. Each key appears only once within a compacted segment.

Sorted segments provide several advantages:

- Merging segments is efficient, similar to the merge step of merge sort.
- The full index does not need every key. A sparse index can point to occasional keys.
- Nearby records can be grouped into compressed blocks.
- Range queries become efficient because keys are stored in order.

### How writes work

An LSM-tree-based storage engine commonly uses this process:

1. A write is added to an in-memory sorted structure called a **memtable**.
2. The write is also recorded in a **write-ahead log (WAL)** for crash recovery.
3. When the memtable reaches a size threshold, it is flushed to disk as an SSTable.
4. Reads check the memtable and then the on-disk SSTables, newest first.
5. Background compaction merges SSTables and discards overwritten or deleted values.

```text
Client write
     │
     ├──▶ Write-ahead log
     │
     ▼
  Memtable
     │ flush
     ▼
  SSTable 3 (newest)
  SSTable 2
  SSTable 1 (oldest)
     │
     ▼ background compaction
  Fewer, larger SSTables
```

This family of structures is called a **Log-Structured Merge Tree (LSM-tree)**.

### Compaction strategies

| Strategy | How it works | Typical trade-off |
|---|---|---|
| **Size-tiered** | Newer, smaller SSTables are merged into older, larger ones. | Good write throughput, but may use more space temporarily. |
| **Leveled** | SSTables are organized into levels with bounded size and limited key-range overlap. | More compaction work, but more predictable reads and space usage. |

### Bloom filters

A read for a missing key could otherwise check many SSTables. A **Bloom filter** is a memory-efficient probabilistic structure that can say:

- “Definitely not in this SSTable,” or
- “Possibly in this SSTable.”

False positives are possible, but false negatives are not. Bloom filters therefore avoid many unnecessary disk reads.

### Typical systems

LSM-tree ideas appear in systems such as LevelDB, RocksDB, Cassandra, HBase, and Lucene.

---

## 5. B-Trees

**B-trees** are the most widely used indexing structure in relational databases. Instead of organizing data into variable-length log segments, a B-tree divides storage into fixed-size **pages**—commonly a few kilobytes each.

Each page contains keys and references to child pages or stored values. The root page leads to intermediate branch pages, which lead to leaf pages.

```text
                    [Root: 40 | 80]
                    /       |       \
                   /        |        \
          [10 | 20 | 30] [50 | 60] [90 | 100]
               leaf          leaf       leaf
```

Searching starts at the root and follows the key ranges until it reaches the correct leaf. Since each page has many children, the tree is shallow even for very large datasets.

### Writing to a B-tree

To update an existing key, the database finds its page, changes the page, and writes it back. To insert a key into a full page, the page splits into two and the parent is updated.

B-trees overwrite pages in place, which creates crash-safety challenges. A common protection is a **write-ahead log**:

1. Record the intended page changes in the WAL.
2. Make the WAL durable.
3. Modify the tree pages.
4. Use the WAL to recover if a crash interrupts the update.

Concurrency control, such as latches, prevents multiple threads from corrupting the tree while changing pages.

### B-tree optimizations

Implementations may improve performance by:

- Using copy-on-write instead of overwriting pages.
- Storing abbreviated keys in branch pages to increase the branching factor.
- Placing nearby leaf pages close together on disk.
- Linking leaf pages for efficient range scans.
- Adding sibling pointers, as in B+ trees, to reduce rebalancing complexity.

---

## 6. Comparing B-Trees and LSM-Trees

Neither structure is universally better. Their behavior depends on the workload, storage hardware, compaction strategy, and database implementation.

| Aspect | LSM-tree | B-tree |
|---|---|---|
| **Write pattern** | Mostly sequential appends and background merges | Updates fixed-size pages in place |
| **Write throughput** | Often higher | Often lower for write-heavy workloads |
| **Read path** | May check multiple structures and SSTables | Usually follows one path from root to leaf |
| **Range queries** | Efficient because SSTables are sorted | Efficient through ordered leaf pages |
| **Space amplification** | Old versions coexist until compaction | Can leave fragmented or partially empty pages |
| **Write amplification** | Compaction may rewrite data repeatedly | WAL and page updates may write data more than once |
| **Predictability** | Compaction can affect latency | Mature and often more predictable for reads |
| **Concurrency** | Background compaction and immutable files help | Page updates require careful concurrency control |

### Why LSM-trees can be faster for writes

Sequential writes generally use storage efficiently. LSM-trees batch random updates in memory and later write sorted files. This can produce high throughput, especially on write-heavy workloads.

### Why B-trees can be faster for reads

A B-tree lookup has a direct, bounded path. An LSM-tree may need to check the memtable, several SSTables, indexes, and Bloom filters before finding a value.

### Compaction pressure

LSM-tree compaction shares disk bandwidth with active reads and writes. If incoming writes arrive faster than compaction can process them, pending work accumulates. Eventually, the system may throttle writes to avoid running out of disk space.

The lesson is to evaluate **tail latency**, not only average throughput: background maintenance can cause occasional slow requests.

---

## 7. Other Indexing Structures

### Secondary indexes

A primary key uniquely identifies a record. A **secondary index** supports queries by other fields, such as email, city, or product category.

Secondary-index values may point directly to the row or contain a list of matching row identifiers. Unlike a primary key, a secondary-index key is not necessarily unique.

### Heap files and clustered indexes

In a **heap file**, rows are stored without a particular order and indexes point to their locations. This avoids duplicating the entire row across multiple indexes.

A **clustered index** stores the row directly with the primary index, so records are physically arranged according to that index. Reads through the clustered key become faster, but secondary indexes must refer back to it.

Some databases use a compromise called a **covering index**, which stores selected columns in the index. A query using only those columns can be answered without reading the main row.

| Approach | Benefit | Cost |
|---|---|---|
| **Heap file** | One canonical row location | Index lookup may require an extra read |
| **Clustered index** | Fast reads by the clustered key | More expensive updates and only one physical ordering |
| **Covering index** | Some queries use the index alone | More storage and write work |

### Multi-column indexes

A concatenated index such as `(last_name, first_name)` sorts by the first column and then the second. It efficiently supports queries by `last_name`, or by both columns, but not necessarily by `first_name` alone.

Multi-dimensional indexes are needed for queries such as geographic bounding boxes. Techniques include space-filling curves and specialized structures such as R-trees.

### Full-text and fuzzy indexes

Exact indexes answer exact conditions. Search systems need to find words, prefixes, spelling variations, and related terms. Full-text indexes commonly use an **inverted index** that maps each term to the documents containing it.

### In-memory databases

When the dataset fits in memory, a database can avoid many disk-oriented structures. Durability may still come from a log, periodic snapshots, or replication.

In-memory systems are valuable not only because memory is fast, but because they can support data structures that are difficult to represent efficiently on disk, such as queues and sets.

---

## 8. Transaction Processing and Analytics

Databases often serve two very different workload patterns.

### Online Transaction Processing (OLTP)

OLTP systems handle application interactions:

- Many short queries.
- Small numbers of records per query.
- Lookups by key or index.
- Frequent inserts and updates.
- Low-latency responses.

Examples include placing an order, updating a profile, or transferring money.

### Online Analytical Processing (OLAP)

OLAP systems support analysis and reporting:

- Fewer but much larger queries.
- Scans across millions or billions of rows.
- Aggregations such as sums, averages, and counts.
- Reads of a few columns from many records.
- Data loaded in bulk and changed less often.

Examples include revenue by region, retention by signup month, or average delivery time by warehouse.

| Characteristic | OLTP | OLAP |
|---|---|---|
| **Main users** | End users and applications | Analysts and business intelligence tools |
| **Query pattern** | Point lookups and small ranges | Large scans and aggregations |
| **Writes** | Frequent, low-latency updates | Bulk import or ETL |
| **Data view** | Current application state | Historical events across systems |
| **Typical storage** | Row-oriented | Column-oriented |

Running heavy analytical queries on the production transaction database can harm user-facing performance. This led to separate analytical databases and **data warehouses**.

---

## 9. Data Warehousing

A **data warehouse** combines data from multiple operational systems into a form designed for analysis.

```text
Application DB ──┐
Billing system ──┼──▶ ETL / ELT ──▶ Data warehouse ──▶ Reports
Event stream ────┤                                      Dashboards
CRM ─────────────┘                                      Analysis
```

The transfer process is traditionally called **Extract, Transform, Load (ETL)**:

1. Extract data from source systems.
2. Transform it into consistent schemas and formats.
3. Load it into the warehouse.

Modern systems may load first and transform inside the warehouse (**ELT**), but the architectural goal is similar: keep analytical work isolated from operational systems.

### Stars and snowflakes

A common warehouse model is the **star schema**:

- A central **fact table** stores events such as purchases, clicks, or calls.
- **Dimension tables** describe attributes such as product, customer, date, or location.

```text
                 Date dimension
                       │
Customer dimension ─ Fact table ─ Product dimension
                       │
                Store dimension
```

A **snowflake schema** further normalizes dimensions into related tables. It reduces duplication but requires more joins and is more complex to navigate.

Fact tables can become enormous because they record individual events. Warehouse storage engines are therefore optimized for scanning and aggregating large datasets.

---

## 10. Column-Oriented Storage

In a row-oriented database, all values from one row are stored together. This is ideal when an application needs the whole record.

```text
Row storage:
[order 1: date, product, quantity, price]
[order 2: date, product, quantity, price]
```

In a **column-oriented** database, all values from one column are stored together.

```text
Column storage:
date:     [d1, d2, d3, ...]
product:  [p1, p2, p3, ...]
quantity: [q1, q2, q3, ...]
price:    [$1, $2, $3, ...]
```

An analytical query may read billions of rows but only two or three columns. Column storage avoids loading unused columns, reducing disk I/O and memory use.

For reconstruction to work, every column must preserve the same row order: the value at position `i` in each column belongs to the same logical record.

---

## 11. Compression, Sort Order, and Writing

### Column compression

Column values often repeat. A product-category column, for example, may contain only a few distinct values across millions of rows. Storing similar values together makes compression highly effective.

**Bitmap encoding** can represent each distinct value with a bitmap indicating which rows contain it. Bitmap operations then answer filters quickly:

- `category = 'books'` uses one bitmap.
- `category IN ('books', 'music')` uses bitwise OR.
- `category = 'books' AND region = 'north'` uses bitwise AND.

Vectorized execution lets the CPU process batches of compressed column values efficiently instead of interpreting one row at a time.

### Sort order

Rows in a column store can be sorted by commonly queried fields. If data is sorted by `date`, then queries for a date range read a contiguous region instead of scanning the entire dataset.

Sorting by low-cardinality columns also improves compression because repeated values appear together. Secondary sort columns help queries after the first column has narrowed the range.

Replicated copies can use different sort orders to support different query patterns.

### Writes to column stores

Updating a row in place would require changing several separate column files. Analytical systems often handle writes with an LSM-tree-like approach:

1. New data enters an in-memory or row-oriented staging area.
2. Queries combine recent staged data with older column files.
3. Background processes merge new records into immutable column-oriented files.

This favors bulk loading and analytical reads over frequent single-row updates.

---

## 12. Aggregation and Materialized Views

Many analytical queries repeatedly calculate the same metrics. A **materialized view** stores a query's result so it does not need to be recomputed from raw data every time.

Unlike a normal view, which is only a saved query definition, a materialized view contains actual derived data. It must be refreshed when its source data changes.

### Data cubes

A **data cube** precomputes aggregates across several dimensions—for example, sales by date, product, and region.

```text
                 Product
                    ▲
                   / \
                  /   \
              Sales ─────▶ Region
                 │
                 ▼
                Date
```

Data cubes make known queries extremely fast, but they reduce flexibility:

- They only answer dimensions and aggregations chosen in advance.
- They consume additional storage.
- They add refresh and maintenance costs.
- Raw data is still needed for unexpected questions.

Materialized aggregates are best used as an acceleration layer, not as a replacement for detailed source data.

---

## 13. Choosing a Storage Model

| If your workload mainly needs... | Good starting point | Why |
|---|---|---|
| Exact key-value lookups with a manageable key set | Hash index | Simple and fast point reads |
| High write throughput and range scans | LSM-tree | Batches writes and keeps keys sorted |
| Predictable point reads and mature transaction support | B-tree | Direct lookup path and broad database support |
| Frequent access to complete records | Row-oriented storage | Keeps a record's fields together |
| Large scans over a few fields | Column-oriented storage | Reads only relevant columns and compresses well |
| Repeated analytical aggregations | Materialized views or cubes | Precomputes expensive results |

These are starting points, not rules. Real databases often combine techniques: B-trees with write-ahead logs, LSM-trees with Bloom filters, row stores with covering indexes, or column stores with an in-memory write buffer.

When evaluating a database, ask:

1. What are the dominant read and write patterns?
2. Are queries point lookups, ranges, text searches, or aggregations?
3. Must writes have consistently low latency?
4. How much write and space amplification is acceptable?
5. What happens while compaction, checkpointing, or refresh work is running?
6. Does the workload need current operational state or long-term historical analysis?

---

## 14. Key Takeaways

- An index speeds up reads by maintaining additional data, so it always adds write and storage overhead.
- Append-only logs make writes simple and sequential, but they need indexes, segmentation, and compaction.
- Hash indexes are excellent for exact lookups but poor for range queries.
- SSTables and LSM-trees turn random updates into sorted sequential writes, often improving write throughput.
- B-trees organize fixed-size pages into a shallow hierarchy and provide a predictable read path.
- Background maintenance matters: compaction can create write amplification and latency spikes.
- OLTP and OLAP have fundamentally different access patterns and usually benefit from separate systems.
- Row storage works well when queries need whole records; column storage works well when queries scan a few fields across many records.
- Compression, sort order, materialized views, and data cubes exchange flexibility and write cost for faster analytical reads.

The chapter's broader lesson is that a storage engine is not “fast” in isolation. It is fast for a particular workload. Understanding the workload—especially its queries, update patterns, and latency requirements—is what makes the choice of storage technology meaningful.
