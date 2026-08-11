# ⚡ Amazon DynamoDB — Lesson 4

## Capacity, Performance & Scaling

This lesson is about one of the most important **production DynamoDB skills**:

> **How do you make DynamoDB handle increasing traffic without throttling, excessive latency, or unnecessary cost?**

The core concepts are:

```text
RCU / WCU
   ↓
Capacity Modes
   ↓
Partition Distribution
   ↓
Hot Partitions
   ↓
Throttling
   ↓
Adaptive Capacity
   ↓
Auto Scaling
   ↓
Performance Monitoring
   ↓
Cost Optimization
```

---

# 1. What Does "Capacity" Mean in DynamoDB?

DynamoDB capacity represents how much read and write workload your table/index can handle.

The two fundamental concepts are:

```text
RCU = Read Capacity Unit
WCU = Write Capacity Unit
```

Think of them as the units DynamoDB uses to measure read and write throughput under its capacity model.

---

# 2. Why Capacity Matters

Imagine your application normally receives:

```text
1,000 requests/sec
```

but during a sale:

```text
50,000 requests/sec
```

If your DynamoDB configuration and data distribution cannot accommodate the workload, you may see:

```text
Request
   ↓
DynamoDB
   ↓
Throttling
   ↓
Application errors/retries
   ↓
Higher latency
```

Therefore, capacity planning isn't simply:

> "How many requests per second do I have?"

You must also consider:

```text
Item size
Read/write mix
Consistency
Key distribution
Indexes
Traffic patterns
Burst behavior
```

---

# 3. RCU — Read Capacity Unit

RCU represents read capacity.

For the common DynamoDB provisioned/on-demand capacity model:

* One RCU provides capacity for **one strongly consistent read per second for an item up to 4 KB**.
* Eventually consistent reads consume **half as much read capacity** for the same 4 KB unit.
* Transactional reads consume **twice the read capacity** of strongly consistent reads.

### Simplified mental model

```text
4 KB
 │
 ├── Strongly consistent → 1 RCU
 │
 ├── Eventually consistent → 0.5 RCU
 │
 └── Transactional → 2 RCU
```

The exact capacity calculation becomes important when sizing high-volume workloads.

---

# 4. WCU — Write Capacity Unit

WCU represents write capacity.

For the common capacity model:

* One WCU provides capacity for **one write per second for an item up to 1 KB**.
* Larger items consume proportionally more capacity.
* Transactional writes consume more capacity than ordinary writes.

Simplified:

```text
1 KB write
    ↓
1 WCU
```

So:

```text
5 KB write
    ↓
Approximately 5 WCUs
```

for a standard non-transactional write, subject to DynamoDB's capacity calculation rules.

---

# 5. Why Item Size Matters

Suppose you have:

```text
Item A = 1 KB
Item B = 10 KB
```

A workload performing 1,000 writes/sec of Item B requires substantially more write capacity than 1,000 writes/sec of Item A.

Therefore:

> **Request rate alone is not enough for capacity planning.**

You need:

```text
Requests/sec
×
Item size
×
Operation characteristics
```

---

# 6. Read Example

Suppose you have:

```text
Item size = 4 KB
Read rate = 1,000 reads/sec
```

For strongly consistent reads:

```text
1 read × 1 RCU
× 1,000
=
1,000 RCUs
```

For eventually consistent reads:

```text
1 read × 0.5 RCU
× 1,000
=
500 RCUs
```

This demonstrates why consistency selection can affect capacity consumption.

---

# 7. Write Example

Suppose:

```text
Item size = 1 KB
Write rate = 2,000 writes/sec
```

Approximately:

```text
2,000 WCU
```

for standard writes under the simplified capacity model.

If the item becomes:

```text
10 KB
```

the write capacity requirement increases significantly.

---

# 8. Capacity Mode

DynamoDB provides two primary capacity modes:

```text
1. Provisioned capacity
2. On-demand capacity
```

Choosing correctly is a major production decision.

---

# 9. Provisioned Capacity

With provisioned capacity, you specify expected read/write capacity.

Conceptually:

```text
Application
     ↓
Expected workload
     ↓
Provisioned RCU/WCU
     ↓
DynamoDB
```

Example:

```text
RCU = 5,000
WCU = 2,000
```

### Useful when

1. Workload is relatively predictable.
2. Traffic patterns are stable.
3. You want more explicit capacity control.
4. Cost optimization is important for sustained predictable workloads.
5. Auto Scaling can adjust capacity as workload changes.
6. You understand your traffic profile.

---

# 10. On-Demand Capacity

On-demand mode is designed for workloads where you don't want to manage provisioned throughput directly.

Conceptually:

```text
Traffic
  ↓
DynamoDB
  ↓
Capacity automatically adapts
```

It's particularly useful for:

```text
Unpredictable traffic
New applications
Spiky workloads
Variable workloads
Applications where simplicity is important
```

---

# 11. Provisioned vs On-Demand

| Characteristic                   | Provisioned                   | On-Demand             |
| -------------------------------- | ----------------------------- | --------------------- |
| Explicit capacity configuration  | Yes                           | No                    |
| Good for predictable workloads   | Excellent                     | Good                  |
| Good for unpredictable workloads | Requires planning             | Excellent             |
| Auto Scaling available           | Yes                           | Not required          |
| Operational complexity           | Higher                        | Lower                 |
| Cost optimization                | Can be strong for steady load | Pay-per-request model |
| Capacity planning                | Important                     | Simplified            |

### Interview answer

Don't say:

> "On-demand is always better."

Instead:

> **"I choose capacity mode based on workload predictability, traffic patterns, cost requirements, and operational simplicity."**

---

# 12. What Is Auto Scaling?

With provisioned capacity, DynamoDB Auto Scaling can adjust provisioned throughput based on workload.

Conceptually:

```text
Traffic increases
       ↓
Utilization increases
       ↓
Auto Scaling
       ↓
Provisioned capacity increases
```

When traffic decreases:

```text
Traffic decreases
       ↓
Utilization decreases
       ↓
Capacity can scale down
```

This helps reduce manual capacity management.

---

# 13. Auto Scaling Is Not Magic

A common interview mistake is:

> "Auto Scaling prevents all throttling."

Not necessarily.

Scaling takes time.

If traffic suddenly jumps:

```text
100 requests/sec
      ↓
50,000 requests/sec
```

capacity may not instantly match the new workload.

Therefore, for sudden unpredictable spikes, you need to consider:

```text
On-demand capacity
Traffic shaping
Caching
Pre-scaling
Good key distribution
Retry behavior
```

---

# 14. What Is a Partition?

DynamoDB distributes data across underlying partitions.

Conceptually:

```text
DynamoDB Table
      │
 ┌────┼────┬────┐
 ▼    ▼    ▼    ▼
 P1   P2   P3   P4
```

Your application doesn't normally manage these physical partitions directly.

Instead, your responsibility is to create a data model that allows DynamoDB to distribute workload effectively.

---

# 15. Why Partition-Key Design Matters

Suppose you have:

```text
PK = CustomerId
```

and millions of customers:

```text
C001
C002
C003
...
C999999
```

This can provide good distribution if traffic is reasonably distributed.

But if you use:

```text
PK = Country
```

and most traffic is:

```text
USA
```

then a large percentage of requests may concentrate around:

```text
PK = USA
```

This can create a **hot key**.

---

# 16. What Is a Hot Partition?

A hot partition occurs when workload becomes disproportionately concentrated on a small portion of the data.

Example:

```text
                 DynamoDB
                    │
       ┌────────────┼────────────┐
       ▼            ▼            ▼
      P1            P2           P3
      │             │            │
      ▼             ▼            ▼
Normal load      Normal load   HUGE LOAD
                                ↑
                            Hot partition
```

The table may have plenty of total capacity, but one portion of the workload is disproportionately heavy.

---

# 17. Hot Key vs Hot Partition

These terms are related but not identical.

### Hot key

One partition-key value receives unusually high traffic.

Example:

```text
PRODUCT#IPHONE
```

receives millions of requests.

### Hot partition

The underlying physical partition becomes disproportionately loaded.

A hot key can contribute to a hot partition.

### Interview-safe answer

> **"A hot key is a logical key receiving disproportionate traffic; this can lead to uneven physical partition utilization and throttling."**

---

# 18. Example — Viral Product

Imagine an e-commerce application.

Normally:

```text
Product A → 100 requests/sec
Product B → 100 requests/sec
Product C → 100 requests/sec
```

Then Product A goes viral:

```text
Product A → 100,000 requests/sec
```

If the application repeatedly targets:

```text
PK = PRODUCT#A
```

you have a concentrated access pattern.

This is a classic hot-key scenario.

---

# 19. Why Total Capacity Isn't Enough

Suppose your table appears to support:

```text
100,000 WCU
```

You might think:

> "We can handle 100,000 writes/sec."

Not necessarily.

If workload distribution is poor:

```text
Most writes → One/few keys
```

you can still experience throttling.

Therefore:

> **Think about capacity AND distribution.**

---

# 20. Adaptive Capacity

DynamoDB has mechanisms that can help accommodate uneven workloads, including **adaptive capacity**.

The idea is that DynamoDB can dynamically allocate available capacity more effectively to partitions experiencing higher demand.

But don't interpret adaptive capacity as:

> "I can use any key design I want."

You should still design partition keys for good distribution.

### Golden rule

> **Adaptive capacity helps good designs; it does not eliminate the need for good key design.**

---

# 21. Write Sharding

Suppose one logical key is extremely hot:

```text
PK = PRODUCT#A
```

You could sometimes distribute writes across multiple logical shard keys.

For example:

```text
PRODUCT#A#0
PRODUCT#A#1
PRODUCT#A#2
PRODUCT#A#3
```

Then:

```text
Incoming writes
      │
 ┌────┼────┬────┐
 ▼    ▼    ▼    ▼
 #0   #1   #2   #3
```

This is called **write sharding** or key sharding.

---

# 22. Write Sharding Trade-Off

Sharding distributes workload, but introduces complexity.

Without sharding:

```text
Read one key
```

With sharding:

```text
Read shard 0
Read shard 1
Read shard 2
Read shard 3
       ↓
Combine results
```

Therefore:

> **Don't shard everything.**

Use it when a workload genuinely has a concentrated hot-key problem.

---

# 23. Read Hot Keys

Hot keys aren't only about writes.

Suppose:

```text
PRODUCT#IPHONE
```

is requested millions of times per minute.

Even if the data rarely changes, reads can become concentrated.

Potential solutions can include:

```text
Caching
CloudFront where appropriate
Application-level caching
DynamoDB Accelerator (DAX) where appropriate
Better access distribution
```

The solution depends on the workload.

---

# 24. What Is DAX?

**Amazon DynamoDB Accelerator (DAX)** is a managed in-memory cache designed specifically for DynamoDB workloads.

Conceptually:

```text
Application
    ↓
   DAX
    │
 ┌──┴──┐
 ▼     ▼
Cache  DynamoDB
```

For workloads requiring extremely fast repeated reads, DAX can reduce read latency and reduce some read traffic reaching DynamoDB.

But:

> **DAX does not fix a fundamentally bad data model.**

---

# 25. DynamoDB and Caching

Suppose:

```text
Product P001
```

is requested:

```text
1 million times
```

but changes rarely.

Caching can be much more efficient than making every request hit DynamoDB.

Conceptually:

```text
Request
  ↓
Cache hit?
 ├── Yes → Return
 └── No
       ↓
   DynamoDB
       ↓
    Cache result
```

Caching is particularly useful for highly popular read-heavy objects.

---

# 26. Throttling

A DynamoDB request can be throttled when the workload exceeds available capacity or encounters other service limits.

Conceptually:

```text
Traffic
  ↓
Available capacity
  ↓
Exceeded
  ↓
Throttling
```

Possible causes include:

```text
Insufficient provisioned capacity
Hot partitions
Hot keys
Sudden traffic spikes
GSI capacity constraints
Other service limits
```

---

# 27. Throttling Does NOT Always Mean "Increase Capacity"

This is an important production lesson.

Suppose:

```text
Table capacity = 100,000 WCU
```

but one key receives disproportionate traffic.

Increasing the total capacity may not solve the underlying distribution problem.

You should investigate:

```text
Which operation?
Which key?
Which partition?
Base table or GSI?
What is the traffic distribution?
```

---

# 28. Throttling Troubleshooting Flow

Use this:

```text
Throttling
    ↓
Which operation?
    ↓
Read or Write?
    ↓
Base table or GSI?
    ↓
Provisioned or On-Demand?
    ↓
Check consumed capacity
    ↓
Check throttled requests
    ↓
Check key distribution
    ↓
Look for hot keys
    ↓
Check item size
    ↓
Check traffic spike
    ↓
Choose solution
```

Possible solutions:

```text
Increase capacity
Change capacity mode
Fix key distribution
Add caching
Use sharding
Optimize access patterns
Reduce unnecessary traffic
```

---

# 29. Base Table vs GSI Throttling

This is a very important production issue.

Suppose:

```text
Table
 ├── Base table
 ├── GSI-1
 └── GSI-2
```

Your application writes an item.

That write may also affect an index.

Therefore, throttling can be associated with:

```text
Base table
```

or:

```text
GSI
```

You need to identify **where the bottleneck actually is** before changing configuration.

---

# 30. GSI Capacity Problem

Imagine:

```text
Base table
WCU = sufficient

GSI
WCU = insufficient
```

A write that affects the GSI can experience problems related to the index.

Therefore:

> **When troubleshooting writes, don't look only at the base table.**

Always consider indexes.

We'll go deep into GSI/LSI behavior in Lesson 5.

---

# 31. Bursty Traffic

Consider:

```text
Normal:
1,000 req/sec

Sale starts:
100,000 req/sec
```

A sudden traffic spike is called a burst/spike workload.

Your design should consider:

```text
Capacity mode
Pre-scaling if applicable
Caching
Traffic control
Retry strategy
Key distribution
Application architecture
```

---

# 32. On-Demand for Unpredictable Traffic

If your workload looks like:

```text
10 AM → 1,000 req/sec
11 AM → 2,000
12 PM → 50,000
1 PM  → 500
2 PM  → 100,000
```

on-demand capacity can be attractive because you don't have to continuously manage provisioned capacity for every fluctuation.

But cost should still be evaluated against your actual workload.

---

# 33. Provisioned for Predictable Workloads

Suppose your traffic is:

```text
Every day:
09:00 → 5,000 req/sec
10:00 → 5,500
11:00 → 5,200
...
```

Provisioned capacity with Auto Scaling may provide a good balance between:

```text
Cost
+
Control
+
Automation
```

---

# 34. Cost Optimization

DynamoDB cost optimization starts with the data model.

Important areas:

```text
Item size
Read/write frequency
Capacity mode
Indexes
Unused attributes
Duplicate data
Scan usage
Caching
TTL
Access patterns
```

For example:

```text
Bad:
Scan 100 million items
to find 1 item

Better:
GetItem using known key
```

The second design can be dramatically more efficient.

---

# 35. Why Smaller Items Help

Suppose an item contains:

```text
Customer profile
+
10 MB historical data
+
Large metadata
```

but your API only needs:

```text
Name
Email
Status
```

Putting unnecessary data into the hot item can increase:

```text
Storage
Read/write capacity
Network transfer
Latency
```

Consider separating rarely accessed large data when appropriate.

---

# 36. Time-to-Live and Cost

Suppose you store temporary sessions:

```text
Session
CreatedAt
ExpiresAt
```

If old sessions remain indefinitely, storage grows.

DynamoDB TTL can automatically remove expired items over time.

Common candidates:

```text
Sessions
Temporary tokens
Ephemeral state
Temporary caches
Expired records
```

TTL is not an exact real-time deletion mechanism, so applications shouldn't depend on deletion happening at an exact second.

---

# 37. Performance Metrics

In production, monitor metrics such as:

```text
ConsumedReadCapacityUnits
ConsumedWriteCapacityUnits
ThrottledRequests
SuccessfulRequestLatency
SystemErrors
UserErrors
```

Also examine metrics related to:

```text
GSI
Partitions
Operations
Application traffic
```

The exact metrics and dimensions available depend on the DynamoDB configuration and AWS monitoring tools you're using.

---

# 38. Latency vs Throughput

These are different.

### Latency

How long one operation takes.

```text
Request
   ↓
10 ms
```

### Throughput

How much workload the system processes.

```text
10,000 requests/sec
```

You can have:

```text
High throughput
+
Low latency
```

which is often a DynamoDB goal.

But bad data distribution or throttling can negatively affect both.

---

# 39. Performance Optimization Sequence

When DynamoDB performance is poor, don't immediately increase capacity.

Use this order:

```text
1. Identify slow operation
       ↓
2. Check access pattern
       ↓
3. Check Query vs Scan
       ↓
4. Check key distribution
       ↓
5. Check item size
       ↓
6. Check throttling
       ↓
7. Check base table/GSI
       ↓
8. Check capacity mode
       ↓
9. Consider caching
       ↓
10. Re-test
```

This is much more effective than blindly increasing capacity.

---

# 40. Real-Time Scenario — Throttling During Sale

### Situation

An e-commerce application normally receives:

```text
5,000 writes/sec
```

During a flash sale:

```text
80,000 writes/sec
```

Suddenly you see throttling.

### Investigation

Ask:

```text
Is capacity provisioned?
Is Auto Scaling configured?
Did traffic spike suddenly?
Are writes distributed?
Is one product extremely popular?
Are GSIs also receiving writes?
```

### Possible solution

Depending on the findings:

```text
On-demand capacity
+
Better key distribution
+
Caching
+
Write sharding
+
Appropriate retry/backoff
```

There is no universal one-line fix.

---

# 41. Real-Time Scenario — One Hot Product

### Situation

You have:

```text
10 million products
```

but:

```text
PRODUCT#IPHONE
```

receives 90% of requests.

### Problem

The logical workload is extremely concentrated.

### Possible solution

For read-heavy workloads:

```text
Cache popular product
```

For write-heavy workloads:

```text
Consider write sharding
```

For mixed workloads:

```text
Evaluate access pattern,
caching, sharding,
and data-model changes together.
```

---

# 42. Real-Time Scenario — GSI Throttling

### Situation

Base table appears healthy.

But writes are being throttled.

You discover:

```text
Base Table → healthy
GSI → heavily consumed
```

### Diagnosis

The index is part of the write path.

### Lesson

> **Always investigate GSIs when diagnosing DynamoDB write performance.**

---

# 43. Real-Time Scenario — Query Is Slow

Suppose:

```text
GET /customer/orders
```

takes too long.

You discover:

```text
Query
PK = CUSTOMER#001
```

returns:

```text
500,000 items
```

The Query itself may be key-based, but you're retrieving an enormous item collection.

### Better approach

Consider:

```text
Pagination
Time-based sort keys
More targeted sort-key conditions
Archiving old data
Alternative access pattern
```

The problem isn't always "Query vs Scan."

Sometimes the Query is simply too broad.

---

# 44. Real-Time Scenario — Scan in Production

Application:

```text
GET /customer?email=x@example.com
```

Implementation:

```text
Scan entire table
```

Traffic:

```text
10,000 requests/min
```

### Diagnosis

The application has a bad access pattern.

### Better solution

Create an access path such as:

```text
Email → GSI → Customer
```

rather than repeatedly scanning.

This connects Lesson 2 and Lesson 4:

> **Good data modeling is a performance optimization.**

---

# 45. Real-Time Scenario — Traffic Suddenly Drops

Suppose:

```text
10,000 req/sec
```

drops to:

```text
100 req/sec
```

With provisioned capacity, Auto Scaling can reduce provisioned throughput according to its configuration.

With on-demand capacity, you don't manually reduce provisioned capacity.

This illustrates the operational difference between the two capacity modes.

---

# 46. Capacity Planning Checklist

Before production launch, estimate:

### Reads

```text
Average reads/sec
Peak reads/sec
Item size
Consistency
Query vs GetItem
```

### Writes

```text
Average writes/sec
Peak writes/sec
Item size
Conditional writes
Transactions
```

### Indexes

```text
Number of GSIs
Indexed item size
GSI write/read traffic
```

### Distribution

```text
Partition-key cardinality
Popular keys
Expected hot keys
Traffic distribution
```

### Operations

```text
Retry behavior
Caching
Monitoring
Alerting
Capacity mode
```

---

# 47. DynamoDB Scaling Strategy

Think in four layers:

```text
                 SCALING
                    │
       ┌────────────┼────────────┐
       ▼            ▼            ▼
   Capacity       Data          Traffic
     Mode         Model        Management
       │            │             │
       ▼            ▼             ▼
On-demand/      Good keys      Caching
Provisioned     Sharding       Backoff
Auto Scaling    Indexes        Throttling
       │            │             │
       └────────────┼─────────────┘
                    ▼
               Monitoring
```

---

# 48. The DynamoDB Performance Hierarchy

When optimizing performance, think in this order:

```text
1. Correct access pattern
        ↓
2. Correct primary key
        ↓
3. Good partition-key distribution
        ↓
4. Efficient Query/GetItem
        ↓
5. Appropriate item size
        ↓
6. Appropriate indexes
        ↓
7. Correct capacity mode
        ↓
8. Caching
        ↓
9. Application retry strategy
        ↓
10. Continuous monitoring
```

This prevents a common mistake:

> **Trying to solve a data-model problem with more capacity.**

---

# 49. Top 20 Interview Questions — Lesson 4

### Q1. What is RCU?

Read Capacity Unit — a unit used to measure DynamoDB read throughput.

### Q2. What is WCU?

Write Capacity Unit — a unit used to measure DynamoDB write throughput.

### Q3. How does item size affect capacity?

Larger items consume more read/write capacity than smaller items.

### Q4. What are DynamoDB capacity modes?

```text
Provisioned
On-demand
```

### Q5. When would you choose on-demand?

For workloads with unpredictable or rapidly changing traffic where operational simplicity and automatic capacity handling are valuable.

### Q6. When would you choose provisioned?

For predictable workloads where explicit capacity control and cost optimization for sustained traffic are important.

### Q7. What is Auto Scaling?

A mechanism that adjusts provisioned DynamoDB capacity according to workload utilization.

### Q8. What is a hot key?

A partition-key value receiving disproportionately high traffic.

### Q9. What is a hot partition?

A physical partition experiencing disproportionately high workload.

### Q10. Does increasing total capacity always fix throttling?

**No.** Hot keys, partition imbalance, indexes, sudden spikes, or other limits may be the underlying cause.

### Q11. What is adaptive capacity?

DynamoDB's ability to better allocate available capacity to partitions experiencing uneven workloads.

### Q12. Does adaptive capacity eliminate the need for good partition keys?

**No.**

### Q13. What is write sharding?

Distributing a hot logical key across multiple shard keys to spread workload.

### Q14. What is DAX?

A managed in-memory cache designed specifically for DynamoDB workloads.

### Q15. Why can a GSI cause write performance issues?

Writes that affect indexed attributes can also require index updates, creating additional capacity requirements.

### Q16. What is the first thing to investigate when DynamoDB throttles?

Identify the operation, resource, workload distribution, capacity mode, and whether a hot key or index is involved.

### Q17. Why is Scan dangerous at scale?

Because it can examine large amounts of data and consume significant resources compared with targeted key-based operations.

### Q18. Can Query still be inefficient?

Yes. A Query can retrieve an excessively large result set if the key condition is too broad.

### Q19. How can caching improve DynamoDB performance?

It can serve frequently requested data without sending every request to DynamoDB.

### Q20. What is the most important performance principle?

> **Fix the data model and access pattern before throwing more capacity at the problem.**

---

# 🧠 Lesson 4 — Final Production Mental Model

```text
                         APPLICATION
                              │
                              ▼
                       ACCESS PATTERN
                              │
                              ▼
                       KEY DISTRIBUTION
                              │
                 ┌────────────┴────────────┐
                 ▼                         ▼
             READ PATH                  WRITE PATH
                 │                         │
             Get/Query              Put/Update/Delete
                 │                         │
                 └────────────┬────────────┘
                              ▼
                           DYNAMODB
                              │
                 ┌────────────┼────────────┐
                 ▼            ▼            ▼
              Capacity      GSI         Partitions
                 │            │            │
                 ▼            ▼            ▼
            RCU / WCU    Extra Load    Distribution
                 │                         │
                 └────────────┬────────────┘
                              ▼
                         THROTTLING?
                              │
                    ┌─────────┴─────────┐
                    ▼                   ▼
                   NO                  YES
                    │                   │
                    ▼                   ▼
                Monitor          Diagnose Root Cause
                                      │
                          ┌───────────┼───────────┐
                          ▼           ▼           ▼
                       Capacity    Hot Key      GSI
                          │           │           │
                          ▼           ▼           ▼
                       Scale       Shard       Optimize
                          │           │           │
                          └───────────┼───────────┘
                                      ▼
                                   Re-test
```

# 🔥 Lesson 4 Golden Rules

1. **RCU/WCU depend on more than request count — item size and operation characteristics matter.**
2. **Choose provisioned vs on-demand based on workload behavior, not fashion.**
3. **A high-capacity table can still suffer from hot keys.**
4. **Adaptive capacity helps, but good partition-key design remains essential.**
5. **Don't solve every throttling problem by simply increasing capacity.**
6. **Always investigate GSIs when diagnosing write problems.**
7. **Use caching for extremely hot read-heavy data when appropriate.**
8. **Use write sharding only when the workload genuinely requires it.**
9. **Query is not automatically efficient if the result set is enormous.**
10. **The best DynamoDB performance optimization is usually a good access-pattern/data-model design.**

## 🎯 Lesson 4 takeaway

The production mindset should be:

> **Traffic → Access Pattern → Key Distribution → Capacity → Partitions → Indexes → Throttling → Root Cause → Optimization → Monitoring**

**Next: Lesson 5 — Indexes & Advanced Querying: GSI, LSI, sparse indexes, projections, index capacity, index design, Query vs Index Query, and real-world GSI/LSI scenarios.**
