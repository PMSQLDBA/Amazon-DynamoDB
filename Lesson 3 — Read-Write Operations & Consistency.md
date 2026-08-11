# ⚡ Amazon DynamoDB — Lesson 3

## Read/Write Operations & Consistency

This lesson focuses on **how applications actually interact with DynamoDB**.

The key concepts are:

> **PutItem → GetItem → UpdateItem → DeleteItem → Query → Scan → Batch Operations → Consistency**

---

# 1. DynamoDB API Operations — Big Picture

The most important operations are:

```text
id="dy1"
Write Operations
├── PutItem
├── UpdateItem
├── DeleteItem
└── BatchWriteItem

Read Operations
├── GetItem
├── Query
├── Scan
└── BatchGetItem

Advanced
├── Conditional Writes
├── Transactions
└── PartiQL
```

For production applications, you should understand **why** you use each operation, not just memorize its name.

---

# 2. PutItem

`PutItem` creates a new item or replaces an existing item with the same primary key.

Example:

```json
{
  "CustomerId": "C001",
  "Name": "Alice",
  "Country": "USA"
}
```

Conceptually:

```text
Application
    ↓
PutItem
    ↓
DynamoDB
    ↓
Item stored
```

### Important behavior

If an item with the same primary key already exists, a normal `PutItem` can replace the existing item.

Therefore:

> **Don't use PutItem blindly when you must protect an existing record.**

Conditional expressions can be used when you need safeguards.

---

# 3. PutItem — Real-Time Example

Suppose:

```text
PK = CUSTOMER#001
SK = PROFILE
```

You execute:

```text
PutItem
```

with:

```text
Name = Alice
Country = USA
```

The item becomes:

```text
PK              SK
────────────────────────
CUSTOMER#001    PROFILE
```

with its attributes.

If you later put another item using the same key:

```text
PK = CUSTOMER#001
SK = PROFILE
```

you need to understand whether you're intentionally replacing/updating that item.

---

# 4. Conditional PutItem

Suppose your requirement is:

> "Create this customer only if the customer doesn't already exist."

You can use a condition such as:

```text
attribute_not_exists(PK)
```

Conceptually:

```text
PutItem
   │
   ▼
Does PK already exist?
   │
 ┌─┴─────────┐
No           Yes
│             │
▼             ▼
Create       Reject
```

This prevents accidental overwriting.

---

# 5. GetItem

`GetItem` retrieves a specific item using its primary key.

For a simple primary key:

```text
PK = CUSTOMER#001
```

For a composite key:

```text
PK = CUSTOMER#001
SK = PROFILE
```

Conceptually:

```text
Application
    ↓
GetItem
    ↓
Primary Key
    ↓
Specific Item
```

This is one of the most efficient DynamoDB access patterns.

---

# 6. GetItem vs Query

This is extremely important.

### GetItem

Use when you know the **complete primary key** of one item.

```text
PK + SK
   ↓
One specific item
```

### Query

Use when you want multiple items associated with a partition-key value.

```text
PK
 ↓
Multiple matching items
```

Example:

```text
GetItem:
CUSTOMER#001 + ORDER#1001

Query:
CUSTOMER#001
```

The second can return multiple orders.

---

# 7. UpdateItem

`UpdateItem` modifies attributes of an existing item.

For example:

```text
Before:

Name = Alice
Status = ACTIVE
Points = 100
```

You might update:

```text
Points = 150
```

without rewriting the entire item.

Conceptually:

```text
Existing Item
     ↓
UpdateItem
     ↓
Modified Attributes
```

---

# 8. Why UpdateItem Is Important

Suppose an item contains:

```text
Name
Email
Address
Phone
Preferences
Orders
```

You only want to change:

```text
Phone
```

An update operation can modify the relevant attribute rather than requiring you to reconstruct the entire logical object.

This is useful for:

```text
Counters
Status changes
Timestamps
Profile updates
Inventory changes
```

---

# 9. Update Expressions

DynamoDB supports update expressions.

Common operations include:

```text
SET
REMOVE
ADD
DELETE
```

For example, conceptually:

```text
SET Status = "SHIPPED"
```

or:

```text
SET UpdatedAt = current_time
```

This allows targeted changes.

---

# 10. Atomic Counters

A common DynamoDB pattern is maintaining counters.

Example:

```text
Product stock = 100
```

You want:

```text
Stock = Stock - 1
```

Instead of:

```text
Read Stock
   ↓
Application calculates Stock - 1
   ↓
Write Stock
```

you can use an update operation that modifies the value atomically.

This reduces certain race-condition problems.

---

# 11. DeleteItem

`DeleteItem` removes an item identified by its primary key.

Example:

```text
PK = CUSTOMER#001
SK = SESSION#ABC
```

Conceptually:

```text
Application
    ↓
DeleteItem
    ↓
Key
    ↓
Item removed
```

As with writes, conditional expressions can protect against deleting something unexpectedly.

---

# 12. Conditional Delete

Suppose you want:

> Delete the order only if its status is `CANCELLED`.

Conceptually:

```text
DeleteItem
Condition:
Status = CANCELLED
```

If:

```text
Status = CANCELLED
```

the deletion succeeds.

If:

```text
Status = SHIPPED
```

the operation fails.

This is an important production safety pattern.

---

# 13. Query — The Most Important Read Operation

`Query` retrieves items based on a partition-key value.

Suppose:

```text
PK = CUSTOMER#001
```

and the table contains:

```text
PK              SK
────────────────────────
CUSTOMER#001    PROFILE
CUSTOMER#001    ORDER#001
CUSTOMER#001    ORDER#002
CUSTOMER#001    ORDER#003
```

A Query can retrieve items belonging to:

```text
CUSTOMER#001
```

---

# 14. Query with Sort-Key Conditions

Query becomes even more powerful with a sort key.

Suppose:

```text
PK = CUSTOMER#001
```

and:

```text
SK
──────────────────────
ORDER#001
ORDER#002
ORDER#003
ORDER#004
```

You can use supported sort-key conditions such as:

```text
=
<
<=
>
>=
BETWEEN
begins_with
```

depending on the access pattern.

---

# 15. Example — Recent Orders

Suppose your sort key is time-oriented:

```text
PK = CUSTOMER#001

SK
──────────────────────────────
ORDER#2026-08-08#001
ORDER#2026-08-09#002
ORDER#2026-08-10#003
ORDER#2026-08-11#004
```

You can design queries around the sort-key structure.

This is why **sort-key design from Lesson 2** is so important.

---

# 16. Query vs Scan

Memorize this:

```text
Query
  ↓
Uses key structure
  ↓
Targeted access

Scan
  ↓
Examines table/index broadly
  ↓
Potentially expensive
```

### Production preference

```text
GetItem / Query
        ↑
    Prefer these
        ↑
     Scan only
when appropriate
```

---

# 17. Why Scan Can Be Expensive

Suppose your table contains:

```text
100 million items
```

A Scan is conceptually asking DynamoDB to examine the table's items rather than directly targeting a known partition-key access path.

Even if you eventually return only:

```text
10 items
```

the operation may have examined much more data.

Therefore:

> **A FilterExpression does not turn a Scan into an efficient key lookup.**

---

# 18. Query + FilterExpression

Suppose you query:

```text
PK = CUSTOMER#001
```

and then specify:

```text
Status = PENDING
```

as a filter.

DynamoDB first evaluates the items matching the key condition, then applies the filter to the returned result set.

Therefore:

```text
Key condition
    ↓
Items evaluated
    ↓
Filter applied
    ↓
Final returned items
```

The filter isn't a replacement for good key design.

---

# 19. ProjectionExpression

Sometimes an item contains many attributes:

```text
Name
Email
Address
Phone
Preferences
History
Metadata
```

but your application needs only:

```text
Name
Email
```

A projection expression can request specific attributes.

This can reduce the amount of data returned to the application.

But remember:

> **Projection doesn't automatically mean the underlying read becomes free.**

Capacity consumption depends on the data DynamoDB needs to evaluate for the operation, not simply the number of attributes your application chooses to display.

---

# 20. Pagination

DynamoDB operations can return results in pages rather than returning an unlimited number of items at once.

The response can indicate that more results remain.

Conceptually:

```text
Query
  ↓
Page 1
  ↓
LastEvaluatedKey
  ↓
Query again
  ↓
Page 2
```

Applications must correctly handle pagination when retrieving large result sets.

---

# 21. BatchGetItem

Suppose you need:

```text
Customer C001
Customer C002
Customer C003
Customer C004
```

You could perform individual GetItem calls.

Or, where appropriate, use:

```text
BatchGetItem
```

to retrieve multiple items in a batch request.

### Benefits

1. Fewer application-level request operations.
2. Useful when retrieving multiple known keys.
3. Can improve application efficiency.
4. Supports requests across tables.
5. Applications must still handle partial results/retries appropriately.
6. Batch operations aren't a substitute for good data modeling.

---

# 22. BatchWriteItem

`BatchWriteItem` supports batches of:

```text
PutItem
DeleteItem
```

It is useful for bulk-style writes.

Example:

```text
BatchWrite
 ├── Put item A
 ├── Put item B
 ├── Put item C
 └── Delete item D
```

### Important

BatchWriteItem does **not** provide arbitrary transactional semantics.

For all-or-nothing multi-item operations, DynamoDB transactions are the appropriate feature.

We'll cover that in Lesson 7.

---

# 23. Read Consistency

DynamoDB provides two important read consistency models:

```text
Eventually Consistent Read
Strongly Consistent Read
```

Think:

```text
Eventually consistent
       ↓
Potentially slightly older view

Strongly consistent
       ↓
Most up-to-date committed value
```

---

# 24. Eventually Consistent Reads

An eventually consistent read may not immediately reflect a very recent successful write.

Example:

```text
T0
Write:
Balance = 100

T1
Read:
Potentially sees previous value
```

This model is useful when the application doesn't require the newest value on every read.

Examples might include:

```text
Product browsing
Non-critical dashboards
Some cached-like reads
General content retrieval
```

The correct choice depends on the application.

---

# 25. Strongly Consistent Reads

A strongly consistent read requests the latest value available from the relevant DynamoDB replica set after a successful write.

Example:

```text
Write
  ↓
Strongly consistent read
  ↓
Latest committed value
```

This is useful for workloads where stale reads would cause incorrect application behavior.

Examples might include:

```text
Critical state
Certain inventory checks
Workflow state
Application decisions requiring current data
```

---

# 26. Strong vs Eventually Consistent

| Feature                                   | Eventually Consistent               | Strongly Consistent                |
| ----------------------------------------- | ----------------------------------- | ---------------------------------- |
| May return older data                     | Yes                                 | No, within its supported semantics |
| Read capacity                             | Lower                               | Higher                             |
| Use case                                  | General reads                       | Freshness-sensitive reads          |
| Availability across all DynamoDB features | Broader                             | More limited                       |
| Application decision                      | When slight staleness is acceptable | When freshness is required         |

### Important

Strong consistency is **not automatically "better."**

It is a business/application requirement.

---

# 27. Real-Time Scenario — Order Status

Suppose:

```text
Order = SHIPPED
```

The customer immediately opens the order page.

If the application reads from a context where eventual consistency is acceptable, it might briefly see:

```text
PROCESSING
```

If the business requirement says:

> "After confirming the update, the next critical read must reflect it."

then you need to evaluate whether a strongly consistent read or another architecture is appropriate.

---

# 28. Read-After-Write Scenario

Suppose:

```text
1. User updates profile.
2. Application immediately reads profile.
```

Question:

> Could the second operation see an older value?

Potentially, if the read uses eventual consistency.

Therefore the application should explicitly determine whether:

```text
Stale data is acceptable
```

or:

```text
Latest data is required
```

This is a critical distributed-systems concept.

---

# 29. Conditional Writes

Conditional expressions are extremely useful.

Suppose:

```text
Inventory = 5
```

You want to sell one item only if inventory is greater than zero.

Conceptually:

```text
Update Inventory
Condition:
Inventory > 0
```

Then:

```text
Inventory = 5
       ↓
Condition succeeds
       ↓
Inventory = 4
```

If:

```text
Inventory = 0
```

the condition fails.

This helps prevent overselling caused by race conditions.

---

# 30. Optimistic Concurrency Control

Imagine two users modify the same record.

Current version:

```text
Version = 10
```

User A reads:

```text
Version = 10
```

User B also reads:

```text
Version = 10
```

User A updates first:

```text
Version = 11
```

User B attempts:

```text
Update where Version = 10
```

The condition fails because the version is now:

```text
11
```

This is a common optimistic concurrency pattern.

---

# 31. Idempotency

In distributed systems, the same request may be retried.

Suppose:

```text
Create payment
```

The application sends the request.

Network problem occurs.

The application doesn't know whether DynamoDB processed it.

It retries.

Without an idempotency strategy, you could accidentally create duplicate business records.

A common solution is to use a unique idempotency key and conditional write.

Conceptually:

```text
RequestId = ABC123

If ABC123 doesn't exist:
    process request

If ABC123 already exists:
    don't process again
```

---

# 32. Transactions

DynamoDB supports transactional operations.

For example:

```text
Create Order
+
Decrease Inventory
+
Create Payment Record
```

If these operations must succeed together, transactions can be considered.

Conceptually:

```text
Transaction
 ├── Order
 ├── Inventory
 └── Payment
       │
       ▼
  All succeed
       OR
  Transaction fails
```

Transactions are covered in detail in Lesson 7.

---

# 33. Read Capacity and Write Capacity

DynamoDB capacity is measured differently from traditional database CPU sizing.

Important concepts include:

```text
RCU = Read Capacity Unit
WCU = Write Capacity Unit
```

These represent consumed read/write capacity under DynamoDB's capacity model.

Your consumption depends on factors such as:

```text
Item size
Operation type
Consistency
Request volume
```

---

# 34. Why Item Size Matters

Suppose:

```text
Item A = 1 KB
Item B = 20 KB
```

Reading/writing the larger item consumes more capacity than the smaller item under the relevant DynamoDB capacity rules.

Therefore:

> **Data modeling affects both performance and cost.**

This is why unnecessary large attributes and inefficient item structures can become expensive at scale.

---

# 35. Write Amplification Through GSIs

Suppose your table has:

```text
Base Table
+
GSI 1
+
GSI 2
```

Writes affecting indexed attributes can also require updates to the relevant indexes.

Therefore:

```text
Application Write
       │
       ▼
Base Table
       │
   ┌───┴───┐
   ▼       ▼
 GSI 1    GSI 2
```

Indexes are powerful, but they aren't free.

We'll study this properly in Lesson 5.

---

# 36. Real-Time Scenario — High-Traffic API

Suppose:

```text
10,000 requests/sec
```

and each request performs:

```text
GetItem
```

using a known partition key.

This is a workload DynamoDB can be well suited for.

But you still need to consider:

```text
Key distribution
Item size
Capacity mode
Hot keys
Latency
Retries
Throttling
```

High request volume alone doesn't guarantee a good DynamoDB design.

---

# 37. Real-Time Scenario — Bad Scan Architecture

Suppose:

```text
GET /customers?email=user@example.com
```

and the implementation does:

```text
Scan entire customer table
   ↓
Find email
   ↓
Return customer
```

This is a red flag.

Instead, if email lookup is an important access pattern, consider a key/index design that supports:

```text
Email
 ↓
Targeted lookup
```

This is exactly why Lesson 2's access-pattern-first modeling matters.

---

# 38. Real-Time Scenario — Inventory

Suppose:

```text
Stock = 1
```

Two customers purchase simultaneously.

Bad pattern:

```text
Customer A → Read stock = 1
Customer B → Read stock = 1

A → Write 0
B → Write 0
```

Both applications may believe they successfully purchased the final item.

A better pattern can use a conditional update:

```text
Update stock
Condition:
stock >= 1
```

Only one operation should successfully consume the final unit.

This is a classic DynamoDB conditional-write use case.

---

# 39. Real-Time Scenario — Duplicate Order

Application receives:

```text
POST /order
RequestId = REQ123
```

Network timeout occurs.

Application retries.

Without idempotency:

```text
REQ123 → Order A
REQ123 → Order B
```

With an idempotency record:

```text
REQ123
   ↓
Already processed?
   │
  Yes
   ↓
Return existing result
```

This protects against duplicate processing.

---

# 40. Error Handling

Production applications must handle DynamoDB errors appropriately.

Examples include:

```text
ConditionalCheckFailedException
ProvisionedThroughputExceededException
ThrottlingException
ResourceNotFoundException
ValidationException
```

Don't treat every error as:

> "Retry immediately."

Different errors require different responses.

---

# 41. Retry Strategy

For transient throttling or service-related failures:

```text
Request
  ↓
Failure
  ↓
Backoff
  ↓
Retry
  ↓
Success
```

Use an appropriate **exponential backoff with jitter** strategy rather than immediately hammering DynamoDB with retries.

Otherwise:

```text
Failure
 ↓
Retry
 ↓
Retry
 ↓
Retry
 ↓
More load
 ↓
More throttling
```

can create a feedback loop.

---

# 42. Idempotent Retries

Retries become much safer when operations are designed to be idempotent.

For example:

```text
Set Status = SHIPPED
```

is generally easier to retry safely than:

```text
Increment something blindly
```

For business operations, use:

```text
Conditional writes
Idempotency keys
Transactions where appropriate
```

---

# 43. Batch Operations — Important Caveat

Batch APIs can return **unprocessed items**.

Therefore your application should be prepared to:

```text
Receive response
     ↓
Check unprocessed items
     ↓
Retry appropriately
```

Don't assume:

> "Batch request returned successfully, therefore every item necessarily completed."

Your implementation must follow the API response semantics.

---

# 44. Lesson 3 — Production Decision Tree

When you need to access DynamoDB:

```text
Do I know the exact primary key?
        │
       YES
        ↓
     GetItem

        NO
        │
        ▼
Do I know the partition key?
        │
       YES
        ↓
      Query

        NO
        │
        ▼
Is there an appropriate GSI?
        │
       YES
        ↓
    Query the GSI

        NO
        │
        ▼
Reconsider the data model
        │
        ▼
Avoid making Scan the
normal application path
```

This is one of the most useful DynamoDB mental models.

---

# 45. Top 15 Interview Questions — Lesson 3

### 1. What does PutItem do?

Creates or replaces an item identified by its primary key.

### 2. What does GetItem do?

Retrieves a specific item using its complete primary key.

### 3. What does UpdateItem do?

Modifies attributes of an existing item and can perform atomic updates.

### 4. What does DeleteItem do?

Deletes an item identified by its primary key.

### 5. Query vs Scan?

**Query** uses a key-based access pattern; **Scan** examines a broader table/index and can be expensive.

### 6. What is eventual consistency?

A read may temporarily return a value that doesn't reflect the most recent successful write.

### 7. What is strong consistency?

A supported read operation can request the latest committed value rather than accepting eventual consistency.

### 8. Why isn't strong consistency always used?

Because it has different capacity/performance characteristics and isn't necessary for every workload.

### 9. What is a conditional write?

A write that succeeds only if a specified condition is satisfied.

### 10. How do conditional writes help inventory systems?

They can prevent stock from being decremented when the available inventory condition is no longer true.

### 11. What is optimistic locking?

Using a version/condition so an update succeeds only if the record hasn't changed since it was read.

### 12. Why are retries dangerous?

Retries can duplicate business operations unless the operation is designed to be idempotent.

### 13. What is BatchGetItem?

An API for retrieving multiple known items in a batch request.

### 14. What is BatchWriteItem?

An API for batching PutItem/DeleteItem operations.

### 15. What should you do when an important application operation requires Scan?

**Revisit the data model.** Determine whether a primary key or secondary index can support that access pattern efficiently.

---

# 🧠 Lesson 3 — Final Mental Model

```text
                         DYNAMODB OPERATIONS
                                │
             ┌──────────────────┴──────────────────┐
             │                                     │
           READ                                  WRITE
             │                                     │
      ┌──────┼──────┐                    ┌─────────┼─────────┐
      ▼      ▼      ▼                    ▼         ▼         ▼
 GetItem  Query   Scan                PutItem  UpdateItem DeleteItem
      │      │      │                    │         │         │
      └──────┴──────┘                    └─────────┴─────────┘
             │                                     │
             ▼                                     ▼
       KEY-BASED ACCESS                      CONDITIONAL WRITE
             │                                     │
             ▼                                     ▼
      Efficient Reads                       Safe State Changes
             │                                     │
             └──────────────────┬──────────────────┘
                                ▼
                         CONSISTENCY
                        /           \
                       /             \
              EVENTUAL             STRONG
                  │                   │
                  ▼                   ▼
             Lower-cost          Freshness-sensitive
             general reads       application reads
```

## 🔥 Lesson 3 takeaway

The production mindset is:

> **Know the key → use GetItem/Query → avoid unnecessary Scan → choose consistency deliberately → use conditional writes for concurrency → design retries to be safe and idempotent.**

**Next: Lesson 4 — DynamoDB Capacity, Performance & Scaling: RCU/WCU, On-Demand vs Provisioned, throttling, hot partitions, adaptive capacity, auto scaling, and real-time performance troubleshooting.**
