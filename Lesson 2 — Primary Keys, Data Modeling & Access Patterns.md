# ⚡ Amazon DynamoDB — Lesson 2

## Primary Keys, Data Modeling & Access Patterns

This is arguably the **most important DynamoDB lesson**.

In RDS, you typically start with entities, relationships, normalization, and SQL queries.

In DynamoDB, you should start with:

> **What does the application need to read and write?**

Then design the keys and items around those access patterns.

---

# 1. What is a Primary Key?

Every DynamoDB table has a **primary key**.

It uniquely identifies each item.

DynamoDB supports two primary-key types:

```text
1. Simple Primary Key
   └── Partition Key

2. Composite Primary Key
   ├── Partition Key
   └── Sort Key
```

The choice between these two is fundamental to your data model.

---

# 2. Simple Primary Key

A simple primary key contains only a **partition key**.

Example:

```text
Table: Customers

Partition Key = CustomerId
```

Data:

```text
CustomerId
──────────
C001
C002
C003
C004
```

Each value must uniquely identify an item.

Therefore:

```text
C001 → One item
C002 → One item
C003 → One item
```

You cannot have two separate items with:

```text
CustomerId = C001
```

in the same table when `CustomerId` is the entire primary key.

---

# 3. Composite Primary Key

A composite primary key contains:

```text
Partition Key + Sort Key
```

Example:

```text
PK = CustomerId
SK = OrderId
```

Data:

```text
PK       SK
────────────────
C001     O001
C001     O002
C001     O003
C002     O004
C002     O005
```

Now multiple items can have:

```text
PK = C001
```

because the combination of:

```text
PK + SK
```

must be unique.

Therefore:

```text
C001 + O001
C001 + O002
C001 + O003
```

are different items.

---

# 4. Why Composite Keys Are So Powerful

Suppose a customer has many orders.

You could model:

```text
PK = CUSTOMER#001
SK = ORDER#001

PK = CUSTOMER#001
SK = ORDER#002

PK = CUSTOMER#001
SK = ORDER#003
```

Now the application can ask:

> "Give me all orders for customer 001."

The answer can be retrieved using:

```text
PK = CUSTOMER#001
```

This is exactly the type of access pattern DynamoDB is designed to handle efficiently.

---

# 5. Partition Key vs Sort Key

Think of them this way:

### Partition Key

Answers:

> **Which logical group does this item belong to?**

### Sort Key

Answers:

> **How do I identify/order/filter items within that group?**

Example:

```text
PK = CUSTOMER#001
SK = ORDER#001
```

means:

```text
Customer group
      │
      └── Order 001
```

Another:

```text
PK = CUSTOMER#001
SK = ORDER#002
```

means:

```text
Customer group
      │
      └── Order 002
```

---

# 6. The Most Important DynamoDB Rule

When designing DynamoDB:

> **Do not start by asking "What tables should I create?"**

Start with:

> **"What queries does my application need?"**

For example:

```text
Application requirements:

1. Get customer by ID
2. Get all orders for customer
3. Get one order
4. Get recent orders
5. Get pending orders
```

These requirements determine your data model.

---

# 7. What is an Access Pattern?

An access pattern describes **how the application needs to access data**.

Examples:

```text
Get customer by customer ID

Get all orders for customer ID

Get order by order ID

Get all orders created today

Get all pending orders

Get all products in category
```

These aren't just queries.

They are the **requirements that drive the DynamoDB design**.

---

# 8. Access Pattern First — Example

Imagine an e-commerce application.

Requirements:

```text
AP1: Get customer profile
AP2: Get customer's orders
AP3: Get a specific order
AP4: Get customer's recent orders
```

Instead of immediately creating:

```text
Customers
Orders
```

you first think about how those requests can be satisfied efficiently.

A possible DynamoDB design:

```text
PK              SK
────────────────────────────
CUSTOMER#C001   PROFILE
CUSTOMER#C001   ORDER#O001
CUSTOMER#C001   ORDER#O002
CUSTOMER#C001   ORDER#O003
```

Now:

```text
AP1
PK = CUSTOMER#C001
SK = PROFILE
```

gets the customer profile.

And:

```text
AP2
PK = CUSTOMER#C001
```

can retrieve the customer's related items.

---

# 9. Composite Keys Enable Item Collections

An **item collection** is the group of items sharing the same partition-key value in a table that uses a composite primary key.

Example:

```text
PK              SK
────────────────────────
CUSTOMER#001    PROFILE
CUSTOMER#001    ORDER#001
CUSTOMER#001    ORDER#002
CUSTOMER#001    ORDER#003
CUSTOMER#001    ADDRESS
```

All of these belong to the same logical item collection:

```text
CUSTOMER#001
```

This is an extremely useful DynamoDB modeling technique.

---

# 10. Sort-Key Design

The sort key doesn't have to simply be:

```text
OrderId
```

You can encode meaning into it.

For example:

```text
ORDER#001
ORDER#002
ORDER#003
```

or:

```text
ORDER#2026-08-11#001
ORDER#2026-08-10#002
ORDER#2026-08-09#003
```

This can enable access patterns based on ordering or prefixes.

For example:

```text
begins_with(SK, 'ORDER#')
```

can retrieve order-related items.

---

# 11. Hierarchical Sort Keys

You can encode hierarchy into the sort key.

Example:

```text
PK = CUSTOMER#001

SK =
PROFILE

ORDER#001
ORDER#002
ORDER#003

ORDER#001#ITEM#001
ORDER#001#ITEM#002
```

Conceptually:

```text
CUSTOMER
   │
   ├── PROFILE
   │
   ├── ORDER 001
   │      ├── ITEM 001
   │      └── ITEM 002
   │
   ├── ORDER 002
   │
   └── ORDER 003
```

This allows DynamoDB to represent relationships without traditional joins.

---

# 12. Why DynamoDB Doesn't Depend on Joins

In an RDS system you might have:

```text
Customer
   │
   └── Orders
         │
         └── OrderItems
```

and query them with joins.

In DynamoDB, you may intentionally store related information in a way that allows the application to retrieve it using keys.

For example:

```text
PK              SK
──────────────────────────────
CUSTOMER#001    PROFILE
CUSTOMER#001    ORDER#001
CUSTOMER#001    ORDER#001#ITEM#01
CUSTOMER#001    ORDER#001#ITEM#02
```

This is a fundamental DynamoDB mindset.

---

# 13. Single-Table Design

Single-table design means multiple entity types can coexist within one DynamoDB table.

Example:

```text
PK              SK
──────────────────────────────
CUSTOMER#001    PROFILE
CUSTOMER#001    ORDER#001
CUSTOMER#001    ORDER#002
PRODUCT#101     DETAILS
PRODUCT#102     DETAILS
```

The same table contains:

```text
Customers
Orders
Products
```

but the key structure identifies the entity type.

---

# 14. Why Use Single-Table Design?

It can allow you to:

1. Retrieve related entities efficiently.
2. Reduce the need for joins.
3. Perform multiple related lookups through key-based access.
4. Model complex relationships within DynamoDB's access model.
5. Optimize application-specific access patterns.
6. Take advantage of DynamoDB's strengths rather than treating it like an RDS database.

But it is **not mandatory**.

---

# 15. When Should You Use Multiple Tables?

Multiple DynamoDB tables can make sense when:

1. Workloads have very different operational characteristics.
2. Different applications own different datasets.
3. Data has different lifecycle requirements.
4. Different security boundaries are required.
5. Independent backup/restore or operational management is useful.
6. Combining entities into one table would make the design unnecessarily complex.

The rule is not:

> "Always use one table."

The rule is:

> **Choose the table structure that best satisfies your access patterns and operational requirements.**

---

# 16. Entity Prefixes

A common modeling technique is to prefix keys with entity types.

Example:

```text
CUSTOMER#001
ORDER#1001
PRODUCT#501
```

Instead of:

```text
001
1001
501
```

Why?

Because the prefix communicates meaning.

For example:

```text
PK = CUSTOMER#001
SK = ORDER#1001
```

is immediately understandable.

It also becomes useful when multiple entity types share a table.

---

# 17. Example — E-Commerce Data Model

Suppose we need:

```text
Customer
Order
Order Item
Product
```

Possible structure:

```text
PK              SK
────────────────────────────────
CUSTOMER#001    PROFILE
CUSTOMER#001    ORDER#1001
CUSTOMER#001    ORDER#1002

ORDER#1001      ITEM#001
ORDER#1001      ITEM#002

PRODUCT#P001   DETAILS
PRODUCT#P002   DETAILS
```

The exact design depends on the application's required access patterns.

The important point is:

> **The keys encode the relationships and retrieval patterns.**

---

# 18. Example — Banking Application

Requirements:

```text
1. Get account
2. Get account transactions
3. Get recent transactions
4. Get transaction by ID
```

Possible design:

```text
PK              SK
────────────────────────────────
ACCOUNT#A001    PROFILE
ACCOUNT#A001    TXN#20260811#T001
ACCOUNT#A001    TXN#20260810#T002
ACCOUNT#A001    TXN#20260809#T003
```

Now:

```text
PK = ACCOUNT#A001
```

can target the account's item collection.

The sort-key structure can help organize transactions chronologically.

---

# 19. Sort Key for Time-Based Data

A very common pattern is:

```text
SK = EVENT#timestamp
```

Example:

```text
PK = DEVICE#D001

SK
────────────────────────────
EVENT#2026-08-11T10:00:00
EVENT#2026-08-11T10:05:00
EVENT#2026-08-11T10:10:00
```

This allows time-oriented queries.

For example:

```text
Get events for DEVICE#D001
between two timestamps
```

can be modeled efficiently using sort-key conditions.

This is extremely useful for:

```text
IoT
Logs
Orders
Transactions
Events
User activity
```

---

# 20. Query vs Scan — Data Modeling Perspective

A good DynamoDB model enables:

```text
Application
     ↓
Known key
     ↓
Query
     ↓
Small targeted result set
```

A poor model often leads to:

```text
Application
     ↓
"I don't know where the data is"
     ↓
Scan
     ↓
Large amount of data examined
     ↓
Higher cost / latency
```

### Golden rule

> **If an important application access pattern requires a large Scan, revisit your data model.**

---

# 21. What About Filtering?

Suppose you Query:

```text
PK = CUSTOMER#001
```

and then filter:

```text
Status = PENDING
```

Filtering does **not** magically make the underlying read free.

The database still has to read the relevant items before applying the filter.

Therefore:

```text
Good:
Key condition → retrieve targeted items

Potentially inefficient:
Broad Query → FilterExpression removes most results
```

This is why key design matters so much.

---

# 22. Key Condition vs Filter Condition

This distinction is extremely important.

### Key condition

Used to identify what DynamoDB should retrieve based on the table/index key.

Example:

```text
PK = CUSTOMER#001
```

### Filter condition

Used after the items matching the key condition are evaluated.

Example:

```text
Status = "PENDING"
```

Therefore, don't use filters as a substitute for proper key design when the access pattern is frequent and performance-sensitive.

---

# 23. GSI as an Access-Pattern Tool

Suppose your base table supports:

```text
Get orders by customer
```

but now the business asks:

> "Get all orders by OrderStatus."

Your existing primary key may not support that efficiently.

You could consider a **Global Secondary Index (GSI)**.

For example:

```text
Base Table

PK              SK
CUSTOMER#001    ORDER#1001
CUSTOMER#002    ORDER#1002
```

GSI:

```text
GSI-PK          GSI-SK
────────────────────────
PENDING         2026-08-11
SHIPPED         2026-08-10
```

Now the GSI provides another access path.

We'll study GSI/LSI deeply in **Lesson 5**.

---

# 24. A Critical DynamoDB Design Question

Suppose someone says:

> "We need to search customers by email."

Don't immediately say:

> "I'll scan the table."

Instead ask:

> **"Is searching by email an important application access pattern?"**

If yes, design an appropriate key/index.

For example, a GSI might use:

```text
GSI PK = EMAIL#user@example.com
```

Then:

```text
Query GSI
    ↓
EMAIL#user@example.com
    ↓
Customer
```

That's a much more DynamoDB-oriented solution.

---

# 25. Many-to-One Relationship

Example:

```text
Customer → Orders
```

One customer has many orders.

DynamoDB can model:

```text
PK              SK
──────────────────────
CUSTOMER#001    ORDER#001
CUSTOMER#001    ORDER#002
CUSTOMER#001    ORDER#003
```

This makes:

```text
Get all orders for customer
```

a natural query.

---

# 26. One-to-Many Relationship

The same pattern applies to:

```text
Department → Employees
```

Example:

```text
PK              SK
────────────────────────
DEPT#IT         EMP#001
DEPT#IT         EMP#002
DEPT#IT         EMP#003
```

Query:

```text
PK = DEPT#IT
```

returns the employees belonging to that department.

---

# 27. Many-to-Many Relationships

This requires more thought.

Example:

```text
Students ↔ Courses
```

A student can take many courses.

A course can have many students.

You may need multiple access patterns:

```text
1. Courses for student
2. Students for course
```

One key structure may efficiently support one direction, while a GSI or additional item representation can support the reverse direction.

### Important lesson

> **DynamoDB data modeling is about designing for the required access paths.**

---

# 28. Real-Time Scenario — Customer Orders

### Requirement

An e-commerce application needs:

```text
1. Customer profile
2. All customer orders
3. Recent customer orders
4. Individual order
```

### Poor design

```text
Customers Table
Orders Table

Then repeatedly join/query both.
```

That's relational thinking.

### Better DynamoDB-oriented approach

Potentially:

```text
PK              SK
────────────────────────────
CUSTOMER#001    PROFILE
CUSTOMER#001    ORDER#20260811#001
CUSTOMER#001    ORDER#20260810#002
CUSTOMER#001    ORDER#20260809#003
```

Now the sort key itself contains useful ordering information.

---

# 29. Real-Time Scenario — Millions of Users

Suppose:

```text
100 million customers
```

and you need:

```text
Get customer profile by CustomerId
```

A natural key is:

```text
PK = CUSTOMER#<CustomerId>
```

For example:

```text
CUSTOMER#100001
CUSTOMER#100002
CUSTOMER#100003
```

The keys provide both:

```text
Identity
+
Distribution
```

This is much better than a design where almost all traffic targets the same partition-key value.

---

# 30. Real-Time Scenario — One Extremely Popular Product

Suppose:

```text
Product P001
```

is extremely popular.

Millions of requests target:

```text
PK = PRODUCT#P001
```

This can create a concentrated workload.

You should recognize this as a potential **hot-key/hot-partition** design problem.

Potential solutions depend on the workload, but can include:

```text
Write sharding
Caching
Alternative access patterns
Workload distribution
```

The correct solution depends on whether the workload is read-heavy, write-heavy, or both.

---

# 31. Real-Time Scenario — Leaderboard

Requirement:

```text
Get top players
Get player score
Get players in ranking range
```

You need to design the keys around these access patterns.

A naive design might not support all three efficiently.

This is where you ask:

```text
What is the primary access pattern?
What is the secondary access pattern?
Do I need a GSI?
How should the sort key represent ranking?
```

The lesson:

> **Don't design the table first and discover access patterns later.**

---

# 32. Data Modeling Workflow

Use this workflow in real projects:

```text
Step 1
   ↓
List business requirements

Step 2
   ↓
Identify access patterns

Step 3
   ↓
Identify entities

Step 4
   ↓
Design primary keys

Step 5
   ↓
Design sort keys

Step 6
   ↓
Check Query/GetItem support

Step 7
   ↓
Identify missing access patterns

Step 8
   ↓
Consider GSI/LSI

Step 9
   ↓
Check partition-key distribution

Step 10
   ↓
Test performance + cost
```

---

# 33. DynamoDB vs Traditional Normalization

### RDS mindset

```text
Normalize data
      ↓
Reduce duplication
      ↓
Use joins
      ↓
Query dynamically
```

### DynamoDB mindset

```text
Identify access patterns
      ↓
Design keys
      ↓
Group related data
      ↓
Optimize reads/writes
      ↓
Accept deliberate duplication when useful
```

This is one of the biggest conceptual shifts when moving from RDS to DynamoDB.

---

# 34. Is Data Duplication Allowed?

Yes.

DynamoDB designs may intentionally duplicate data.

For example, if an application needs to retrieve:

```text
Customer Name
+
Order Details
```

frequently, duplicating selected customer information into an order item may sometimes simplify reads.

But duplication creates a consistency responsibility.

Therefore ask:

```text
Is the duplicated data authoritative?
How is it updated?
What happens if updates fail?
Is eventual consistency acceptable?
```

Don't duplicate data blindly.

---

# 35. Denormalization

DynamoDB commonly uses **denormalization**.

Instead of:

```text
Customer
   ↓
Order
   ↓
Product
```

requiring multiple database operations, you may structure items to support the application's primary reads more directly.

### Benefit

Fewer database round trips.

### Cost

Potentially:

```text
More storage
More write complexity
Duplicate data
Consistency considerations
```

You trade relational normalization for access efficiency.

---

# 36. The "Single Table" Trap

Interview question:

> "Does DynamoDB always require single-table design?"

**Answer: No.**

Single-table design is a powerful pattern, not a mandatory rule.

Use it when it simplifies and optimizes the application's access patterns.

Don't force everything into one table simply because:

> "DynamoDB means one table."

That's incorrect.

---

# 37. The "One Table Per Entity" Trap

Another common mistake:

```text
Customers
Orders
Products
Payments
Addresses
Reviews
```

and then treating DynamoDB exactly like RDS.

This isn't necessarily wrong, but it can miss DynamoDB's strengths.

You should evaluate whether related access patterns can be served more efficiently through composite keys and item collections.

---

# 38. Production Data-Model Review Checklist

Before approving a DynamoDB model, ask:

### Access

1. What are the top application access patterns?
2. Can each critical access pattern use GetItem or Query?
3. Are any critical paths dependent on Scan?

### Keys

4. Is the partition key well distributed?
5. Is the sort key useful for required queries?
6. Could a hot key develop?

### Indexes

7. Do we really need each GSI?
8. What access pattern does each index support?
9. What is the additional storage/capacity impact?

### Operations

10. What happens as traffic increases?
11. What happens when one key becomes extremely popular?
12. What happens during failure or recovery?

---

# 39. Top Interview Questions — Lesson 2

### Q1. Why is access-pattern-first design important in DynamoDB?

Because DynamoDB is optimized around key-based access rather than arbitrary relational queries. The keys should be designed to support the application's known read/write patterns efficiently.

---

### Q2. What is the difference between a simple and composite primary key?

```text
Simple:
Partition Key

Composite:
Partition Key + Sort Key
```

A composite key allows multiple items to share the same partition-key value while remaining uniquely identified by the combined key.

---

### Q3. What is an item collection?

A group of items that share the same partition-key value in a table with a composite primary key.

---

### Q4. Why are sort keys important?

They allow multiple related items to share a partition key and provide ordering/range/prefix-based access within that logical group.

---

### Q5. Why should you avoid Scan for normal application traffic?

Because Scan can examine a large amount of data, consuming resources and increasing latency/cost compared with targeted key-based access.

---

### Q6. What is single-table design?

A DynamoDB modeling pattern where multiple entity types are represented within one table using carefully designed partition and sort keys.

---

### Q7. Is denormalization normal in DynamoDB?

Yes. Deliberate duplication can reduce database round trips and improve access efficiency, but it introduces additional consistency and update considerations.

---

### Q8. When should you use a GSI?

When an important access pattern cannot be efficiently supported by the table's primary key and an alternative key structure is required.

---

# 🧠 Lesson 2 — The Mental Model

Memorize this sequence:

```text
                    BUSINESS REQUIREMENTS
                             │
                             ▼
                     ACCESS PATTERNS
                             │
                             ▼
                       PRIMARY KEY
                        /        \
                       /          \
              PARTITION KEY     SORT KEY
                       │            │
                       └─────┬──────┘
                             ▼
                       ITEM COLLECTION
                             │
                    ┌────────┴────────┐
                    ▼                 ▼
                 Query             GetItem
                    │
                    ▼
             Efficient Access
                    │
              ┌─────┴─────┐
              ▼           ▼
             GSI         GSI
              │
              ▼
       Additional Access Path
```

## 🔥 The single sentence to remember

> **In DynamoDB, don't design your database first and then figure out how to query it; identify the application's access patterns first and design the keys to make those access patterns efficient.**

**Next: Lesson 3 — Read/Write Operations, Query vs Scan, Consistency Models, Conditional Operations & DynamoDB API behavior.**
