# ⚡ Amazon DynamoDB — Lesson 1

## DynamoDB Fundamentals & Core Concepts

This lesson establishes the **foundation** for the remaining 9 lessons. The most important mindset change is:

> **DynamoDB is designed around access patterns, not around relational normalization.**

---

# 1. What is Amazon DynamoDB?

1. **Amazon DynamoDB** is a fully managed NoSQL database service provided by AWS.
2. It supports **key-value** and **document** data models.
3. It is designed for applications requiring **high availability and low-latency access at scale**.
4. AWS manages the underlying infrastructure, replication, hardware, and many operational concerns.
5. Applications interact with DynamoDB primarily through APIs and AWS SDKs.
6. DynamoDB is especially useful when application access patterns can be modeled around keys.

### Simple example

Imagine an e-commerce application:

```text
Customer
   │
   ├── Customer Profile
   ├── Orders
   ├── Shopping Cart
   └── Preferences
```

DynamoDB can model these relationships around the application's **actual lookup patterns** rather than requiring traditional relational joins.

---

# 2. DynamoDB vs RDS

This is one of the most important distinctions.

| Amazon RDS                                      | Amazon DynamoDB                            |
| ----------------------------------------------- | ------------------------------------------ |
| Relational database                             | NoSQL database                             |
| Tables → rows → columns                         | Tables → items → attributes                |
| SQL                                             | API/SDK-based operations                   |
| Schema-oriented                                 | Flexible item attributes                   |
| Joins are common                                | Joins are generally avoided                |
| Relational modeling                             | Access-pattern-driven modeling             |
| Vertical scaling + replicas depending on engine | Managed horizontal scaling architecture    |
| Excellent for relational workloads              | Excellent for key-value/document workloads |

### Interview point

Don't say:

> "DynamoDB is better than RDS."

Say:

> **"DynamoDB and RDS solve different database problems. I choose DynamoDB when the workload and access patterns fit a key-value/document model and require highly scalable, low-latency access."**

---

# 3. What is a DynamoDB Table?

A DynamoDB table is the primary logical container for your data.

Example:

```text
Customers
────────────────────────────────
CustomerId    Name       Country
────────────────────────────────
C001          Alice      USA
C002          Bob        India
C003          David      Canada
```

But unlike a relational table, DynamoDB items can have different attributes.

For example:

```text
Item 1:
CustomerId = C001
Name       = Alice
Country    = USA

Item 2:
CustomerId = C002
Name       = Bob
Phone      = 5551234
```

The second item doesn't necessarily need exactly the same non-key attributes.

### Key point

DynamoDB provides flexibility, but **flexible schema does not mean careless schema design**.

Your keys and access patterns still need deliberate design.

---

# 4. What is an Item?

An **item** is a single record in a DynamoDB table.

Think of it approximately as:

```text
RDS:
Row

DynamoDB:
Item
```

Example:

```json
{
  "CustomerId": "C001",
  "Name": "Alice",
  "Country": "USA",
  "Membership": "Premium"
}
```

This entire object is one DynamoDB **item**.

### Important points

1. Every item has a primary-key value.
2. Items can contain multiple attributes.
3. Different items can have different non-key attributes.
4. Items can contain nested document structures.
5. Item size is subject to DynamoDB service limits.
6. Efficient item design is critical for performance and cost.

---

# 5. What is an Attribute?

An **attribute** is an individual data element inside an item.

Example:

```json
{
  "CustomerId": "C001",
  "Name": "Alice",
  "Age": 30,
  "Premium": true
}
```

Here:

```text
CustomerId → Attribute
Name       → Attribute
Age        → Attribute
Premium    → Attribute
```

You can think of an attribute as somewhat similar to a column value in an RDS row, although DynamoDB's data model is different.

---

# 6. What Data Types Does DynamoDB Support?

DynamoDB supports several categories of data types.

### Scalar types

Examples include:

```text
String
Number
Binary
Boolean
Null
```

Example:

```json
{
  "Name": "Alice",
  "Age": 30,
  "Active": true
}
```

### Document types

DynamoDB supports:

```text
Map
List
```

Example:

```json
{
  "Customer": {
    "Name": "Alice",
    "Address": {
      "City": "New Orleans",
      "State": "Louisiana"
    }
  }
}
```

### Set types

DynamoDB also supports sets such as:

```text
String Set
Number Set
Binary Set
```

Example:

```text
Languages = ["English", "French", "Spanish"]
```

---

# 7. What is a Primary Key?

The **primary key** uniquely identifies an item in a DynamoDB table.

There are two primary-key designs:

### Simple primary key

```text
Partition Key
```

Example:

```text
CustomerId
```

```text
C001
C002
C003
```

### Composite primary key

```text
Partition Key + Sort Key
```

Example:

```text
CustomerId + OrderId
```

```text
CustomerId   OrderId
C001         O1001
C001         O1002
C002         O1003
```

This distinction is fundamental and will be covered deeply in **Lesson 2**.

---

# 8. What is a Partition Key?

The partition key is one of the most important DynamoDB concepts.

Suppose:

```text
CustomerId
```

is your partition key.

You might have:

```text
C001
C002
C003
C004
```

DynamoDB uses partition-key values to distribute data across its underlying infrastructure.

### Good partition-key characteristics

1. High cardinality.
2. Good distribution of traffic.
3. Avoids concentrating requests on one key.
4. Represents an important access pattern.
5. Doesn't create a single extremely popular value.
6. Supports efficient retrieval.

### Bad example

Imagine:

```text
PartitionKey = Country
```

and your entire workload is:

```text
USA
USA
USA
USA
USA
USA
...
```

A disproportionately popular key can create a **hot partition** problem.

We'll study this deeply in Lesson 4.

---

# 9. What is a Sort Key?

A sort key is used with a partition key to form a **composite primary key**.

Example:

```text
Partition Key = CustomerId
Sort Key      = OrderId
```

Data could look like:

```text
CustomerId    OrderId
──────────────────────
C001          O001
C001          O002
C001          O003
C002          O004
C002          O005
```

Now DynamoDB can efficiently retrieve:

```text
All orders for C001
```

This is where DynamoDB becomes extremely powerful.

---

# 10. Why is Access Pattern the Most Important DynamoDB Concept?

With a relational database, you might start by asking:

> "What entities do I have?"

With DynamoDB, you should often start by asking:

> **"How will the application access this data?"**

Suppose an application needs:

```text
1. Get customer by ID
2. Get all orders for customer
3. Get a particular order
4. Get customer's recent orders
```

You design the DynamoDB keys around those requirements.

For example:

```text
PK = CUSTOMER#C001
SK = PROFILE

PK = CUSTOMER#C001
SK = ORDER#O001

PK = CUSTOMER#C001
SK = ORDER#O002

PK = CUSTOMER#C001
SK = ORDER#O003
```

Now one partition-key value can group related information.

This is called **access-pattern-driven data modeling**.

---

# 11. What is Single-Table Design?

Single-table design is a common DynamoDB modeling approach.

Instead of:

```text
Customers Table
Orders Table
Products Table
```

you might have:

```text
ApplicationTable
```

containing multiple entity types.

Example:

```text
PK              SK
──────────────────────────────
CUSTOMER#C001   PROFILE
CUSTOMER#C001   ORDER#O001
CUSTOMER#C001   ORDER#O002
CUSTOMER#C002   PROFILE
CUSTOMER#C002   ORDER#O003
```

This allows related data to be retrieved efficiently through key-based operations.

### Important

Single-table design does **not** mean:

> "Always use one table."

It means:

> **Design your tables around application access patterns and DynamoDB's strengths.**

---

# 12. DynamoDB Does Not Work Like SQL

Suppose you have:

```text
Customers
Orders
```

In SQL you might write:

```sql
SELECT *
FROM Customers c
JOIN Orders o
ON c.CustomerId = o.CustomerId;
```

DynamoDB generally doesn't work this way.

Instead, you might model the data so the required information can be retrieved efficiently using keys.

For example:

```text
PK = CUSTOMER#C001
SK = PROFILE

PK = CUSTOMER#C001
SK = ORDER#001

PK = CUSTOMER#C001
SK = ORDER#002
```

Then the application can query:

```text
PK = CUSTOMER#C001
```

and retrieve the related items.

---

# 13. What is Query?

**Query** is one of DynamoDB's most important read operations.

It retrieves items based on a partition-key value and can apply conditions to the sort key when available.

Example:

```text
PK = CUSTOMER#C001
```

Could return:

```text
PROFILE
ORDER#001
ORDER#002
ORDER#003
```

Query is efficient because it uses the table/index key structure.

### Remember

```text
Query → Key-based access
```

This will become extremely important in Lesson 3.

---

# 14. What is Scan?

A **Scan** examines items across a table or index.

Conceptually:

```text
Scan
 │
 ├── Item 1
 ├── Item 2
 ├── Item 3
 ├── Item 4
 ├── ...
 └── Item N
```

This can be expensive for large tables.

### Production rule

> **Design your application so that normal user requests use Query/GetItem rather than large Scans.**

A Scan isn't inherently forbidden.

It can be appropriate for certain administrative, analytical, or maintenance workloads.

But it should not become the normal mechanism for retrieving application data.

---

# 15. What is Eventually Consistent Read?

DynamoDB supports different consistency models.

An eventually consistent read may return data that is not yet the latest state immediately after a write.

Example:

```text
Write item
   ↓
Primary data updated
   ↓
Read immediately
   ↓
Possibly older value
```

This model can provide lower-cost/high-throughput read behavior depending on the operation.

---

# 16. What is Strongly Consistent Read?

A strongly consistent read requests the most up-to-date data available from the relevant DynamoDB replica set at the time of the read.

Conceptually:

```text
Write
  ↓
Read
  ↓
Latest committed value
```

Strong consistency isn't available in every DynamoDB feature/context, so application design needs to account for the specific operation being used.

We'll cover consistency and capacity in detail later.

---

# 17. Why DynamoDB Can Scale So Well

The key architectural idea is **distributed data and workload**.

Conceptually:

```text
                 DynamoDB
                    │
       ┌────────────┼────────────┐
       ▼            ▼            ▼
   Partition A  Partition B  Partition C
       │            │            │
       ▼            ▼            ▼
     Data         Data         Data
```

DynamoDB manages the underlying infrastructure and distribution.

Your responsibility is primarily to design:

```text
Good keys
   ↓
Good data distribution
   ↓
Efficient access patterns
   ↓
Predictable workload
```

---

# 18. What is a Hot Partition?

Imagine most requests use:

```text
PK = CUSTOMER#VIP
```

while millions of other customers have relatively little traffic.

You could end up with:

```text
Partition
    │
    └── CUSTOMER#VIP
          ↑
       Massive
        traffic
```

This is a **hot partition** pattern.

### Symptoms can include

1. Throttling.
2. Uneven traffic distribution.
3. Increased latency.
4. Poor utilization of overall table capacity.
5. One extremely popular partition-key value.
6. Workload concentrated around a small number of keys.

We'll spend significant time on this in Lesson 4.

---

# 19. What Does "Serverless Database" Mean Here?

DynamoDB is often described as serverless from the application's perspective.

You don't manage:

```text
Servers
Operating system
Database hosts
Storage hardware
Traditional database patching
```

Instead, you interact with the managed service.

But "serverless" doesn't mean:

> "No architecture or operations required."

You still need to manage:

```text
Data model
Access patterns
IAM
Capacity
Indexes
Cost
Monitoring
Backups
DR
Application behavior
```

---

# 20. When Should You Choose DynamoDB?

DynamoDB is a strong candidate when you need:

1. Very high request throughput.
2. Predictable low-latency key-based access.
3. Flexible key-value/document data.
4. Massive horizontal scalability.
5. Managed infrastructure with minimal database administration.
6. Applications whose access patterns can be modeled efficiently around keys.

Examples include:

```text
Shopping carts
Gaming state
User sessions
IoT workloads
Metadata stores
High-volume APIs
Event-driven applications
User profiles
Order state
```

---

# 21. When Should You NOT Choose DynamoDB?

DynamoDB may not be the best fit when your application fundamentally depends on:

1. Complex relational joins.
2. Ad-hoc SQL queries across many dimensions.
3. Highly relational transactional modeling.
4. Frequent unpredictable query patterns.
5. Heavy analytical workloads better suited to analytical databases.
6. Features specifically provided by relational engines.

The correct answer is not:

> "DynamoDB is faster."

The correct answer is:

> **"The database should match the workload and access patterns."**

---

# 22. Real-Time Scenario

### Scenario

You are building an online shopping cart.

Requirements:

```text
1. Get customer's cart
2. Add item to cart
3. Remove item from cart
4. Update quantity
5. Clear cart
```

Would DynamoDB be a good candidate?

### Answer

Yes, potentially.

You could model:

```text
PK = CUSTOMER#123
SK = CART
```

with attributes such as:

```text
CustomerId
ProductId
Quantity
UpdatedAt
```

The critical question isn't:

> "Can DynamoDB store a shopping cart?"

Of course it can.

The important question is:

> **"What exact access patterns does the application need, and can I model the keys to satisfy them efficiently?"**

---

# 23. Real-Time Scenario — User Session Store

Suppose you have:

```text
100 million users
```

and each user needs session information.

Requirement:

```text
Get session by SessionId
```

A possible design:

```text
PK = SESSION#abc123
```

Then:

```text
GetItem
     ↓
SESSION#abc123
     ↓
Session information
```

This is a classic DynamoDB-style access pattern because it is:

```text
High volume
+
Key-based
+
Predictable
+
Low latency
```

---

# 24. Real-Time Scenario — Why a Bad Key Causes Problems

Suppose you use:

```text
PK = Status
```

and 95% of your records have:

```text
Status = ACTIVE
```

Your workload may heavily concentrate around:

```text
ACTIVE
```

Instead, think about the application's actual access pattern and distribution.

For example, a better design might use:

```text
PK = CUSTOMER#123
SK = ORDER#456
```

or another key structure appropriate to the actual workload.

---

# 25. DynamoDB Golden Rules

Memorize these:

### Rule 1

**Start with access patterns.**

### Rule 2

**Choose keys before worrying about secondary indexes.**

### Rule 3

**Prefer Query/GetItem over Scan for normal application traffic.**

### Rule 4

**Design partition keys for good distribution.**

### Rule 5

**Don't model DynamoDB exactly like an RDS database.**

### Rule 6

**Don't add GSIs just because they are available.**

### Rule 7

**Monitor throttling and hot partitions.**

### Rule 8

**Think about consistency explicitly.**

### Rule 9

**Think about cost at the data-model level.**

### Rule 10

**Test the real workload, not just a toy dataset.**

---

# 🧠 Lesson 1 — Interview Questions

### Q1. What is DynamoDB?

**Answer:**
DynamoDB is a fully managed AWS NoSQL database supporting key-value and document data models, designed for highly scalable, low-latency workloads.

### Q2. What is the biggest difference between RDS and DynamoDB?

**Answer:**
RDS is relational and SQL-oriented, while DynamoDB is a NoSQL service designed around key-based access patterns and scalable distributed workloads.

### Q3. What is an item?

**Answer:**
An item is a single record in a DynamoDB table, consisting of attributes and identified by its primary key.

### Q4. What is a partition key?

**Answer:**
A partition key determines the primary access/distribution key for DynamoDB data and should be designed for efficient access and good workload distribution.

### Q5. What is a sort key?

**Answer:**
A sort key is the second component of a composite primary key and allows multiple related items to share the same partition-key value.

### Q6. Query vs Scan?

**Answer:**

```text
Query → Key-based, targeted access
Scan  → Examines a broader table/index
```

For normal production application access, prefer **Query/GetItem** whenever the access pattern allows it.

### Q7. What is the most important DynamoDB design principle?

**Answer:**

> **Design the data model around application access patterns.**

---

# 🔥 Lesson 1 Final Mental Model

```text
                 DYNAMODB
                    │
                    ▼
             NoSQL Database
                    │
          ┌─────────┴─────────┐
          ▼                   ▼
      Key-Value            Document
          │                   │
          └─────────┬─────────┘
                    ▼
                  TABLE
                    │
                    ▼
                  ITEMS
                    │
                    ▼
               ATTRIBUTES
                    │
                    ▼
              PRIMARY KEY
               /         \
              /           \
     Partition Key      Sort Key
            │                │
            └───────┬────────┘
                    ▼
             ACCESS PATTERNS
                    │
          ┌─────────┴─────────┐
          ▼                   ▼
        Query               GetItem
          │
          ▼
      Efficient Access
```

## 🎯 What you should know before Lesson 2

You should now be comfortable explaining:

**DynamoDB → Table → Item → Attribute → Primary Key → Partition Key → Sort Key → Access Pattern → Query vs Scan → Hot Partition.**

**Lesson 2 will go deep into the most important DynamoDB skill: *Primary Keys, Composite Keys, Data Modeling, Single-Table Design, and Access-Pattern Design*.**
