# ⚡ Amazon DynamoDB — STRICT 10-Lesson Roadmap

| Lesson | Topic                                                                 | What You’ll Master                                                                                        |
| ------ | --------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------- |
| **1**  | DynamoDB Fundamentals & Core Concepts                                 | NoSQL fundamentals, tables, items, attributes, keys, partition key, sort key                              |
| **2**  | Primary Keys, Data Modeling & Access Patterns                         | Simple/composite keys, cardinality, single-table design, access-pattern-first modeling                    |
| **3**  | Read/Write Operations & Consistency                                   | GetItem, PutItem, UpdateItem, DeleteItem, Query, Scan, eventually consistent vs strongly consistent reads |
| **4**  | Capacity, Performance & Scaling                                       | RCU, WCU, on-demand, provisioned capacity, throttling, hot partitions, adaptive capacity                  |
| **5**  | Indexes & Advanced Querying                                           | GSI, LSI, sparse indexes, projection, index design, Query vs Scan                                         |
| **6**  | DynamoDB Streams, TTL & Event-Driven Architecture                     | Streams, Lambda integration, TTL, change processing, event-driven patterns                                |
| **7**  | Transactions, Conditional Writes & Advanced Features                  | TransactWriteItems, TransactGetItems, conditional expressions, optimistic locking, idempotency            |
| **8**  | Security, Encryption, IAM & Governance                                | IAM, resource policies, KMS, VPC endpoints, least privilege, auditing                                     |
| **9**  | Backup, Global Tables, DR & Multi-Region                              | PITR, on-demand backups, restore, Global Tables, multi-Region architecture, RPO/RTO                       |
| **10** | Production Architecture, Migration, Operations & Real-World Scenarios | Production design, troubleshooting, migrations, cost optimization, monitoring, incident response          |

---

# Lesson 1 — DynamoDB Fundamentals & Core Concepts

### 1. What is Amazon DynamoDB?

1. DynamoDB is AWS's fully managed NoSQL database service.
2. It is designed for high-scale applications requiring predictable low-latency access.
3. It supports key-value and document data models.
4. AWS manages the underlying infrastructure, availability, and scaling mechanisms.
5. Applications interact with DynamoDB through APIs/SDKs rather than traditional SQL.
6. The most important DynamoDB concept is **designing around access patterns**.

### 2. DynamoDB vs relational databases

1. RDS databases are relational and typically use tables, rows, columns, and SQL.
2. DynamoDB uses items and attributes within tables.
3. DynamoDB does not use traditional joins as the primary modeling approach.
4. DynamoDB data modeling starts with application access patterns.
5. It is particularly powerful for high-scale key-based workloads.
6. RDS is often more appropriate when relational queries and complex joins are fundamental.

### 3. What is a DynamoDB table?

1. A table is the primary logical container for DynamoDB data.
2. It contains items.
3. Each item contains attributes.
4. Every table must have a primary key.
5. Items don't have to all contain exactly the same attributes.
6. The table's key design strongly influences performance and scalability.

### 4. What is an item?

1. An item is roughly analogous to a row in a relational database.
2. It is a collection of attributes.
3. Each item is uniquely identified by its primary key.
4. Items can contain different attributes from other items.
5. Item size has service limits.
6. Good item design is essential for efficient access patterns.

### 5. What is an attribute?

1. An attribute is a data element within an item.
2. It is conceptually similar to a column value.
3. DynamoDB supports scalar, document, and set data types.
4. Attributes don't necessarily have to appear in every item.
5. Key attributes have special importance.
6. Attribute design should support the application's access patterns.

---

# Lesson 2 — Primary Keys, Data Modeling & Access Patterns

### 1. What is a partition key?

1. A partition key uniquely identifies an item when used as a simple primary key.
2. DynamoDB uses the partition-key value to distribute data.
3. Good distribution helps avoid hot partitions.
4. High-cardinality values are generally useful.
5. Poor partition-key choices can create uneven workload distribution.
6. Partition-key design is one of the most important DynamoDB decisions.

### 2. What is a composite primary key?

1. A composite primary key consists of a partition key and sort key.
2. Multiple items can share the same partition-key value.
3. The sort key differentiates those items.
4. This allows related items to be grouped logically.
5. It enables efficient range and ordered queries.
6. Composite keys are fundamental to advanced DynamoDB data modeling.

### 3. What is a sort key?

1. A sort key is the second component of a composite primary key.
2. Items with the same partition key can have different sort-key values.
3. DynamoDB stores/query-orders items according to the sort key.
4. It enables conditions such as ranges and prefixes.
5. Sort keys can encode relationships and access patterns.
6. Good sort-key design can dramatically reduce query complexity.

---

# Lesson 3 — Read/Write Operations & Consistency

### Core operations

1. **PutItem** creates or replaces an item.
2. **GetItem** retrieves a specific item by key.
3. **UpdateItem** modifies attributes.
4. **DeleteItem** removes an item.
5. **Query** efficiently retrieves items based on key conditions.
6. **Scan** examines items across a table/index and should be used carefully.

### Query vs Scan

1. Query is key-driven.
2. Query can efficiently retrieve items for a specific partition key.
3. Scan examines a much broader portion of the table/index.
4. Large scans can consume substantial capacity.
5. Production applications should generally design access patterns around Query.
6. If you frequently need Scan, reconsider the data model.

---

# Lesson 4 — Capacity, Performance & Scaling

### RCU and WCU

1. **RCU** represents read capacity consumption.
2. **WCU** represents write capacity consumption.
3. Provisioned capacity lets you explicitly configure expected throughput.
4. On-demand mode automatically handles capacity according to workload behavior.
5. Incorrect capacity planning can result in throttling or unnecessary cost.
6. Workload patterns should determine the capacity mode.

### Hot partitions

1. A hot partition occurs when disproportionate traffic targets a particular partition-key value.
2. The overall table may appear healthy while one partition is overloaded.
3. Poor key distribution can cause throttling.
4. High-cardinality and well-distributed keys help avoid this.
5. Write sharding can sometimes distribute concentrated workloads.
6. Performance troubleshooting should consider key distribution, not just total capacity.

---

# Lesson 5 — Indexes & Advanced Querying

### GSI

1. A Global Secondary Index provides an alternative key structure.
2. It can support an access pattern that the base table key cannot efficiently satisfy.
3. GSI partition and sort keys can differ from the base table.
4. GSIs can be queried independently.
5. GSI design affects storage and capacity consumption.
6. Create indexes to support real access patterns, not arbitrary fields.

### LSI

1. A Local Secondary Index shares the table's partition key.
2. It provides an alternative sort key.
3. It must be defined when the table is created.
4. It is useful when multiple sort-key access patterns are required.
5. LSIs have different constraints from GSIs.
6. Choose GSI vs LSI based on the required data-access model.

---

# Lesson 6 — Streams, TTL & Event-Driven Architecture

### DynamoDB Streams

1. DynamoDB Streams captures item-level changes.
2. Changes can include inserts, updates, and deletes.
3. Streams can trigger Lambda-based processing.
4. This enables event-driven architectures.
5. Common uses include notifications, indexing, and downstream processing.
6. Consumers must be designed for retries and duplicate processing.

### TTL

1. Time to Live allows eligible items to expire automatically.
2. It is useful for temporary data.
3. Examples include sessions, caches, and temporary records.
4. TTL is not a precise real-time deletion mechanism.
5. Applications should not depend on deletion occurring at an exact timestamp.
6. TTL can help control storage growth for expirable data.

---

# Lesson 7 — Transactions, Conditional Writes & Advanced Features

### Transactions

1. DynamoDB transactions allow multiple operations to be coordinated atomically.
2. They are useful when multiple items must change consistently.
3. `TransactWriteItems` supports transactional writes.
4. `TransactGetItems` supports transactional reads.
5. Transactions have additional capacity/performance considerations.
6. Use them when the consistency requirement genuinely requires transactional behavior.

### Conditional writes

1. Conditional expressions allow writes only when specified conditions are satisfied.
2. They can prevent accidental overwrites.
3. They are useful for optimistic concurrency control.
4. They can implement patterns such as "create only if this key doesn't exist."
5. Conditional failures should be handled by the application.
6. They are often preferable to implementing unnecessary read-before-write logic.

---

# Lesson 8 — Security, Encryption, IAM & Governance

### DynamoDB security model

1. IAM controls access to DynamoDB APIs.
2. Fine-grained permissions should follow least privilege.
3. DynamoDB supports encryption at rest.
4. AWS KMS can be used for encryption-key management.
5. VPC endpoints can provide private connectivity from VPC-based workloads.
6. CloudTrail and other AWS security services can support auditing and governance.

### Production security principle

**Application → IAM Role → DynamoDB**

rather than:

**Application → hardcoded AWS credentials → DynamoDB**

1. Use IAM roles where possible.
2. Avoid hardcoded AWS access keys.
3. Grant only required DynamoDB actions.
4. Restrict access to required tables/indexes.
5. Monitor API activity.
6. Regularly review permissions.

---

# Lesson 9 — Backup, Global Tables, DR & Multi-Region

### Backups

1. DynamoDB supports point-in-time recovery.
2. On-demand backups can provide explicit recovery points.
3. Restore operations can create a recovered table.
4. Backup strategy should match business RPO requirements.
5. Recovery should be tested periodically.
6. Backup is different from replication.

### Global Tables

1. Global Tables support multi-Region DynamoDB deployments.
2. They can provide local access to data in multiple Regions.
3. They support multi-Region resilience.
4. Applications can continue operating during certain regional failures.
5. Multi-Region architectures introduce consistency and conflict considerations.
6. Global Tables should be designed around application requirements rather than simply enabled for DR.

---

# Lesson 10 — Production Architecture, Migration, Operations & Scenarios

### Production DynamoDB architecture

1. Start with application access patterns.
2. Design the partition and sort keys around those patterns.
3. Add GSIs only when required.
4. Choose provisioned or on-demand capacity appropriately.
5. Add Streams, TTL, backups, and Global Tables when justified.
6. Monitor throttling, latency, capacity, errors, and application behavior.

### Production mental model

```text
                 APPLICATION
                      │
                      ▼
                ACCESS PATTERNS
                      │
                      ▼
              DATA MODEL / KEYS
                      │
          ┌───────────┼───────────┐
          ▼           ▼           ▼
      Partition      Sort        GSI
         Key          Key       Design
          │           │           │
          └───────────┼───────────┘
                      ▼
                  DYNAMODB
                      │
       ┌──────────────┼──────────────┐
       ▼              ▼              ▼
    Capacity        Streams          TTL
       │              │              │
       ▼              ▼              ▼
    Scaling         Lambda         Expiry
       │
       ▼
   Monitoring
       │
       ▼
 Backup / PITR / Global Tables
       │
       ▼
 Production DR
```

# 🎯 Strict 10-Lesson Completion Target

By the end of these **10 lessons**, you should be able to confidently answer:

* **What is DynamoDB and when should I use it?**
* **How do I choose a partition key?**
* **How do I design a composite primary key?**
* **How do I model DynamoDB around access patterns?**
* **When should I use Query vs Scan?**
* **What are RCU/WCU and how does capacity work?**
* **How do I identify and solve hot partitions?**
* **When should I use GSI vs LSI?**
* **How do Streams and Lambda work together?**
* **How does TTL work?**
* **When should I use conditional writes?**
* **When do I need DynamoDB transactions?**
* **How do I secure DynamoDB with IAM/KMS?**
* **How do I implement backup and PITR?**
* **How do Global Tables work?**
* **How do I design DynamoDB for multi-Region DR?**
* **How do I troubleshoot throttling?**
* **How do I optimize DynamoDB cost?**
* **How do I migrate an existing application to DynamoDB?**
* **How do I design a production-grade DynamoDB architecture?**

### 🔥 Final DynamoDB mental model

**Access Patterns → Keys → Data Model → Query Efficiency → Capacity → Indexes → Streams/TTL → Security → Backup/DR → Production Operations**

That sequence is the backbone of the **10-lesson DynamoDB course**.
