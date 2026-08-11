# ⚡ Amazon DynamoDB — Lesson 5

## Indexes & Advanced Querying

This lesson covers one of the most important DynamoDB production topics:

> **How do you support new access patterns without redesigning the entire base table?**

The core concepts are:

```text
Primary Key
   ↓
GSI
   ↓
LSI
   ↓
Projection
   ↓
Sparse Index
   ↓
Querying Indexes
   ↓
Index Capacity
   ↓
Advanced Access Patterns
```

---

# 1. Why Do We Need Indexes?

DynamoDB is designed around **access patterns**.

Suppose your table's primary key is:

```text
PK = CustomerId
SK = OrderId
```

You can efficiently find:

```text
Customer → Orders
```

But now the business asks:

> "Find all orders for a particular customer status."

For example:

```text
Status = PENDING
```

The base table key may not support this efficiently.

You could Scan the table—but that is usually a poor production design.

Instead, you can create a **secondary index**.

---

# 2. DynamoDB Secondary Indexes

DynamoDB provides two types:

```text
Secondary Indexes
       │
   ┌───┴────┐
   ▼        ▼
  GSI      LSI
```

### GSI

**Global Secondary Index**

### LSI

**Local Secondary Index**

Both provide alternative ways to query your data.

But their capabilities and constraints are different.

---

# 3. GSI — Global Secondary Index

A GSI allows you to create an alternative key structure.

Suppose base table:

```text
PK = CustomerId
SK = OrderId
```

You create:

```text
GSI1PK = Status
GSI1SK = OrderDate
```

Now you can query:

```text
Status = PENDING
```

and potentially sort/filter through the index sort key.

Conceptually:

```text
Base Table
CustomerId + OrderId
       │
       │
       ▼
     GSI
Status + OrderDate
```

---

# 4. Why Is It Called "Global"?

It is called global because the index's partition-key distribution is independent of the base table's partition-key distribution.

For example:

```text
Base Table:

CustomerId
   ↓
CUSTOMER#001
CUSTOMER#002
CUSTOMER#003
```

GSI:

```text
Status
   ↓
PENDING
SHIPPED
DELIVERED
```

The GSI has its own key structure and access pattern.

---

# 5. GSI Example

Suppose your base table contains:

```text
PK             SK
────────────────────────────
CUSTOMER#001   ORDER#100
CUSTOMER#001   ORDER#101
CUSTOMER#002   ORDER#102
CUSTOMER#003   ORDER#103
```

Attributes:

```text
Status
OrderDate
Amount
```

You want:

> Find all pending orders.

Create:

```text
GSI1PK = Status
GSI1SK = OrderDate
```

Then:

```text
Query GSI
GSI1PK = PENDING
```

This creates a direct access path for the requirement.

---

# 6. GSI Is an Access-Pattern Tool

Do not think:

> "I need an index because indexes are good."

Think:

> **"I have a specific query that my current key design cannot support efficiently."**

Example:

```text
Requirement:
Find orders by customer
```

Base table:

```text
PK = CustomerId
```

No GSI needed.

But:

```text
Requirement:
Find orders by Email
```

If Email isn't part of the key:

```text
Email GSI
```

may be appropriate.

---

# 7. GSI Partition Key

A GSI has its own partition key.

Example:

```text
Base:
PK = CustomerId
SK = OrderId

GSI:
PK = Email
SK = OrderDate
```

This allows:

```text
Query GSI
Email = alice@example.com
```

The base table and GSI can therefore support different access patterns.

---

# 8. GSI Sort Key

A GSI can optionally have a sort key.

Example:

```text
GSI PK = CustomerId
GSI SK = OrderDate
```

Now you can query:

```text
CustomerId = C001
```

and use sort-key conditions such as:

```text
OrderDate >= ...
OrderDate BETWEEN ...
begins_with(...)
```

where applicable.

---

# 9. GSI Without Sort Key

A GSI can have only a partition key.

Example:

```text
GSI1PK = Email
```

This is useful when the access pattern is simply:

```text
Find user by email
```

For example:

```text
Query:
Email = alice@example.com
```

If you don't need ordering/range conditions, a sort key may not be necessary.

---

# 10. GSI With Sort Key

Use a sort key when you need an additional dimension.

Example:

```text
GSI1PK = CustomerId
GSI1SK = CreatedAt
```

This supports:

```text
Customer's orders
+
ordered by creation time
```

This is a very common design.

---

# 11. GSI Projection

A GSI doesn't necessarily need to contain every attribute from the base table.

You choose what attributes are projected.

Common projection choices include:

```text
KEYS_ONLY
INCLUDE
ALL
```

---

# 12. KEYS_ONLY Projection

With:

```text
Projection = KEYS_ONLY
```

the index contains key attributes needed to identify the item, rather than all non-key attributes.

Conceptually:

```text
Base Item
 ├── CustomerId
 ├── OrderId
 ├── Status
 ├── Address
 ├── Notes
 └── Metadata

GSI
 ├── Index keys
 └── Table keys
```

This can keep the index smaller.

---

# 13. INCLUDE Projection

With:

```text
Projection = INCLUDE
```

you specify additional non-key attributes that the index should contain.

For example:

```text
GSI
 ├── CustomerId
 ├── OrderDate
 ├── OrderAmount
 └── Status
```

You might choose this when the application needs those attributes directly from the index.

---

# 14. ALL Projection

With:

```text
Projection = ALL
```

the index contains all attributes from the base item.

This is convenient, but can make the index larger.

Larger indexes can mean:

```text
More storage
More index write work
Potentially more capacity consumption
```

Therefore:

> **Don't automatically choose ALL.**

---

# 15. Projection Decision

Think:

```text
Does my query need only keys?
       │
      YES
       ↓
  KEYS_ONLY

Does it need a few attributes?
       │
      YES
       ↓
   INCLUDE

Does it need the entire item?
       │
      YES
       ↓
      ALL
```

The goal is to balance:

```text
Read efficiency
+
Index size
+
Write cost
+
Application simplicity
```

---

# 16. GSI Query

Suppose:

```text
GSI1PK = CustomerId
GSI1SK = OrderDate
```

Query:

```text
CustomerId = C001
```

The query is performed against the index.

Conceptually:

```text
Application
    ↓
Query GSI
    ↓
GSI1PK = C001
    ↓
Matching index items
```

This avoids scanning the entire base table.

---

# 17. Querying GSI vs Base Table

Base table:

```text
PK = CustomerId
SK = OrderId
```

GSI:

```text
PK = Email
SK = OrderDate
```

Then:

```text
CustomerId lookup
      ↓
Base table Query

Email lookup
      ↓
GSI Query
```

This is the essence of secondary indexes:

> **One dataset, multiple efficient access paths.**

---

# 18. GSI Eventual Consistency

A major interview point:

> **GSI reads are eventually consistent.**

You cannot request strongly consistent reads from a GSI.

So if your application requires strongly consistent reads for a particular access pattern, you need to design around that constraint rather than simply adding a GSI.

---

# 19. GSI Propagation

When you write to the base table:

```text
PutItem
   ↓
Base Table
   ↓
GSI update
```

The GSI is maintained asynchronously.

Therefore, immediately after a write, a GSI query may not reflect that write yet.

Example:

```text
T0:
Create Order#100

T1:
Query GSI

Potentially:
Order#100 not visible yet
```

This is an important distributed-system behavior.

---

# 20. GSI and Write Capacity

Suppose:

```text
Base table
+
GSI1
+
GSI2
```

Application performs:

```text
PutItem
```

The write can cause updates to:

```text
Base table
GSI1
GSI2
```

Therefore:

> **Indexes can increase write-related capacity consumption and cost.**

This is one reason you shouldn't create unnecessary GSIs.

---

# 21. GSI Hot Partition

A GSI can have its own hot-key problem.

Suppose:

```text
GSI PK = Status
```

and most records are:

```text
Status = ACTIVE
```

Then:

```text
ACTIVE
   ↓
Huge number of items
Huge traffic
```

This can create a concentrated workload.

Remember:

> **A well-designed base table does not automatically mean a well-designed GSI.**

---

# 22. GSI Partition-Key Cardinality

A good GSI partition key generally needs enough distribution for the workload.

Bad example:

```text
GSI PK = Country
```

if:

```text
90% = USA
```

Potentially problematic.

Better candidates might be attributes with greater distribution, depending on the actual access pattern.

But:

> **High cardinality alone doesn't make a good key.**

The key must support the required query.

---

# 23. GSI Sparse Index

One of the most powerful DynamoDB patterns is a **sparse GSI**.

An item appears in a GSI only when it contains the attributes that define the index key.

Suppose:

```text
GSI PK = EscalationStatus
```

Only urgent tickets have:

```text
EscalationStatus = OPEN
```

Then the GSI contains only those urgent items.

Conceptually:

```text
Base Table
100 million items
       │
       ▼
Sparse GSI
100,000 urgent items
```

This can be extremely useful.

---

# 24. Sparse Index Example

Base table:

```text
Ticket
 ├── T001 → normal
 ├── T002 → normal
 ├── T003 → urgent
 ├── T004 → normal
 └── T005 → urgent
```

GSI:

```text
EscalationStatus = OPEN
```

Only:

```text
T003
T005
```

might appear.

Now the application can efficiently query:

```text
All currently escalated tickets
```

without scanning the entire ticket table.

---

# 25. LSI — Local Secondary Index

Now let's look at the second index type:

> **Local Secondary Index**

An LSI uses the **same partition key as the base table**, but provides a different sort key.

Example base table:

```text
PK = CustomerId
SK = OrderId
```

LSI:

```text
PK = CustomerId
SK = OrderDate
```

Notice:

```text
Partition key = SAME
Sort key = DIFFERENT
```

---

# 26. LSI Mental Model

Think:

```text
BASE TABLE
CustomerId + OrderId
       │
       ▼
LSI
CustomerId + OrderDate
```

The partition remains the same.

Only the sort-key access pattern changes.

---

# 27. GSI vs LSI

Memorize this table:

| Feature                      | GSI                                | LSI                                             |
| ---------------------------- | ---------------------------------- | ----------------------------------------------- |
| Partition key                | Can be different                   | Must be same as base                            |
| Sort key                     | Optional                           | Required                                        |
| Created after table creation | Yes                                | No — must be created with table                 |
| Strongly consistent reads    | No                                 | Yes, supported                                  |
| Capacity                     | Separate capacity model            | Shares table's capacity                         |
| Partition-key distribution   | Independent                        | Same as base                                    |
| Item collection limit        | No 10 GB LSI item-collection limit | 10 GB per partition-key item collection         |
| Typical use                  | New access pattern                 | Alternate sort order/range under same partition |

The distinction is extremely important.

---

# 28. LSI Example

Suppose:

```text
CustomerId = C001
```

Orders:

```text
OrderId
OrderDate
Amount
```

Base table:

```text
C001 + OrderId
```

You want:

> "Get this customer's orders ordered by amount."

An LSI could use:

```text
PK = CustomerId
SK = Amount
```

Now the same customer partition supports an alternative sort-key view.

---

# 29. LSI Creation Restriction

One major difference:

> **LSIs must be created when the table is created.**

You cannot simply decide months later:

```text
"Oh, let's add an LSI."
```

to an existing table.

This is one reason GSIs are more commonly used for evolving applications.

---

# 30. LSI 10 GB Limit

LSIs have an important constraint:

> **The total size of items sharing a partition key across the base table and its LSIs is limited to 10 GB.**

This means you need to be careful when a single partition-key value can accumulate a huge amount of data.

Example:

```text
PK = CUSTOMER#001
```

with millions of large items could eventually create an issue.

---

# 31. GSI vs LSI — Interview Shortcut

Remember:

```text
GSI
=
Different partition key possible
+
Can add later
+
Eventually consistent reads only

LSI
=
Same partition key
+
Different sort key
+
Must exist at table creation
+
Strong consistency supported
+
10 GB item-collection limit
```

This answer covers most interview questions.

---

# 32. Advanced Query — KeyConditionExpression

When querying DynamoDB, distinguish:

```text
KeyConditionExpression
```

from:

```text
FilterExpression
```

Key conditions determine which items DynamoDB accesses through the key structure.

Example:

```text
PK = CUSTOMER#001
AND
SK BETWEEN ORDER#100 AND ORDER#200
```

This is fundamentally different from filtering arbitrary attributes after the key lookup.

---

# 33. FilterExpression

A filter can remove items from the result after DynamoDB evaluates the relevant key-based items.

Example:

```text
PK = CUSTOMER#001
Filter:
Status = PENDING
```

Conceptually:

```text
Query matching key
        ↓
Evaluate items
        ↓
Apply filter
        ↓
Return matching results
```

Therefore:

> **FilterExpression is not a substitute for a good key condition.**

---

# 34. KeyCondition vs Filter

This is a common interview question.

### KeyConditionExpression

Used for:

```text
Partition key
+
Supported sort-key conditions
```

### FilterExpression

Used to remove unwanted items after the key-based operation.

Therefore:

```text
GOOD:
PK = C001
SK BETWEEN X AND Y

LESS IDEAL:
PK = C001
Filter = arbitrary attribute
```

if the filter eliminates most of the data.

---

# 35. begins_with()

A very useful DynamoDB sort-key function is:

```text
begins_with()
```

Example:

```text
SK = ORDER#2026-08-...
```

You could design the sort key to support:

```text
begins_with(SK, "ORDER#2026-08")
```

This is why sort-key structure matters.

---

# 36. BETWEEN

Suppose:

```text
SK = timestamp
```

You can query:

```text
BETWEEN start AND end
```

This is useful for:

```text
Orders in date range
Events in time range
Logs in time range
Messages in time range
```

Time-oriented sort keys are therefore extremely powerful.

---

# 37. ScanIndexForward

DynamoDB Query supports ordering through the sort key.

Conceptually:

```text
ScanIndexForward = true
```

returns results in ascending sort-key order.

And:

```text
ScanIndexForward = false
```

returns them in descending order.

For example:

```text
Newest orders first
```

can often be implemented by querying the sort key in descending order.

---

# 38. Limit

You can use a limit to control the number of items evaluated/returned per request semantics.

For example:

```text
Get latest 20 orders
```

can be designed around:

```text
Query
+
ScanIndexForward = false
+
Limit
```

This is much better than retrieving thousands of orders and sorting them in the application.

---

# 39. Pagination with Indexes

Large GSI queries can also require pagination.

Conceptually:

```text
Query GSI
   ↓
Page 1
   ↓
LastEvaluatedKey
   ↓
Page 2
   ↓
LastEvaluatedKey
   ↓
Page 3
```

Your application should not assume:

> "One Query call returns everything."

---

# 40. Projection vs Filter

These solve different problems.

### Projection

Controls:

> **Which attributes are returned/stored in the index projection.**

### Filter

Controls:

> **Which evaluated items remain in the response.**

Don't confuse them.

---

# 41. Example — Customer Search

Requirement:

> Find customer by email.

Base table:

```text
PK = CustomerId
```

You don't know CustomerId.

Bad:

```text
Scan
Filter email = x@example.com
```

Better:

```text
GSI
PK = Email
```

Then:

```text
Query GSI
Email = x@example.com
```

This is the classic secondary-index use case.

---

# 42. Example — Orders by Status

Requirement:

> Find all pending orders created today.

Possible GSI:

```text
GSI PK = Status
GSI SK = OrderDate
```

Query:

```text
Status = PENDING
AND
OrderDate BETWEEN start AND end
```

Now DynamoDB can use the index to target the access pattern.

---

# 43. Example — User's Recent Orders

Base table:

```text
PK = UserId
SK = OrderId
```

Requirement:

> Get user's latest orders.

Potential LSI/GSI design:

```text
PK = UserId
SK = OrderDate
```

Then:

```text
Query
ScanIndexForward = false
Limit = 20
```

This can efficiently retrieve the latest items without application-side sorting of the entire collection.

---

# 44. Example — Open Support Tickets

Requirement:

> Find all open/escalated tickets.

Instead of:

```text
Scan entire table
Filter Status = OPEN
```

consider a sparse GSI:

```text
GSI PK = Status
```

where only relevant records carry the indexed attribute.

Then:

```text
Query GSI
Status = OPEN
```

This can dramatically reduce unnecessary data access.

---

# 45. Index Overuse

A common mistake is:

```text
Every attribute
   ↓
Create GSI
```

Example:

```text
GSI_Email
GSI_Name
GSI_City
GSI_Status
GSI_Phone
GSI_Date
GSI_Type
GSI_Category
...
```

This can create:

```text
Higher cost
More write work
More operational complexity
More storage
More opportunities for hot keys
```

Indexes should exist because they support real access patterns.

---

# 46. Index Design Checklist

Before creating a GSI, ask:

```text
1. What exact query does it support?
2. What is the partition key?
3. Is the partition key well distributed?
4. Do I need a sort key?
5. What attributes need projection?
6. Will this GSI be write-heavy?
7. Could it become a hot key?
8. Is a sparse index appropriate?
9. What consistency does the application require?
10. Is the index actually necessary?
```

---

# 47. Advanced Access Pattern Design

Consider this application:

```text
E-commerce
```

Requirements:

```text
1. Get order by order ID
2. Get customer's orders
3. Find orders by status
4. Find orders by date
5. Find high-value orders
```

Don't immediately create five indexes.

Instead, analyze the actual query patterns and determine whether one or more carefully designed keys/indexes can support multiple requirements.

DynamoDB rewards **intentional modeling**.

---

# 48. GSI Write Cost Example

Suppose:

```text
Base item = 2 KB
```

and the item is projected into a GSI.

A write to the base table can result in write capacity consumption for both:

```text
Base table
+
GSI
```

The exact amount depends on the size of the data written to each resource and DynamoDB's capacity rules.

Therefore:

> **A GSI isn't just a read optimization; it can affect write capacity and cost.**

---

# 49. GSI Storage Considerations

Suppose your base table contains:

```text
100 GB
```

and you create several large projected indexes.

Your total DynamoDB storage footprint can become substantially larger than the base table alone.

Therefore:

```text
Base storage
+
Index storage
=
Total storage footprint
```

Projection decisions matter.

---

# 50. Real-Time Scenario — Production Search Requirement

### Requirement

The application needs:

```text
GET /users?email=alice@example.com
```

### Current table

```text
PK = UserId
```

### Bad implementation

```text
Scan
Filter Email
```

### Better design

```text
GSI:
PK = Email
```

Then:

```text
Query GSI
Email = alice@example.com
```

### Production lesson

> **When a query is frequent and predictable, model the data for that query rather than relying on Scan.**

---

# 51. Real-Time Scenario — GSI Becomes Hot

Suppose:

```text
GSI PK = Status
```

Values:

```text
OPEN
CLOSED
```

and:

```text
95% of traffic → OPEN
```

Now the GSI has concentrated workload.

Potential solutions depend on the access pattern:

```text
Better partition-key design
Composite/sharded key
Caching
Alternative access path
Workload redistribution
```

Don't simply assume:

> "The base table is fine, so the GSI must be fine."

---

# 52. Real-Time Scenario — Need Strong Consistency

Requirement:

> "The read must reflect the latest value."

You propose:

```text
GSI
```

But GSI reads are eventually consistent.

Therefore the design must change.

Possible approaches include:

```text
Query the base table when its key supports the access pattern
```

or:

```text
Redesign the data model
```

depending on the requirement.

This is a key architecture constraint.

---

# 53. Real-Time Scenario — LSI vs GSI

Requirement:

```text
Same CustomerId
Different ordering criteria
```

Example:

```text
CustomerId + OrderId
CustomerId + OrderDate
```

An LSI can be a candidate because:

```text
Partition key stays the same.
Sort key changes.
```

But if the access pattern requires:

```text
Email + OrderDate
```

then an LSI cannot provide that because its partition key must remain the base table's partition key.

A GSI would be the relevant mechanism.

---

# 54. Real-Time Scenario — Sparse GSI

Requirement:

> "Show only customers whose account requires manual review."

Instead of indexing every customer with:

```text
ReviewStatus = NORMAL
```

you can design the index so only customers requiring review have the indexed attribute.

Then:

```text
GSI
PK = REVIEW_REQUIRED
```

contains only relevant items.

This is a powerful operational pattern for:

```text
Queues
Alerts
Exceptions
Escalations
Pending workflows
```

---

# 55. Real-Time Scenario — Too Many GSIs

A team creates:

```text
12 GSIs
```

because:

> "We might need them later."

Production impact can include:

```text
More storage
Higher write overhead
More capacity requirements
More monitoring
More complexity
Potential hot indexes
```

Better approach:

> **Create indexes based on concrete access patterns and measured requirements.**

---

# 56. GSI Backfill

When adding a GSI to an existing table, DynamoDB must populate the index from existing data.

Conceptually:

```text
Existing Table
100 million items
       ↓
Create GSI
       ↓
Index population/backfill
       ↓
GSI becomes usable
```

For large production tables, this operation should be planned carefully because it can affect resource consumption and operational behavior.

---

# 57. Index Monitoring

For production GSIs, monitor:

```text
Consumed capacity
Throttling
Latency
Traffic
Storage
Key distribution
```

If an index becomes hot, investigate:

```text
GSI partition key
Traffic distribution
Query pattern
Write pattern
```

---

# 58. Query vs Scan — Final Comparison

| Feature                     | Query                         | Scan                                                     |
| --------------------------- | ----------------------------- | -------------------------------------------------------- |
| Uses key                    | Yes                           | No targeted key requirement                              |
| Targeted access             | Yes                           | No                                                       |
| Good for production APIs    | Usually                       | Usually not                                              |
| Can use GSI                 | Yes                           | Yes                                                      |
| Can use sort-key conditions | Yes                           | No key-condition equivalent                              |
| Large datasets              | Much better when well modeled | Potentially expensive                                    |
| Typical use                 | Normal application access     | Admin jobs, analytics-like operations, exceptional cases |

---

# 59. GSI vs LSI — Final Comparison

```text
                  SECONDARY INDEX
                         │
               ┌─────────┴─────────┐
               ▼                   ▼
              GSI                 LSI
               │                   │
        Different PK          Same PK
               │                   │
        Optional SK           Required SK
               │                   │
        Add later             Create with table
               │                   │
      Eventual consistency   Strong consistency possible
               │                   │
       Independent key       Same partition grouping
```

Memorize this diagram.

---

# 60. Lesson 5 — Production Decision Tree

When you need a new query:

```text
Need new access pattern
        ↓
Can base-table PK support it?
        │
       YES
        ↓
   Query base table

        NO
        ↓
Can a GSI support it?
        │
       YES
        ↓
   Design GSI
        │
        ├── Choose PK
        ├── Choose SK
        ├── Choose projection
        └── Check distribution

        NO
        ↓
Reconsider data model
```

For an alternative sort order **within the same partition key**, consider whether an LSI is appropriate when the table's creation-time constraints allow it.

---

# 61. Top 25 Interview Questions — Lesson 5

### 1. What is a GSI?

A Global Secondary Index provides an alternate key structure that can use a different partition key from the base table.

### 2. What is an LSI?

A Local Secondary Index uses the same partition key as the base table but a different sort key.

### 3. Biggest difference between GSI and LSI?

**GSI can have a different partition key; LSI must use the base table's partition key.**

### 4. Can you add a GSI later?

Yes.

### 5. Can you add an LSI later?

No. LSIs must be defined when the table is created.

### 6. Can GSI reads be strongly consistent?

No.

### 7. Can LSI reads be strongly consistent?

Yes, where supported by the operation.

### 8. What is projection?

The set of attributes stored in an index for an indexed item.

### 9. What are common projection types?

```text
KEYS_ONLY
INCLUDE
ALL
```

### 10. What is a sparse GSI?

An index containing only items that have the attributes required to appear in the index.

### 11. Why use a sparse GSI?

To efficiently query a small subset of items such as open tickets, escalations, or pending workflows.

### 12. Does a GSI affect writes?

Yes. Writes that affect indexed data can require index updates and additional capacity.

### 13. Can a GSI become hot?

Yes.

### 14. Can a base table be healthy while its GSI has a problem?

Yes.

### 15. What is a FilterExpression?

A condition used to remove items from the result after the key-based operation evaluates them.

### 16. Is FilterExpression a replacement for a key condition?

No.

### 17. What is KeyConditionExpression?

The key-based condition used by Query to select items using the partition key and supported sort-key conditions.

### 18. What does begins_with() do?

It allows a supported sort-key query to match values beginning with a specified prefix.

### 19. What does ScanIndexForward do?

Controls ascending or descending sort-key traversal for Query results.

### 20. Why shouldn't every attribute have a GSI?

Because indexes increase storage, write work, capacity requirements, complexity, and potential hot-key risks.

### 21. When would you choose an LSI?

When you need an alternate sort-key access pattern while retaining the same partition key and the LSI's creation-time/size constraints are acceptable.

### 22. When would you choose a GSI?

When you need a new access pattern that may require a different partition key.

### 23. What is the LSI 10 GB limitation?

An item collection sharing a partition key has a 10 GB limit across the base table and its LSIs.

### 24. Why can a GSI query return stale data?

Because GSI propagation is asynchronous and GSI reads are eventually consistent.

### 25. What is the most important index principle?

> **Create indexes to support known access patterns, not simply because more indexes appear to provide more flexibility.**

---

# 🧠 Lesson 5 — Final Mental Model

```text
                         DYNAMODB DATA
                              │
                              ▼
                       ACCESS PATTERNS
                              │
             ┌────────────────┼────────────────┐
             ▼                ▼                ▼
        Base Table           GSI              LSI
             │                │                │
       Same PK/SK        New PK/SK         Same PK
                                             New SK
             │                │                │
             └────────────────┼────────────────┘
                              ▼
                            QUERY
                              │
                 ┌────────────┼────────────┐
                 ▼            ▼            ▼
             KeyCondition   Filter      Projection
                 │            │            │
                 ▼            ▼            ▼
             Targeted      Remove       Return/
             access        results      index attrs
                 │
                 ▼
             Performance
                 │
       ┌─────────┼──────────┐
       ▼         ▼          ▼
   Distribution Capacity  Cost
       │         │          │
       └─────────┼──────────┘
                 ▼
             Production
```

# 🔥 Lesson 5 Golden Rules

1. **DynamoDB indexes exist to support access patterns.**
2. **GSI = alternate partition key; LSI = same partition key with alternate sort key.**
3. **GSI can be added later; LSI must be created with the table.**
4. **GSI reads are eventually consistent; LSI can support strong consistency.**
5. **Projection affects what data is stored in an index.**
6. **KEYS_ONLY, INCLUDE, and ALL have different cost/storage implications.**
7. **Sparse GSIs are excellent for exception/queue/escalation-style workloads.**
8. **GSIs can introduce their own hot-key and throttling problems.**
9. **FilterExpression doesn't replace good key design.**
10. **Avoid creating indexes "just in case."**
11. **A GSI can increase write capacity consumption.**
12. **Always design the index around the actual query the application needs.**

## 🎯 Lesson 5 takeaway

The production mindset is:

> **Access Pattern → Key Design → GSI/LSI Decision → Projection → Distribution → Capacity → Query → Monitor**

**Next: Lesson 6 — DynamoDB Streams, Event-Driven Architecture, TTL, Lambda Integration & CDC: how to capture item-level changes and build real-time workflows around DynamoDB.**
