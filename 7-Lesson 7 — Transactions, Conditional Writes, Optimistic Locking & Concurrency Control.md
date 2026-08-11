# ⚡ Amazon DynamoDB — Lesson 7

## Transactions, Conditional Writes, Optimistic Locking & Concurrency Control

This lesson covers one of the most important **production DynamoDB topics**:

> **How do you keep data correct when multiple users, services, or Lambda functions modify the same data at the same time?**

The core concepts are:

```text
Conditional Writes
        ↓
Optimistic Locking
        ↓
Transactions
        ↓
Concurrency Control
        ↓
Atomic Business Operations
```

---

# 1. Why Concurrency Matters

Imagine two users try to buy the last product.

Current stock:

```text
Stock = 1
```

User A reads:

```text
Stock = 1
```

User B reads:

```text
Stock = 1
```

Both attempt:

```text
Stock = Stock - 1
```

Without proper concurrency control, both could succeed incorrectly.

You might end up with:

```text
Stock = -1
```

or sell two units when only one existed.

---

# 2. The Lost Update Problem

Consider:

```text
Initial:
Balance = $100
```

Two processes read the same value.

```text
Process A → reads $100
Process B → reads $100
```

A writes:

```text
$100 + $50 = $150
```

B writes:

```text
$100 + $20 = $120
```

Final value:

```text
$120
```

But the expected result might be:

```text
$170
```

Process B overwrote A's update.

This is a:

> **Lost update**

---

# 3. DynamoDB Conditional Writes

A powerful solution is:

> **Conditional expressions**

Instead of:

```text
Update item
```

you say:

```text
Update item ONLY IF condition is true
```

Conceptually:

```text
UPDATE stock
SET stock = stock - 1
IF stock >= 1
```

This is extremely useful for preventing invalid state transitions.

---

# 4. Conditional Write Example

Suppose:

```text
ProductId = P100
Stock = 5
```

You want to sell one item.

Condition:

```text
Stock >= 1
```

Update:

```text
Stock = Stock - 1
```

DynamoDB evaluates the condition as part of the write.

If the condition is true:

```text
SUCCESS
```

If false:

```text
CONDITIONAL CHECK FAILED
```

---

# 5. Why Conditional Writes Are Powerful

They prevent the classic:

```text
Read
  ↓
Check
  ↓
Write
```

race condition.

Instead, the condition and write are evaluated by DynamoDB as part of the write operation.

This is much safer for concurrent workloads.

---

# 6. Common Conditional Write Use Cases

Use conditional writes for:

```text
Inventory reservation
Unique username creation
State transitions
Preventing duplicate records
Optimistic locking
Account balance validation
Order processing
Preventing accidental overwrites
```

---

# 7. Example — Create Only If Item Doesn't Exist

Suppose:

```text
Username = chinnu
```

must be unique.

Two users attempt:

```text
Create username = chinnu
```

Use a condition equivalent to:

```text
attribute_not_exists(primaryKey)
```

Conceptually:

```text
Create user
IF username does not already exist
```

First request:

```text
SUCCESS
```

Second request:

```text
CONDITION FAILED
```

This provides an application-level uniqueness guard.

---

# 8. Conditional Expressions

Common functions/operators include:

```text
attribute_exists()
attribute_not_exists()
begins_with()
contains()
BETWEEN
IN
=
<>
<
>
<=
>=
```

These can be combined with logical operators such as:

```text
AND
OR
NOT
```

---

# 9. Example — State Transition

Suppose an order is:

```text
Status = PENDING
```

You want:

```text
PENDING → PAID
```

But you don't want:

```text
CANCELLED → PAID
```

Use:

```text
Condition:
Status = PENDING

Update:
Status = PAID
```

Now concurrent or invalid updates are rejected.

---

# 10. Why This Is Better Than Application-Only Validation

Bad pattern:

```text
Application reads:
Status = PENDING

Application checks:
"Okay, I can update."

Another process changes:
Status = CANCELLED

First application writes:
Status = PAID
```

The original validation is now stale.

Better:

```text
Update Status = PAID
IF Status = PENDING
```

The condition is evaluated as part of the write.

---

# 11. Optimistic Locking

Optimistic locking assumes:

> **Conflicts are possible, but relatively uncommon.**

Instead of locking the database item before editing it, each version of the item gets a version number.

Example:

```text
OrderId = O100
Status = PENDING
Version = 5
```

---

# 12. Optimistic Locking Workflow

Process A reads:

```text
Version = 5
```

Process B also reads:

```text
Version = 5
```

A updates:

```text
Version = 6
```

B attempts:

```text
UPDATE
IF Version = 5
```

But current version is:

```text
6
```

Therefore:

```text
CONDITION FAILED
```

B knows:

> "Someone changed this record after I read it."

---

# 13. Version Attribute Pattern

Typical item:

```text
{
    "OrderId": "O100",
    "Status": "PENDING",
    "Version": 5
}
```

Update:

```text
SET Status = "PAID",
    Version = Version + 1

IF Version = 5
```

If successful:

```text
Version = 6
```

---

# 14. Optimistic Locking vs Pessimistic Locking

### Optimistic

```text
Read
 ↓
Modify
 ↓
Write IF version unchanged
```

### Pessimistic

Conceptually:

```text
Acquire lock
 ↓
Modify
 ↓
Release lock
```

DynamoDB applications commonly use **optimistic concurrency techniques** rather than traditional long-held database locks.

---

# 15. Why Optimistic Locking Works Well in DynamoDB

DynamoDB is designed for highly distributed workloads.

Traditional locking can introduce:

```text
Blocking
Contention
Long-held locks
Reduced throughput
Complex failure handling
```

Optimistic concurrency instead lets requests proceed and rejects conflicting updates.

---

# 16. Handling Optimistic Lock Failures

Suppose:

```text
Expected Version = 5
Actual Version = 6
```

The application can:

```text
1. Reload item
2. Recalculate desired change
3. Retry conditionally
```

Example:

```text
Read Version 6
      ↓
Recalculate
      ↓
Update IF Version = 6
```

But don't blindly retry forever.

---

# 17. Conditional Check Failed Is Not Always an Infrastructure Failure

This is important.

A conditional failure often means:

> **Your business rule was false.**

Example:

```text
Stock >= 1
```

fails because:

```text
Stock = 0
```

That's not necessarily a server outage.

Your application should handle it as a business outcome.

---

# 18. Transactions

Conditional writes protect individual operations.

But sometimes you need multiple writes to succeed or fail together.

Example:

```text
Order
Inventory
Payment record
```

You may need:

```text
All succeed
OR
None succeed
```

This is where:

> **DynamoDB Transactions**

become important.

---

# 19. What Is a DynamoDB Transaction?

A transaction allows you to perform multiple DynamoDB actions as a single logical unit.

Conceptually:

```text
Transaction
 ├── Update Item A
 ├── Update Item B
 └── Put Item C

        ↓

   ALL SUCCESS
        OR
   ALL ROLLED BACK
```

This provides atomicity across multiple items.

---

# 20. Example — Order + Inventory

Suppose:

```text
Inventory:
Stock = 1
```

Create:

```text
Order O100
```

and decrease:

```text
Stock = 0
```

You don't want:

```text
Order created
BUT
stock not reduced
```

or:

```text
Stock reduced
BUT
order not created
```

A transaction can make the operations atomic.

---

# 21. Transaction Example

Conceptually:

```text
TransactWriteItems

1. Put Order O100
2. Update Product P100
   IF Stock >= 1
```

Result:

```text
If condition succeeds:
    Order created
    Stock decreased

If condition fails:
    Neither change is committed
```

This is a very powerful pattern.

---

# 22. Transactions + Conditions

You can combine:

```text
Transaction
+
Conditional expressions
```

For example:

```text
Create Order
AND
Decrease Stock
IF Stock >= 1
```

This gives you both:

```text
Atomicity
+
Business-rule validation
```

---

# 23. Transaction APIs

Two important DynamoDB transaction operations are:

```text
TransactWriteItems
TransactGetItems
```

### TransactWriteItems

Used for atomic writes such as:

```text
Put
Update
Delete
ConditionCheck
```

### TransactGetItems

Used to retrieve multiple items transactionally.

---

# 24. Transaction Actions

`TransactWriteItems` can contain actions such as:

```text
Put
Update
Delete
ConditionCheck
```

This lets you build sophisticated atomic operations.

---

# 25. ConditionCheck

A `ConditionCheck` allows you to verify a condition as part of a transaction without necessarily modifying that item.

Conceptually:

```text
Transaction:

Check:
Account status = ACTIVE

Then:
Create transaction record
Update another item
```

If the condition fails:

```text
Entire transaction fails.
```

---

# 26. Transaction Example — Bank Transfer

Suppose:

```text
Account A = $100
Account B = $50
```

Transfer:

```text
$30
```

Desired result:

```text
Account A = $70
Account B = $80
```

You don't want only one side updated.

Transaction:

```text
1. Decrease A by $30
2. Increase B by $30
```

Both succeed:

```text
A = $70
B = $80
```

Otherwise:

```text
No partial transfer
```

---

# 27. Add a Balance Condition

You can make the transfer safer:

```text
Decrease A by $30
IF A.balance >= 30

Increase B by $30
```

Now:

```text
Balance = $20
```

means:

```text
Condition fails
```

and the transaction does not partially apply.

---

# 28. Transaction vs Conditional Write

This is a common interview question.

### Conditional write

Usually:

```text
One item
+
Condition
+
Atomic update
```

Example:

```text
Decrease stock
IF stock >= 1
```

### Transaction

```text
Multiple item operations
+
Atomicity
```

Example:

```text
Create order
+
Decrease inventory
```

---

# 29. When Should You NOT Use Transactions?

Transactions are powerful but shouldn't automatically be used for every write.

If you only need:

```text
Update one item
IF condition
```

a conditional write may be simpler and cheaper.

Use transactions when the business operation genuinely requires:

> **Atomic multi-item/multi-operation behavior.**

---

# 30. Transaction Costs and Performance

Transactions have additional overhead compared with ordinary single-item writes.

Therefore:

```text
Simple operation
     ↓
Normal write

Single-item business rule
     ↓
Conditional write

Multiple related operations requiring atomicity
     ↓
Transaction
```

Use the simplest mechanism that guarantees correctness.

---

# 31. Transaction Conflicts

Two transactions can compete for the same items.

Example:

```text
Transaction A
     ↓
Product P100

Transaction B
     ↓
Product P100
```

One may fail because of a conflict.

Your application needs appropriate retry handling.

---

# 32. Retry Strategy

For retryable failures:

```text
Attempt 1
   ↓
Failure
   ↓
Short backoff
   ↓
Attempt 2
   ↓
Failure
   ↓
Longer backoff
```

This is:

> **Exponential backoff**

Usually combine it with:

> **Jitter**

so many clients don't retry at exactly the same time.

---

# 33. Why Jitter?

Imagine 10,000 clients all fail at:

```text
12:00:00
```

If all retry at:

```text
12:00:01
```

they create another traffic spike.

With jitter:

```text
Client A → 12:00:00.72
Client B → 12:00:01.14
Client C → 12:00:00.91
...
```

The retries are spread out.

---

# 34. Hot Item Concurrency

Suppose thousands of users update:

```text
Counter = same item
```

This creates contention around a single logical item.

Even with correct conditional writes, the workload may become inefficient.

This is related to:

> **Hot keys / hot partitions**

covered earlier.

---

# 35. Atomic Counters

DynamoDB supports atomic numeric updates.

Conceptually:

```text
SET counter = counter + 1
```

This avoids:

```text
Read counter
+
Application increments
+
Write counter
```

which can create lost-update problems.

---

# 36. Atomic Counter Example

Current:

```text
Views = 100
```

Two requests:

```text
+1
+1
```

Using an atomic update:

```text
Views = Views + 1
```

results in:

```text
102
```

without needing the application to read the old value first.

---

# 37. Atomic Counter vs Transaction

An atomic counter is useful when:

```text
One numeric attribute
```

needs to be changed safely.

A transaction is useful when:

```text
Multiple related changes
```

must succeed together.

---

# 38. Real-Time Scenario — Inventory

Requirement:

> Sell a product only when stock exists.

Bad:

```text
Read stock
 ↓
if stock > 0
 ↓
Write stock - 1
```

Good:

```text
Update stock = stock - 1
IF stock >= 1
```

If you also need to create an order atomically:

```text
Transaction
 ├── Create order
 └── Decrease inventory
      IF stock >= 1
```

---

# 39. Real-Time Scenario — Unique Registration

Requirement:

> Email address can only be registered once.

Pattern:

```text
Create user
IF email does not already exist
```

If another request already created the record:

```text
ConditionalCheckFailed
```

Application returns:

```text
Email already registered
```

---

# 40. Real-Time Scenario — Order State Machine

Allowed:

```text
PENDING → PAID
PAID → SHIPPED
SHIPPED → DELIVERED
```

Not allowed:

```text
DELIVERED → PENDING
CANCELLED → SHIPPED
```

Implement each transition with conditions:

```text
Update status
IF current status = expected previous status
```

This prevents invalid concurrent transitions.

---

# 41. Real-Time Scenario — Prevent Double Payment

Suppose two requests attempt:

```text
Order O100 → PAID
```

Use:

```text
Update order
SET Status = PAID
IF Status = PENDING
```

Only one request can successfully perform the transition.

The second request encounters a condition failure.

For payment systems, you'd also need a robust external-payment idempotency design.

---

# 42. Real-Time Scenario — Wallet Transfer

Requirement:

```text
Wallet A → Wallet B
```

Transaction:

```text
1. Decrease A
   IF balance >= amount

2. Increase B

3. Create transfer record
```

All operations succeed or fail together.

---

# 43. Real-Time Scenario — Seat Reservation

Current:

```text
Seat A1 = AVAILABLE
```

Two users attempt reservation.

Conditional update:

```text
SET status = RESERVED
IF status = AVAILABLE
```

One succeeds.

The other receives:

```text
Condition failed
```

This prevents double booking of the same logical seat.

---

# 44. Real-Time Scenario — Optimistic UI Update

Suppose an admin edits:

```text
Product price
```

Record:

```text
Price = $100
Version = 7
```

Admin A reads version 7.

Admin B reads version 7.

A updates:

```text
Price = $110
Version = 8
```

B attempts:

```text
Price = $120
IF Version = 7
```

Fails.

B can then:

```text
Reload
Show conflict
Allow user to retry
```

This prevents silent overwrites.

---

# 45. Real-Time Scenario — Prevent Duplicate Job Creation

Suppose:

```text
JobId = J100
```

Multiple workers might attempt to create the same job.

Use:

```text
Put Job
IF JobId does not exist
```

Only one creation succeeds.

This is a simple distributed coordination technique.

---

# 46. Conditional Writes + Streams

These concepts work together.

Example:

```text
Update:
PENDING → PAID

Condition:
Status = PENDING
```

Successful update:

```text
DynamoDB
   ↓
Stream
   ↓
Lambda
```

The stream consumer can now safely process the legitimate state transition.

---

# 47. Transactions + Streams

Transactions and Streams can also work together.

Suppose:

```text
Transaction:
Create Order
Update Inventory
```

After the transaction commits, the resulting item changes can appear in DynamoDB Streams.

Downstream consumers can then react asynchronously.

Important principle:

> **The transaction provides atomicity for the database changes; Streams provide asynchronous propagation of those changes.**

---

# 48. Concurrency Control Decision Tree

Use this mental model:

```text
Need simple write?
      │
      ▼
Normal Put/Update/Delete

Need business condition?
      │
      ▼
Conditional Write

Need safe concurrent editing?
      │
      ▼
Optimistic Locking

Need multiple changes to succeed together?
      │
      ▼
Transaction

Need numeric increment?
      │
      ▼
Atomic Update

Need downstream event?
      │
      ▼
DynamoDB Streams
```

---

# 49. Common Mistake #1

### Bad design

```text
GetItem
 ↓
Check condition in application
 ↓
UpdateItem
```

Why?

The condition can become stale between the read and write.

### Better

```text
UpdateItem
ConditionExpression
```

Put the critical condition into the write operation.

---

# 50. Common Mistake #2

Using a transaction for everything.

If you only need:

```text
Stock > 0
```

for one item, a conditional update may be enough.

Don't introduce unnecessary transaction complexity.

---

# 51. Common Mistake #3

Ignoring conditional failures.

A developer might treat:

```text
ConditionalCheckFailed
```

as a generic application error.

Instead ask:

> **What business condition failed?**

Examples:

```text
Stock = 0
Order already paid
Record already exists
Version changed
```

These may be expected business outcomes.

---

# 52. Common Mistake #4

Blind retries.

Bad:

```text
Failure
 ↓
Retry immediately
 ↓
Failure
 ↓
Retry immediately
 ↓
...
```

Better:

```text
Retry
 ↓
Exponential backoff
 +
Jitter
 +
Maximum attempts
```

---

# 53. Common Mistake #5

Ignoring idempotency

Suppose transaction/event processing is retried.

If the operation causes:

```text
Charge
Send email
Create external resource
```

you must understand whether repeating it is safe.

---

# 54. Common Mistake #6

Using DynamoDB transactions to solve an application modeling problem.

Sometimes the better solution is to redesign the item structure so related information is stored together.

Remember:

> **Good DynamoDB data modeling can eliminate the need for cross-item transactions.**

---

# 55. Denormalization vs Transactions

Suppose you frequently need:

```text
Order
+
Customer summary
+
Order status
```

If these can safely live together in one item, a single atomic item update may be simpler.

Instead of:

```text
Transaction:
Update A
Update B
Update C
```

you may sometimes model:

```text
One item
```

and update it atomically.

But don't denormalize blindly; choose the model based on access patterns and consistency requirements.

---

# 56. Transactional vs Eventually Consistent Architecture

### Stronger atomic workflow

```text
Transaction
 ↓
All related changes commit together
```

### Event-driven workflow

```text
DynamoDB
 ↓
Stream
 ↓
Lambda
 ↓
Eventually update downstream
```

These solve different problems.

A transaction gives atomicity **within the DynamoDB transaction**.

Streams provide **asynchronous propagation**.

---

# 57. Conditional Write vs Transaction Interview Question

### Question:

> "I need to decrease stock only if stock > 0. Do I need a transaction?"

Answer:

> Not necessarily.

If only one item needs to change:

```text
UpdateItem
+
ConditionExpression
```

is usually sufficient.

If you also need:

```text
Create order
+
Decrease stock
```

atomically:

```text
TransactWriteItems
```

is appropriate.

---

# 58. Optimistic Locking Interview Question

### Question:

> "Two admins edit the same product. How do you prevent one from overwriting the other?"

Answer:

Use a version attribute.

```text
Version = 5
```

Update:

```text
IF Version = 5
```

and increment:

```text
Version = 6
```

If another process already changed it:

```text
Condition fails
```

Then reload and resolve the conflict.

---

# 59. Transaction Interview Question

### Question:

> "How would you transfer money between two DynamoDB items?"

Answer:

Use a transaction:

```text
Debit Account A
IF balance >= amount

Credit Account B

Create transfer record
```

All changes should succeed or fail atomically.

---

# 60. Production Architecture

A robust DynamoDB application can combine everything:

```text
                         APPLICATION
                              │
                              ▼
                         DynamoDB
                              │
          ┌───────────────────┼──────────────────┐
          │                   │                  │
          ▼                   ▼                  ▼
 Conditional Writes      Transactions       Atomic Updates
          │                   │                  │
          └───────────────────┼──────────────────┘
                              │
                              ▼
                       Correct State
                              │
                              ▼
                         DynamoDB
                         Streams
                              │
                              ▼
                           Lambda
                              │
                    ┌─────────┼─────────┐
                    ▼         ▼         ▼
                   SQS      EventBridge  SNS
                    │
                    ▼
              Downstream Systems
```

---

# 🎯 Lesson 7 — Top 30 Interview Questions

### 1. What is a conditional write?

A write that succeeds only when a specified condition is true.

### 2. Why use conditional writes?

To enforce business rules and protect against concurrent updates.

### 3. What is optimistic locking?

Using a version value to detect whether an item changed since it was read.

### 4. How does version-based optimistic locking work?

Read version → update only if version is unchanged → increment version.

### 5. What happens if the version changed?

The conditional update fails, allowing the application to detect a concurrency conflict.

### 6. What is a DynamoDB transaction?

An atomic group of multiple DynamoDB operations.

### 7. What is `TransactWriteItems`?

An API for performing multiple write operations atomically.

### 8. What is `TransactGetItems`?

An API for transactionally retrieving multiple items.

### 9. Can transactions contain conditions?

Yes.

### 10. What is `ConditionCheck`?

A transaction action that validates a condition without necessarily modifying the item.

### 11. Can transactions span multiple DynamoDB items?

Yes, subject to DynamoDB transaction limits.

### 12. Can transactions span multiple DynamoDB tables?

Yes, within the supported transaction constraints.

### 13. When should you use a conditional write instead of a transaction?

When a single-item operation with a business condition is sufficient.

### 14. When should you use a transaction?

When multiple related operations must succeed or fail atomically.

### 15. What is a lost update?

When one concurrent update unintentionally overwrites another update.

### 16. How can DynamoDB prevent lost updates?

Conditional writes, optimistic locking, or appropriate transactional designs.

### 17. What is an atomic counter?

An atomic numeric update such as incrementing a value without first reading it.

### 18. What is a hot item?

An item receiving a disproportionately high number of concurrent operations.

### 19. What happens when a condition fails?

The write is rejected and the application receives a conditional failure response.

### 20. Is a conditional failure necessarily a system error?

No. It can represent an expected business condition.

### 21. How do you prevent double booking?

Use a conditional update such as `status = AVAILABLE`.

### 22. How do you prevent duplicate creation?

Condition the creation on the item not already existing.

### 23. How do you implement order state transitions safely?

Use conditional updates based on the expected current state.

### 24. How do you transfer money atomically?

Use a transaction with conditional balance validation.

### 25. Should every operation use a transaction?

No. Use the simplest mechanism that provides the required correctness.

### 26. Why use exponential backoff?

To reduce repeated pressure during transient conflicts or throttling.

### 27. Why add jitter?

To prevent many clients from retrying simultaneously.

### 28. Can transactions and Streams be used together?

Yes. Transactions provide atomic database changes; Streams can propagate resulting changes asynchronously.

### 29. Can good data modeling reduce the need for transactions?

Yes. Co-locating related data can sometimes eliminate cross-item atomic operations.

### 30. What is the key lesson?

> **Use conditions for business rules, optimistic locking for concurrent edits, transactions for atomic multi-item operations, and idempotent retries for distributed reliability.**

---

# 🔥 Lesson 7 Golden Rules

1. **Never rely solely on application-side checks for concurrency-sensitive writes.**
2. **Put critical conditions into DynamoDB write operations.**
3. **Use conditional writes for single-item business rules.**
4. **Use optimistic locking when multiple actors may edit the same item.**
5. **Use version numbers to detect stale updates.**
6. **Use transactions when multiple operations must succeed or fail together.**
7. **Use atomic counters for safe numeric increments/decrements.**
8. **Treat conditional failures as meaningful business outcomes when appropriate.**
9. **Use exponential backoff + jitter for suitable retries.**
10. **Design important operations to be idempotent.**
11. **Don't use transactions when a simpler conditional write is sufficient.**
12. **Good DynamoDB data modeling can reduce transaction requirements.**
13. **Conditional writes + Streams are excellent for safe state-transition architectures.**
14. **Transactions provide database atomicity; Streams provide asynchronous propagation.**
15. **Concurrency correctness should be designed explicitly—not added after production problems occur.**

## 🧠 One-line memory trick

> **Condition = "Is this allowed?" → Version = "Did someone change it?" → Transaction = "Do all these changes happen together?" → Atomic update = "Change this value safely."**

**Next: Lesson 8 — Security, IAM, Encryption, VPC Endpoints, Fine-Grained Access & DynamoDB Protection.**
