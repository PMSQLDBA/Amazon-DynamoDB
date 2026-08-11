# ⚡ Amazon DynamoDB — Lesson 6

## Streams, TTL & Event-Driven Architecture

This lesson moves DynamoDB from a **database-only component** into an **event-driven system**.

The core production architecture is:

```text
                    APPLICATION
                         │
                         ▼
                    DynamoDB
                         │
              ┌──────────┴──────────┐
              │                     │
              ▼                     ▼
        DynamoDB Streams          TTL
              │                     │
              ▼                     ▼
           Lambda              Expired Items
              │
      ┌───────┼────────┐
      ▼       ▼        ▼
   SQS/SNS  EventBridge  Other Services
      │
      ▼
Downstream Processing
```

---

# 1. Why DynamoDB Streams Matter

Normally, your application writes data:

```text
Application
     ↓
DynamoDB
```

But other systems may need to know:

> "Something changed in DynamoDB."

For example:

* A customer was created.
* An order status changed.
* A payment record was updated.
* A product was deleted.
* A session expired.

Instead of repeatedly querying DynamoDB to discover changes, you can use:

> **DynamoDB Streams**

---

# 2. What Is DynamoDB Streams?

DynamoDB Streams captures a **time-ordered sequence of item-level modifications** in a DynamoDB table.

Conceptually:

```text
PutItem
   ↓
DynamoDB
   ↓
Stream record

UpdateItem
   ↓
DynamoDB
   ↓
Stream record

DeleteItem
   ↓
DynamoDB
   ↓
Stream record
```

This enables downstream systems to react to database changes.

---

# 3. Streams Are Not the Same as a Message Queue

A common misconception is:

> "DynamoDB Streams is just SQS."

It isn't.

DynamoDB Streams is primarily a **change-data-capture mechanism for DynamoDB item modifications**.

It tells downstream consumers:

```text
What changed?
Which item changed?
What type of change occurred?
```

SQS, on the other hand, is primarily a messaging/queueing service.

---

# 4. What Events Does a Stream Capture?

DynamoDB Streams can capture:

```text
INSERT
MODIFY
REMOVE
```

Meaning:

```text id="f0ibbr"
INSERT
  ↓
New item created

MODIFY
  ↓
Existing item changed

REMOVE
  ↓
Item deleted
```

These events can be consumed by applications.

---

# 5. Stream View Types

When enabling DynamoDB Streams, you choose what information the stream records contain.

The four stream view types are:

```text id="l8x4oz"
KEYS_ONLY
NEW_IMAGE
OLD_IMAGE
NEW_AND_OLD_IMAGES
```

This is an important interview topic.

---

# 6. KEYS_ONLY

With:

```text id="2p9bwy"
KEYS_ONLY
```

the stream record contains the item's key attributes.

Useful when downstream processing only needs to know:

> "Which item changed?"

Example:

```text id="9fjr5e"
CustomerId = C001
OrderId = O100
```

The downstream consumer can then potentially retrieve additional data if needed.

---

# 7. NEW_IMAGE

With:

```text id="g5n1e2"
NEW_IMAGE
```

the stream record contains the item as it exists **after the modification**.

Example:

```text id="6z4x8m"
Before:
Status = PENDING

After:
Status = SHIPPED
```

The stream provides the new image:

```text id="4d3v5z"
Status = SHIPPED
```

Useful for downstream systems that care about the new state.

---

# 8. OLD_IMAGE

With:

```text id="8w0u4c"
OLD_IMAGE
```

the stream contains the item's state **before the modification**.

Example:

```text id="n4g1qb"
Before:
Status = PENDING

After:
Status = SHIPPED
```

The stream provides:

```text id="7v4y5a"
Status = PENDING
```

Useful when you need to know what changed from the previous state.

---

# 9. NEW_AND_OLD_IMAGES

With:

```text id="4t7yqj"
NEW_AND_OLD_IMAGES
```

you get both:

```text id="w2c5y9"
OLD
 +
NEW
```

This is extremely useful for determining the actual state transition.

Example:

```text id="r5j0q3"
OLD:
Status = PENDING

NEW:
Status = SHIPPED
```

You can then derive:

```text id="z4h1sa"
PENDING → SHIPPED
```

---

# 10. Which Stream View Should You Choose?

Use this mental model:

```text id="u3l1xk"
Need only item identity?
       ↓
KEYS_ONLY

Need new state?
       ↓
NEW_IMAGE

Need previous state?
       ↓
OLD_IMAGE

Need state transition?
       ↓
NEW_AND_OLD_IMAGES
```

Don't choose `NEW_AND_OLD_IMAGES` automatically.

More information can mean more data processing and downstream cost.

---

# 11. Stream Record Example

Imagine:

```text id="f42s9n"
OrderId = O100
Status = PENDING
```

Application updates:

```text id="6kqz5m"
Status = SHIPPED
```

With `NEW_AND_OLD_IMAGES`, conceptually:

```text id="w9n4yk"
{
    Key:
        OrderId = O100

    OldImage:
        Status = PENDING

    NewImage:
        Status = SHIPPED
}
```

A Lambda function can interpret:

```text id="d7p5v0"
PENDING → SHIPPED
```

and trigger downstream processing.

---

# 12. Stream Retention

DynamoDB Streams records are retained for a limited period.

The standard retention period is:

> **24 hours**

This is important.

DynamoDB Streams should not be treated as a permanent event archive.

If you need long-term event retention, design an explicit archival/event-storage mechanism.

---

# 13. Streams + Lambda

One of the most common architectures is:

```text id="s7j5q8"
DynamoDB
    │
    ▼
DynamoDB Streams
    │
    ▼
AWS Lambda
    │
    ▼
Business Logic
```

Example:

```text id="6aj4m8"
Order updated
      ↓
Stream
      ↓
Lambda
      ↓
Send notification
```

This is a classic serverless event-driven pattern.

---

# 14. Real-Time Example — Order Processing

Suppose an order changes:

```text id="h8k5z2"
Status:
PENDING
   ↓
PAID
```

DynamoDB Stream captures:

```text id="r1g4o0"
MODIFY
```

Lambda receives the event.

Lambda could:

```text id="m3w7px"
1. Validate state transition
2. Publish event
3. Update another service
4. Trigger fulfillment
5. Send notification
```

Now the database change becomes an event source.

---

# 15. Event-Driven Architecture

Instead of:

```text id="d6t2v7"
Application
   ↓
DynamoDB
   ↓
Application polls DynamoDB
   ↓
Detect change
```

use:

```text id="c1m8y2"
Application
   ↓
DynamoDB
   ↓
Streams
   ↓
Lambda
   ↓
Event processing
```

This eliminates unnecessary polling.

---

# 16. Polling vs Streams

### Polling

```text id="7r8q5w"
Every 5 seconds:
"Did anything change?"
```

Problems:

* Wasted reads
* Higher latency
* More application complexity
* Difficult scaling
* Potentially expensive

### Streams

```text id="8m5d1p"
Change occurs
     ↓
Stream record
     ↓
Consumer
```

This is more naturally event-driven.

---

# 17. Streams for Audit Pipelines

Suppose your application needs an audit trail.

DynamoDB:

```text id="p0r6u2"
Customer record changed
```

Stream:

```text id="w2s8x4"
MODIFY
```

Lambda:

```text id="c5k7n9"
Transform event
```

Then store it in:

```text id="g9q2m1"
S3
OpenSearch
Another DynamoDB table
Data warehouse
```

This creates an asynchronous audit pipeline.

---

# 18. Streams for Search Synchronization

Suppose DynamoDB is your source of truth.

You also maintain a search system.

Architecture:

```text id="k2h8v5"
DynamoDB
    ↓
Streams
    ↓
Lambda
    ↓
Search Index
```

When:

```text id="u4j9p3"
Product updated
```

the stream triggers processing.

Lambda updates the search index.

This is a common **event-driven synchronization** pattern.

---

# 19. Streams for Cache Invalidation

Suppose:

```text id="x7b4m1k"
DynamoDB
+
Application Cache
```

A customer record changes.

Stream:

```text id="a6c9v2"
MODIFY
```

Lambda:

```text id="n8d5q0"
Invalidate cache entry
```

Now downstream caches can be kept more synchronized with the database.

---

# 20. Important: Streams Are Asynchronous

Do not assume:

```text id="r7t3q1"
PutItem
 ↓
Lambda executes immediately
```

The processing is asynchronous.

Therefore:

```text id="m5c2v9"
DynamoDB write
      ↓
Stream
      ↓
Consumer
      ↓
Eventually processed
```

This means downstream systems should tolerate small delays.

---

# 21. Streams and Ordering

DynamoDB Streams preserve ordering for changes to items within the relevant stream/shard processing model, but you should not design your entire distributed architecture assuming global ordering across the entire table.

For example:

```text id="j4m9p2"
Order A:
PENDING → PAID

Order B:
PENDING → CANCELLED
```

Don't assume events for unrelated items have one global ordering relationship.

The key principle is:

> **Design consumers to be safe with distributed, asynchronous event processing.**

---

# 22. Idempotency

This is one of the most important event-driven concepts.

Suppose Lambda processes:

```text id="b8n2c7"
Order O100
```

and updates another system.

If the same event is processed again, you don't want:

```text id="h4m6v9"
Charge customer
Charge customer again
```

Therefore:

> **Stream consumers should be idempotent whenever possible.**

---

# 23. What Is Idempotency?

An operation is idempotent if performing it multiple times produces the same intended final effect.

Example:

```text id="n3p7x1"
Set Status = SHIPPED
```

Repeated execution:

```text id="r5q2m8"
SHIPPED
SHIPPED
SHIPPED
```

doesn't change the final state.

Compare that with:

```text id="s9k4d2"
Balance = Balance + $10
```

which can produce different results if processed repeatedly.

---

# 24. Stream Consumer Failure

Imagine:

```text id="q2v8m6"
DynamoDB
 ↓
Stream
 ↓
Lambda
 ↓
FAIL
```

You need a failure strategy.

Common components include:

```text id="t6w4p1"
Retries
Error handling
Dead-letter destinations
Monitoring
Idempotency
```

The exact mechanism depends on how the stream is integrated with Lambda and your event-processing architecture.

---

# 25. Lambda + Streams Batch Processing

Lambda processes DynamoDB Stream records in batches.

Conceptually:

```text id="c8j5v2"
Stream
 ├── Record 1
 ├── Record 2
 ├── Record 3
 └── Record 4
       ↓
    Lambda
```

Batch processing can improve efficiency.

But your code should correctly handle partial failures and retries.

---

# 26. Partial Batch Failure

Suppose:

```text id="p7k3m9"
Batch:
Record 1 → SUCCESS
Record 2 → FAIL
Record 3 → SUCCESS
```

A robust design should avoid unintentionally reprocessing successful records without considering idempotency.

Modern Lambda event-source mechanisms provide options for partial batch failure handling.

The architectural lesson is:

> **Design stream consumers for retries and partial failures.**

---

# 27. TTL — Time to Live

Now let's move to TTL.

TTL is used to automatically expire DynamoDB items after a specified time.

Example:

```text id="g4x8q2"
Session
ExpiresAt = timestamp
```

DynamoDB can later remove the expired item.

---

# 28. TTL Example

Suppose you store:

```text id="u8p3m6"
SessionId = S100
UserId = U100
ExpiresAt = ...
```

After expiration:

```text id="e6r1y4"
Item becomes eligible for TTL deletion.
```

This is useful for temporary data.

---

# 29. Common TTL Use Cases

TTL is useful for:

```text id="j3n8v5"
Sessions
Temporary tokens
Verification codes
Temporary cache records
Ephemeral state
Short-lived workflow data
Temporary locks
```

However, don't treat TTL as an exact timer.

---

# 30. TTL Is Not Exact-Time Deletion

This is a very important interview point.

Suppose:

```text id="s7c2m9"
ExpiresAt = 10:00:00
```

You should **not** assume:

```text id="w5r8q1"
10:00:00 exactly
   ↓
Item is immediately gone
```

TTL deletion is asynchronous.

Therefore:

> **TTL is for eventual expiration, not precise real-time scheduling.**

---

# 31. TTL Attribute

You configure a specific DynamoDB attribute as the TTL attribute.

For example:

```text id="p4m7z2"
expiresAt
```

The value represents an expiration timestamp.

The application should consistently write valid TTL values.

---

# 32. TTL and Data Types

TTL values need to follow DynamoDB's supported TTL format requirements.

The important production principle is:

> **Store the expiration timestamp in the configured TTL attribute using the expected format.**

Don't mix:

```text id="a8n4q6"
milliseconds
seconds
formatted strings
```

without understanding what your application and DynamoDB expect.

---

# 33. TTL + Streams

One of the most interesting patterns:

```text id="m3q7v9"
DynamoDB
   │
   ├── TTL
   │
   ▼
Expired item deleted
   │
   ▼
DynamoDB Stream
   │
   ▼
Lambda
```

TTL deletion can therefore participate in an event-driven workflow.

---

# 34. Example — Expired Session

Suppose:

```text id="c9x5m1"
Session
ExpiresAt = future timestamp
```

TTL eventually removes it.

Stream captures:

```text id="v4k8q2"
REMOVE
```

Lambda receives the event.

Lambda can:

```text id="z6p3n7"
1. Remove session from cache
2. Update analytics
3. Notify another service
4. Clean up related resources
```

This creates a powerful cleanup pipeline.

---

# 35. TTL Is Not a Business Workflow Scheduler

Bad architecture:

```text id="j2m6v8"
"At exactly 12:00 PM,
TTL will trigger my payment."
```

Wrong.

TTL is not designed for exact-time business actions.

Better:

```text id="q8w4p3"
EventBridge Scheduler
Step Functions
Application scheduler
```

depending on the requirement.

TTL should generally be treated as an **eventual data expiration mechanism**.

---

# 36. TTL and Storage Cleanup

Imagine:

```text id="u5d8n2"
100 million temporary sessions
```

Without expiration:

```text id="r3m7x9"
Data grows continuously
```

With TTL:

```text id="e2k6v4"
Expired sessions
      ↓
Eventually removed
```

This helps control long-term accumulation of ephemeral data.

---

# 37. TTL and Cost

TTL can help reduce storage accumulation from expired items.

However:

> **Don't assume TTL instantly eliminates all storage or downstream processing costs.**

Deletion is asynchronous and the operational impact depends on workload and architecture.

---

# 38. TTL and Streams — Important Detail

When TTL deletes an item, the deletion can appear in DynamoDB Streams.

This allows you to distinguish:

```text id="b6n9q4"
Normal application delete
```

from:

```text id="v3x7k2"
TTL-driven expiration
```

when the stream record metadata indicates the deletion reason.

This is useful for downstream cleanup workflows.

---

# 39. Event-Driven Order Architecture

Let's build a real-world example.

```text id="d9f3k6"
              E-Commerce
                  │
                  ▼
              DynamoDB
                  │
                  ▼
              Streams
                  │
                  ▼
               Lambda
                  │
        ┌─────────┼─────────┐
        ▼         ▼         ▼
   Inventory   Payment   Notification
```

Order update:

```text id="m2x7v4"
PENDING
   ↓
PAID
```

Stream:

```text id="p8c3n5"
MODIFY
```

Lambda:

```text id="r4k9z1"
Publish event
```

Downstream services react independently.

---

# 40. Why Event-Driven Architecture?

Without events:

```text id="v5d8q2"
Order Service
   ↓
Inventory Service
   ↓
Payment Service
   ↓
Notification Service
```

This can create tight coupling.

With events:

```text id="h3m7x9"
Order Service
      ↓
   DynamoDB
      ↓
    Stream
      ↓
    Event
   /   |    \
  ↓    ↓     ↓
Inventory Payment Notification
```

Services become more loosely coupled.

---

# 41. DynamoDB as an Event Source

DynamoDB can become the source of change events:

```text id="j8n4c6"
State Change
     ↓
DynamoDB
     ↓
Stream
     ↓
Event Processing
```

This is especially useful when DynamoDB is the system of record.

---

# 42. Streams + SQS

Sometimes Lambda processing needs additional buffering.

Architecture:

```text id="p2v7m4"
DynamoDB
   ↓
Stream
   ↓
Lambda
   ↓
SQS
   ↓
Worker
```

Why?

Because SQS can provide:

```text id="w8c3n5"
Buffering
Decoupling
Retry behavior
Backpressure handling
```

This can be useful when downstream processing is slow or bursty.

---

# 43. Streams + SNS

If many subscribers need to react to an event:

```text id="q4m8z1"
DynamoDB
   ↓
Stream
   ↓
Lambda
   ↓
SNS
  ├── Email
  ├── Analytics
  └── Notifications
```

SNS can fan out an event to multiple subscribers.

---

# 44. Streams + EventBridge

For broader event-driven architectures, you may transform DynamoDB changes into application events and route them through EventBridge.

Conceptually:

```text id="k7x3p9"
DynamoDB
   ↓
Stream
   ↓
Lambda
   ↓
EventBridge
   ↓
Rules
 ├── Service A
 ├── Service B
 └── Service C
```

This is useful when you want routing based on event characteristics.

---

# 45. Streams vs EventBridge

Don't confuse the two.

### DynamoDB Streams

Focused on:

> **DynamoDB item changes**

### EventBridge

Focused on:

> **Event routing across applications/services**

A common architecture can use both.

---

# 46. Streams vs SQS

### DynamoDB Streams

```text id="f8k3v1"
Database change capture
```

### SQS

```text id="n4c7z2"
Message queue
```

Typical architecture:

```text id="m5x9q3"
DynamoDB
   ↓
Streams
   ↓
Lambda
   ↓
SQS
   ↓
Workers
```

Each service has a different responsibility.

---

# 47. Exactly-Once Myth

A common mistake:

> "My event system guarantees exactly-once business processing."

Distributed systems often require consumers to tolerate retries and duplicate processing.

Therefore:

```text id="c6m2v8"
At-least-once-style processing considerations
          ↓
Idempotent consumers
          ↓
Safe retries
```

should be part of your design.

---

# 48. Duplicate Event Scenario

Suppose:

```text id="q9v3k5"
Order O100 → PAID
```

Lambda receives event.

It charges downstream:

```text id="h4m8z2"
$100
```

Then Lambda fails before acknowledging/finishing processing.

The event may be retried.

Without idempotency:

```text id="s7c1x9"
Charge $100
Charge $100 again
```

With idempotency:

```text id="p5n2m6"
Event ID already processed
       ↓
Skip duplicate effect
```

This is critical in real systems.

---

# 49. Idempotency Key

A downstream system can store an idempotency identifier such as:

```text id="r3x7v9"
EventID
```

or:

```text id="k6m2q4"
OrderId + StateTransition
```

Then:

```text id="c8p1z5"
Already processed?
      │
   ┌──┴──┐
  YES    NO
   │      │
 Skip   Process
          │
          ▼
       Record ID
```

The exact design depends on the business operation.

---

# 50. State Transition Events

`NEW_AND_OLD_IMAGES` is particularly powerful for state machines.

Example:

```text id="w4n8c2"
OLD:
PENDING

NEW:
PAID
```

Your Lambda can determine:

```text id="j5q9m3"
PENDING → PAID
```

Then:

```text id="f7x2k6"
if transition == PENDING → PAID:
    publish PaymentCompleted
```

This is much safer than reacting to:

```text id="a3m8v1"
Status = PAID
```

without knowing the previous state.

---

# 51. Avoid Recursive Stream Loops

This is a critical production problem.

Suppose:

```text id="z8p4k2"
DynamoDB
 ↓
Stream
 ↓
Lambda
 ↓
Updates same DynamoDB item
 ↓
Stream
 ↓
Lambda
 ↓
Updates item
 ↓
...
```

You can accidentally create:

> **An infinite event loop.**

---

# 52. How to Prevent Stream Loops

Possible strategies:

```text id="q6m3v8"
1. Separate source and derived tables
2. Ignore events generated by the consumer
3. Add event-source/type attributes
4. Only update when state actually changes
5. Use conditional writes
6. Carefully design event ownership
```

The best solution depends on the architecture.

---

# 53. Conditional Updates + Streams

Suppose Lambda should only transition:

```text id="v5x2m9"
PENDING → PAID
```

Use conditional logic so invalid or duplicate transitions don't continually generate changes.

This can improve:

```text id="j7c4n8"
Correctness
Idempotency
Loop prevention
Concurrency handling
```

---

# 54. Real-Time Scenario — Customer Profile Sync

Requirement:

> Keep customer data synchronized between DynamoDB and another system.

Architecture:

```text id="b3m8q1"
DynamoDB
   ↓
Stream
   ↓
Lambda
   ↓
External System
```

When:

```text id="p7x4c9"
Customer address changes
```

Lambda receives the change and updates the external system.

### Important considerations

```text id="z2n6m5"
Retry
Idempotency
External API failure
Rate limits
Dead-letter strategy
Monitoring
```

---

# 55. Real-Time Scenario — Session Cleanup

Application creates:

```text id="h5k9v2"
Session
ExpiresAt = 30 minutes later
```

TTL eventually deletes it.

Stream:

```text id="r8m3q6"
REMOVE
```

Lambda:

```text id="c4x7n1"
Invalidate cache
```

This creates automated cleanup without requiring a scheduled cleanup job to scan the table.

---

# 56. Real-Time Scenario — Expired Verification Codes

Store:

```text id="m9p3v6"
Code = 123456
ExpiresAt = timestamp
```

TTL eventually removes it.

The application should still validate expiration itself when security requires it.

Why?

Because TTL deletion isn't immediate.

So:

```text id="w6k2x8"
Application:
"Is current time < expiry?"
```

should remain the authoritative security check.

TTL is cleanup—not your only authorization mechanism.

---

# 57. Real-Time Scenario — Audit Trail

Requirement:

> Every customer update must be recorded.

Architecture:

```text id="j4n8c5"
DynamoDB
   ↓
Stream
   ↓
Lambda
   ↓
S3
```

Store:

```text id="p6x2m9"
Timestamp
Entity
Key
Old state
New state
Actor/context
Event type
```

This creates a durable audit history separate from the short-lived stream.

---

# 58. Real-Time Scenario — Search Index

Requirement:

> DynamoDB stores products; a search engine provides text search.

Architecture:

```text id="r3k7v1"
Product update
      ↓
DynamoDB
      ↓
Stream
      ↓
Lambda
      ↓
Search index
```

If Lambda fails:

```text id="m5q8c2"
Retry
   ↓
Idempotent update
```

This is a common CDC architecture.

---

# 59. Real-Time Scenario — Fan-Out

Requirement:

> When an order is completed, five systems need to react.

Architecture:

```text id="x7n4p8"
DynamoDB
   ↓
Stream
   ↓
Lambda
   ↓
EventBridge/SNS
   │
   ├── Analytics
   ├── Email
   ├── Inventory
   ├── Loyalty
   └── Fraud
```

This reduces direct coupling between the order database and every downstream service.

---

# 60. Real-Time Scenario — High Traffic Event Processing

Suppose:

```text id="k2m8q4"
DynamoDB writes:
100,000/sec
```

Stream consumers need to keep up.

Don't assume:

> "Lambda automatically means infinite processing."

You need to consider:

```text id="v5c9n3"
Stream shards
Lambda concurrency
Batch size
Downstream limits
SQS buffering
Backpressure
Retries
Throttling
```

---

# 61. Event Processing Failure Model

A production design should answer:

```text id="p8x3m7"
What if Lambda fails?
What if downstream API is unavailable?
What if the same event appears again?
What if processing is slow?
What if traffic spikes?
What if an event is malformed?
```

A robust architecture typically includes:

```text id="q4n9c2"
Retries
+
Idempotency
+
Monitoring
+
Failure destination/handling
+
Backpressure
```

---

# 62. Observability

For DynamoDB event-driven systems, monitor:

```text id="m7x2p8"
Stream processing latency
Lambda errors
Lambda throttles
Iterator age
Batch failures
Downstream failures
Retry counts
DLQ/failure destination
DynamoDB throttling
```

One particularly important signal:

> **Iterator age**

If it keeps increasing, consumers are falling behind the stream.

---

# 63. What Does Iterator Age Mean?

Conceptually:

```text id="v3c8m1"
New stream records
       ↓
████████████████
       ↓
Consumer
       ↓
Older records still waiting
```

If iterator age increases:

```text id="f6n2q9"
Events are accumulating faster
than consumers can process them.
```

Possible causes:

```text id="k8m4x3"
Lambda throttling
Slow downstream service
Large batches
Consumer errors
Insufficient processing capacity
```

---

# 64. Backpressure

Suppose:

```text id="r2x7m5"
DynamoDB events = 20,000/sec
Downstream API = 5,000/sec
```

You have a mismatch.

If you directly hammer the downstream API:

```text id="n8c3v6"
Downstream
   ↓
Overloaded
   ↓
Failures
   ↓
Retries
   ↓
More load
```

This can create a failure cascade.

A queue such as SQS can help buffer work:

```text id="p5m9k2"
Stream
 ↓
Lambda
 ↓
SQS
 ↓
Controlled workers
```

---

# 65. Eventual Consistency Across Systems

Suppose:

```text id="w7c2m4"
DynamoDB updated
      ↓
Stream
      ↓
Lambda
      ↓
Search index
```

For a short period:

```text id="g3n8x5"
DynamoDB = NEW
Search    = OLD
```

This is normal in asynchronous architectures.

Your application must decide whether this temporary inconsistency is acceptable.

---

# 66. Source of Truth

In this architecture:

```text id="q9m3v7"
DynamoDB
```

should generally remain the **source of truth** for the data it owns.

Downstream systems are derived views:

```text id="x4n8c2"
DynamoDB
   ↓
Stream
   ↓
Search
   ↓
Cache
   ↓
Analytics
```

If the search index is temporarily stale, the authoritative data remains in DynamoDB.

---

# 67. Eventual Repair

What if the stream consumer fails for a long time?

Because DynamoDB Streams has limited retention, you shouldn't treat it as your forever-replay system.

For critical systems, consider:

```text id="m6p2x8"
Durable event archive
Periodic reconciliation
Source-of-truth rebuild
Dead-letter handling
Operational replay strategy
```

This is an important production consideration.

---

# 68. CDC — Change Data Capture

DynamoDB Streams effectively provides a change-data-capture mechanism for DynamoDB item modifications.

Conceptually:

```text id="z3k7n5"
Database state
     ↓
Change
     ↓
CDC event
     ↓
Downstream systems
```

This pattern is useful for:

```text id="j8m4p2"
Replication
Search synchronization
Analytics
Audit
Notifications
Derived data
```

---

# 69. Event-Driven Architecture Best Practices

### 1. Keep consumers idempotent

```text id="w5n2x9"
Duplicate event
    ↓
Same final result
```

### 2. Don't assume immediate processing

```text id="c7m3k8"
Design for eventual consistency.
```

### 3. Monitor lag

```text id="p4x9m1"
Iterator age
```

### 4. Protect downstream services

```text id="q8n5v2"
Queues
Rate limits
Backpressure
```

### 5. Prevent recursive loops

```text id="z6m3c7"
Source → Consumer → Source
```

must be carefully controlled.

---

# 70. Top 30 Interview Questions — Lesson 6

### 1. What is DynamoDB Streams?

A time-ordered stream of item-level modifications made to a DynamoDB table.

### 2. What events are captured?

```text id="v8c2m5"
INSERT
MODIFY
REMOVE
```

### 3. What are the four stream view types?

```text id="p4x7n9"
KEYS_ONLY
NEW_IMAGE
OLD_IMAGE
NEW_AND_OLD_IMAGES
```

### 4. What is NEW_IMAGE?

The item's state after the modification.

### 5. What is OLD_IMAGE?

The item's state before the modification.

### 6. What is NEW_AND_OLD_IMAGES useful for?

Determining state transitions.

### 7. How long are DynamoDB Streams records retained?

**24 hours.**

### 8. Can DynamoDB Streams be used as a permanent event store?

No.

### 9. Can Streams trigger Lambda?

Yes.

### 10. Are stream consumers synchronous with DynamoDB writes?

No. Processing is asynchronous.

### 11. What is TTL?

A mechanism for automatically expiring DynamoDB items after their configured expiration time.

### 12. Does TTL delete an item exactly at the expiration timestamp?

No. TTL deletion is asynchronous.

### 13. What happens when TTL deletes an item?

The deletion can appear as a removal event in DynamoDB Streams.

### 14. Can TTL be used as an exact scheduler?

No.

### 15. What is idempotency?

The ability to safely process the same event more than once without producing an incorrect duplicate effect.

### 16. Why is idempotency important?

Because distributed event processing can involve retries and duplicate processing.

### 17. What is iterator age?

An indicator of how far behind a stream consumer is processing the stream.

### 18. What does increasing iterator age indicate?

Consumers may be falling behind incoming stream records.

### 19. Can Streams trigger an infinite loop?

Yes, if a consumer updates the same table and repeatedly generates events without proper safeguards.

### 20. How do you prevent stream loops?

Use controlled state transitions, event-source markers, separate tables, conditional writes, and careful event ownership.

### 21. Can Streams synchronize DynamoDB with a search system?

Yes.

### 22. Can Streams support audit pipelines?

Yes.

### 23. Can Streams be combined with SQS?

Yes, commonly for buffering and decoupling.

### 24. Can Streams be combined with EventBridge?

Yes, often by transforming DynamoDB changes into application-level events.

### 25. Is DynamoDB Streams the same as SQS?

No.

### 26. Is DynamoDB Streams the same as EventBridge?

No.

### 27. What is CDC?

Change Data Capture—the process of capturing data changes and propagating them to downstream systems.

### 28. What happens if downstream processing is slower than event generation?

Backlog can grow, increasing consumer lag/iterator age; buffering and controlled processing may be needed.

### 29. Should downstream consumers assume exactly-once business processing?

No. Design for retries and idempotency.

### 30. What is the most important event-driven principle?

> **Assume asynchronous processing, retries, duplicates, failures, and temporary inconsistency—and design for them explicitly.**

---

# 🧠 Lesson 6 — Production Architecture

```text id="f1a8c6"
                         APPLICATION
                              │
                              ▼
                         DYNAMODB
                              │
             ┌────────────────┴────────────────┐
             │                                 │
             ▼                                 ▼
         STREAMS                              TTL
             │                                 │
             ▼                                 ▼
          Lambda                        Expiration
             │                                 │
      ┌──────┼──────┐                         ▼
      ▼      ▼      ▼                       REMOVE
     SQS   SNS   EventBridge                  │
      │      │      │                         │
      └──────┼──────┘                         │
             ▼                                │
      Downstream Systems ◄────────────────────┘
             │
             ▼
       Idempotency
             │
             ▼
       Retry / Failure
             │
             ▼
       Monitoring
             │
             ▼
       Reconciliation
```

# 🔥 Lesson 6 Golden Rules

1. **DynamoDB Streams captures INSERT, MODIFY, and REMOVE changes.**
2. **Choose the stream view type based on what the consumer actually needs.**
3. **`NEW_AND_OLD_IMAGES` is excellent for state-transition processing.**
4. **DynamoDB Streams retention is 24 hours, so don't treat it as permanent storage.**
5. **Streams are asynchronous.**
6. **Design consumers to tolerate retries and duplicates.**
7. **Idempotency is mandatory for important side effects.**
8. **TTL is for eventual expiration, not exact-time scheduling.**
9. **TTL deletion can feed DynamoDB Streams.**
10. **Monitor stream consumer lag/iterator age.**
11. **Protect slow downstream services with buffering and backpressure where appropriate.**
12. **Prevent recursive DynamoDB → Stream → Lambda → DynamoDB loops.**
13. **DynamoDB should generally remain the source of truth; downstream systems can be derived views.**
14. **For critical pipelines, think about reconciliation and recovery—not just the happy path.**
15. **Event-driven architecture is powerful because it decouples database changes from downstream business actions.**

## 🎯 Lesson 6 takeaway

Remember this production chain:

> **DynamoDB State Change → Stream → Consumer → Event → Downstream Service → Retry/Idempotency → Monitoring**

And for temporary data:

> **DynamoDB → TTL → Eventual Expiration → Stream REMOVE → Cleanup Workflow**

**Next: Lesson 7 — Transactions, Conditional Writes, Optimistic Locking & Concurrency Control: how to maintain correctness when multiple users/services modify DynamoDB simultaneously.**
