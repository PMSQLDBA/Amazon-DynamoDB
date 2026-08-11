# ⚡ Amazon DynamoDB — Lesson 9

## Global Tables, Multi-Region Architecture, Disaster Recovery, Backup/Restore & Advanced Production Design

This lesson focuses strictly on **production-grade DynamoDB architecture** when you need:

* Multi-Region availability
* Disaster recovery
* Global applications
* Backup and restore
* Regional failure handling
* Failover/failback
* Advanced production architecture

The core architecture is:

```text
                         GLOBAL APPLICATION
                                │
              ┌─────────────────┼─────────────────┐
              ▼                 ▼                 ▼
           Region A           Region B          Region C
              │                 │                 │
              ▼                 ▼                 ▼
        DynamoDB Table   ⇄  DynamoDB Table  ⇄  DynamoDB Table
              │                 │                 │
              └─────────────────┼─────────────────┘
                                │
                         Global Tables
                                │
                    ┌───────────┴───────────┐
                    ▼                       ▼
                 Backup                   PITR
                    │                       │
                    └───────────┬───────────┘
                                ▼
                         Disaster Recovery
```

---

# 1. Why Multi-Region DynamoDB?

A single AWS Region can experience:

```text
Regional service disruption
Network problems
Application deployment failure
Configuration errors
Large-scale operational incidents
```

If your application must continue operating despite a regional failure, you need a multi-Region architecture.

---

# 2. Single-Region Architecture

Basic architecture:

```text
Users
  ↓
Application
  ↓
DynamoDB
  ↓
Region A
```

Advantages:

```text
Simple
Lower operational complexity
Lower cross-region data transfer
Easier troubleshooting
```

Disadvantages:

```text
Regional dependency
Potential regional outage impact
Higher disaster-recovery complexity
```

---

# 3. Multi-Region Architecture

With DynamoDB Global Tables:

```text
                  Users
                    │
             Global Routing
                    │
       ┌────────────┼────────────┐
       ▼            ▼            ▼
   Region A      Region B      Region C
       │            │            │
       ▼            ▼            ▼
  DynamoDB       DynamoDB      DynamoDB
    Table          Table         Table
       ╲            │            ╱
        ╲           │           ╱
         └──── Global Tables ───┘
```

Applications in multiple Regions can access local replicas.

---

# 4. What Are DynamoDB Global Tables?

Global Tables provide a managed multi-Region, multi-active replication capability for DynamoDB.

Conceptually:

```text
Region A
   │
   ▼
Table Replica A
   ⇄
Table Replica B
   ▲
   │
Region B
```

Changes made in one Region are replicated to other participating Regions.

---

# 5. Multi-Active Architecture

A major advantage is:

> **Applications can read and write in multiple Regions.**

For example:

```text
US users
   ↓
US Region
   ↓
Local DynamoDB replica

European users
   ↓
EU Region
   ↓
Local DynamoDB replica
```

This can reduce latency while providing regional resilience.

---

# 6. Global Tables vs Read Replicas

Do not confuse DynamoDB Global Tables with database read replicas from traditional relational databases.

Traditional model:

```text
Primary
   ↓
Read replicas
```

Global Tables:

```text
Region A
   ⇄
Region B
   ⇄
Region C
```

They are designed for multi-Region applications and replication.

---

# 7. Multi-Region Read Latency

Suppose your users are distributed globally:

```text
US
Europe
Asia
```

Without multi-Region data:

```text
Asia
  ↓
US DynamoDB
  ↓
Higher network latency
```

With Global Tables:

```text
Asia
  ↓
Asia DynamoDB replica
  ↓
Lower network latency
```

This can improve global user experience.

---

# 8. Global Tables Replication

Conceptually:

```text
Write in Region A
        │
        ▼
DynamoDB replica A
        │
        ▼
Replication
        │
   ┌────┴────┐
   ▼         ▼
Region B   Region C
```

The replication is managed by AWS rather than requiring you to build your own replication pipeline.

---

# 9. Important: Replication Is Not the Same as Strong Global Consistency

A common misconception:

> "Global Tables means every Region sees every write immediately."

Don't design your application assuming zero replication latency.

Cross-Region replication involves distributed systems behavior.

Therefore you must understand:

```text
Local write
   ↓
Replication
   ↓
Other Region
```

and design business workflows appropriately.

---

# 10. Multi-Region Conflict

Now consider:

```text
Region A:
User updates Profile.Name = Alice

Region B:
User updates Profile.Name = Alicia
```

What happens when both updates target the same item?

This is a:

> **Concurrent multi-Region write conflict**

Your application needs a conflict-aware design.

---

# 11. Last-Writer-Wins

DynamoDB Global Tables uses a conflict-resolution mechanism based on the latest write for conflicting item updates.

Therefore, you should avoid architectures where two Regions frequently modify the same item independently unless you understand and accept the conflict semantics.

---

# 12. Avoiding Conflicts Through Data Ownership

A powerful pattern is:

> **Assign ownership of writes to a Region.**

Example:

```text
User A → Region US
User B → Region EU
User C → Region APAC
```

Then:

```text
User A data
    ↓
US primarily owns writes

User B data
    ↓
EU primarily owns writes
```

Reads can remain globally distributed.

This reduces conflicting writes.

---

# 13. Single-Writer Per Entity

Another strong pattern:

```text
Entity
  ↓
Home Region
  ↓
Only that Region writes
```

For example:

```text
Customer#100
    ↓
US

Customer#200
    ↓
Europe
```

Other Regions can read replicated data.

This gives you:

```text
Local reads
+
Controlled writes
+
Reduced conflicts
```

---

# 14. Multi-Writer Per Entity

Sometimes you genuinely need:

```text
Region A → write
Region B → write
```

to the same logical entity.

Then you must design for conflict resolution.

Ask:

```text
Can writes be merged?
Can fields be independently owned?
Can updates be idempotent?
Can operations be represented as events?
```

If not, multi-writer designs can become difficult.

---

# 15. Field-Level Ownership

Instead of both Regions modifying the same attributes:

```text
Region A owns:
Address

Region B owns:
Preferences
```

Then independent updates can target different attributes.

Example:

```text
Region A:
Update Address

Region B:
Update Language
```

This can reduce conflicting writes.

---

# 16. Global Tables and Application Routing

A typical architecture:

```text
                    Users
                      │
                      ▼
               Global DNS / Routing
                      │
          ┌───────────┼───────────┐
          ▼           ▼           ▼
       Region A    Region B    Region C
          │           │           │
          ▼           ▼           ▼
       App A        App B        App C
          │           │           │
          ▼           ▼           ▼
       DynamoDB     DynamoDB     DynamoDB
          ╲           │           ╱
           ╲          │          ╱
            └── Global Table ───┘
```

---

# 17. Route Users to the Nearest Healthy Region

You can use global routing mechanisms to direct users toward an appropriate Region.

The logic is:

```text
User
 ↓
Routing layer
 ↓
Healthy/appropriate Region
 ↓
Application
 ↓
Local DynamoDB replica
```

If Region A becomes unhealthy:

```text
Region A ❌
     ↓
Routing changes
     ↓
Region B ✅
```

---

# 18. Active-Active Architecture

In active-active:

```text
Region A → ACTIVE
Region B → ACTIVE
```

Both serve traffic.

Advantages:

```text
High availability
Low global latency
Better resource utilization
Regional resilience
```

Challenges:

```text
Conflict handling
Operational complexity
Data consistency considerations
More expensive architecture
```

---

# 19. Active-Passive Architecture

In active-passive:

```text
Region A → ACTIVE
Region B → STANDBY
```

Normally:

```text
Users
 ↓
Region A
```

During failure:

```text
Region A ❌
 ↓
Failover
 ↓
Region B
```

This can simplify write ownership but may leave standby resources underutilized.

---

# 20. Active-Active vs Active-Passive

| Architecture   | Normal State                   | Failover                | Complexity |
| -------------- | ------------------------------ | ----------------------- | ---------- |
| Active-Active  | Multiple Regions serve traffic | Routing shifts          | Higher     |
| Active-Passive | One primary                    | Secondary takes traffic | Lower      |
| Single Region  | One Region                     | External recovery       | Lowest     |

The correct choice depends on:

```text
RTO
RPO
Latency
Budget
Business criticality
Conflict tolerance
```

---

# 21. RTO

RTO means:

> **Recovery Time Objective**

It answers:

> "How quickly must the application be restored?"

Example:

```text
RTO = 5 minutes
```

means:

```text
Major failure
     ↓
Application should recover within ~5 minutes
```

---

# 22. RPO

RPO means:

> **Recovery Point Objective**

It answers:

> "How much data loss can the business tolerate?"

Example:

```text
RPO = 1 minute
```

means the business can tolerate approximately one minute of potential data loss, depending on the recovery architecture.

---

# 23. RTO vs RPO

Remember:

```text
RTO = HOW FAST?
RPO = HOW MUCH DATA?
```

Example:

```text
RTO = 10 minutes
RPO = 1 minute
```

You need:

```text
Recovery within 10 minutes
+
Data loss limited to approximately 1 minute
```

---

# 24. Disaster Recovery Strategies

Common DR approaches include:

```text
Backup & Restore
Pilot Light
Warm Standby
Active-Passive
Active-Active
```

For DynamoDB, the appropriate strategy depends heavily on whether you use:

```text
Single-Region backups
Global Tables
Multi-Region application deployment
```

---

# 25. Backup & Restore Strategy

Basic architecture:

```text
Production DynamoDB
       │
       ▼
Backup / PITR
       │
       ▼
Restore
       │
       ▼
New DynamoDB Table
```

This is simpler than maintaining a fully active second Region.

---

# 26. Point-in-Time Recovery

PITR provides continuous backup capability.

Example:

```text
09:00 → Correct
10:00 → Correct
11:00 → Bad deployment
11:05 → Data corruption detected
```

You can restore to an appropriate earlier point.

---

# 27. Important PITR Concept

PITR doesn't mean:

> "Restore this table in place instantly."

The recovery process involves restoring data into a table.

A typical workflow is:

```text
Original Table
      │
      ▼
Restore
      │
      ▼
New Table
      │
      ▼
Validate
      │
      ▼
Application cutover
```

Plan the operational process ahead of time.

---

# 28. On-Demand Backup

Use on-demand backup when you need an explicit recovery point.

Example:

```text
Before major migration:

Create backup
     ↓
Perform migration
     ↓
Validate
```

If something goes wrong:

```text
Restore backup
```

---

# 29. Backup vs Global Tables

These solve different problems.

### Backup

```text
Recovery
Historical restoration
Accidental deletion
Corruption
```

### Global Tables

```text
Multi-Region availability
Low-latency regional access
Regional resilience
```

Don't treat Global Tables as a replacement for every backup/recovery requirement.

---

# 30. Global Tables Are Not a Backup

This is a critical production lesson.

If bad data is written:

```text
Region A:
Bad update
```

that change can replicate.

Therefore:

```text
Replication ≠ Backup
```

You still need appropriate backup/recovery capabilities.

---

# 31. Disaster Scenario — Accidental Delete

Suppose:

```text
Developer
   ↓
Deletes important records
```

Global Tables may replicate the deletion.

A backup/PITR strategy provides a recovery mechanism.

Typical response:

```text
Detect
 ↓
Stop further damage
 ↓
Identify recovery point
 ↓
Restore
 ↓
Validate
 ↓
Reconcile
 ↓
Resume service
```

---

# 32. Disaster Scenario — Regional Failure

Suppose:

```text
Region A ❌
```

With Global Tables:

```text
Region B ✅
Region C ✅
```

Application traffic can be redirected.

Architecture:

```text
Users
 ↓
Global Routing
 ↓
Region B
 ↓
DynamoDB Region B Replica
```

This is the primary value of multi-Region architecture.

---

# 33. Regional Failover Is More Than DynamoDB

A common mistake:

> "We enabled Global Tables, so DR is complete."

No.

Your entire application must support failover:

```text
DNS / Routing
Application
Compute
API
DynamoDB
Secrets
IAM
Monitoring
Queues
Event processing
External dependencies
```

A database can fail over while your application remains broken.

---

# 34. DR Dependency Mapping

Create a dependency map:

```text
User
 ↓
DNS
 ↓
API
 ↓
Compute
 ↓
DynamoDB
 ↓
SQS
 ↓
Lambda
 ↓
External Payment System
```

For each component ask:

```text
Is it multi-Region?
Can it fail over?
How long?
What data is lost?
How is state recovered?
```

---

# 35. Multi-Region Compute

A proper architecture might deploy:

```text
Region A
 ├── API
 ├── Compute
 └── DynamoDB

Region B
 ├── API
 ├── Compute
 └── DynamoDB
```

Both Regions should be capable of serving production traffic if your DR strategy requires it.

---

# 36. Multi-Region Secrets

If your application depends on:

```text
API credentials
Database-related secrets
Third-party credentials
Encryption configuration
```

those dependencies must also be available in the failover Region.

A common DR mistake:

```text
DynamoDB survives
Application survives
Secret dependency ❌
```

Result:

```text
Application still fails.
```

---

# 37. Multi-Region IAM

IAM is global, but your architecture still needs appropriate:

```text
Roles
Trust relationships
Permissions
KMS access
Resource policies
```

for workloads operating in each Region.

---

# 38. Multi-Region KMS

If customer-managed encryption keys are part of the architecture, carefully design key availability and permissions across Regions.

Your DR plan should explicitly answer:

```text
Can Region B decrypt/access what it needs?
Does Region B have the correct key configuration?
Are key policies correct?
```

---

# 39. Global Tables and Streams

Global Tables can be combined with DynamoDB Streams for event-driven architectures.

Example:

```text
Region A
DynamoDB
   ↓
Streams
   ↓
Lambda
```

But multi-Region event processing requires careful consideration of:

```text
Duplicate events
Ordering
Retries
Idempotency
Region ownership
```

---

# 40. Idempotency in Multi-Region Systems

Suppose an event is processed twice:

```text
Event E100
   ↓
Region A
   ↓
Process

Event E100
   ↓
Region B
   ↓
Process again
```

If processing isn't idempotent, you could:

```text
Charge twice
Create duplicate order
Send duplicate notification
```

Therefore:

> **Idempotency is essential in distributed multi-Region systems.**

---

# 41. Idempotency Key

Example:

```text
RequestId = REQ-10001
```

Store/process it so repeated requests don't perform the business operation twice.

Conceptually:

```text
Request
 ↓
Check IdempotencyKey
 ↓
Already processed?
 ├── Yes → Return previous result
 └── No  → Process
```

---

# 42. Failover and Idempotency

During failover:

```text
Region A
   ↓
Request starts

Region A fails

Region B
   ↓
Retry request
```

The same request might reach Region B.

Without idempotency:

```text
Duplicate operation
```

With idempotency:

```text
Same request
 ↓
Recognized
 ↓
Safe result
```

---

# 43. Global Table Write Ownership

One way to simplify failover:

```text
Normal:
Region A owns writes

Failure:
Region B becomes write owner
```

This is active-passive at the application/business level even if the DynamoDB infrastructure supports multi-Region replication.

This can reduce conflicts.

---

# 44. Active-Active Write Design

If both Regions can write:

```text
Region A → writes
Region B → writes
```

you need to answer:

```text
What happens when both modify the same item?
Which update wins?
Can updates be merged?
Can the operation be replayed?
```

If you can't answer these clearly, don't choose active-active multi-writer casually.

---

# 45. Data Partitioning by Region

A strong pattern:

```text
PK = REGION#US#CUSTOMER#100
PK = REGION#EU#CUSTOMER#200
```

or an equivalent ownership model.

Then:

```text
US entity
 ↓
US write owner

EU entity
 ↓
EU write owner
```

This can simplify global architectures.

---

# 46. Read-Local / Write-Local

Ideal global architecture:

```text
US User
 ↓
US App
 ↓
US DynamoDB

EU User
 ↓
EU App
 ↓
EU DynamoDB
```

Replication keeps data available globally.

This gives:

```text
Low latency
Regional resilience
Reduced cross-region writes
```

---

# 47. Read-Local / Write-Home

Another model:

```text
User
 ↓
Nearest Region
 ↓
Read locally

Write
 ↓
Home Region
```

This can be useful where:

```text
Reads need low latency
Writes require centralized ownership
```

---

# 48. Consistency Requirements

Before designing global architecture, classify each operation.

### Strongly consistency-sensitive

Examples:

```text
Financial balance
Inventory reservation
Seat booking
Order state
```

### Eventually consistent

Examples:

```text
Analytics counters
Recommendations
Activity feeds
Search indexes
```

Your architecture should reflect the business requirement.

---

# 49. Don't Make Everything Globally Synchronous

Trying to make every operation globally synchronous can increase:

```text
Latency
Complexity
Cost
Failure coupling
```

Instead:

```text
Critical state
    ↓
Strong/atomic local design

Non-critical propagation
    ↓
Asynchronous events
```

---

# 50. Disaster Recovery Testing

A DR plan that has never been tested is not a reliable DR plan.

Perform exercises such as:

```text
Region failure simulation
Application failover
DynamoDB restore
Backup validation
DNS/routing failover
Dependency failure
Credential failure
```

Measure:

```text
Actual RTO
Actual RPO
```

---

# 51. Game Day

A production "game day" can simulate:

```text
Region A unavailable
```

Then your team executes:

```text
1. Detect
2. Declare incident
3. Redirect traffic
4. Validate Region B
5. Verify DynamoDB
6. Verify downstream systems
7. Validate business transactions
8. Monitor
```

This exposes hidden dependencies.

---

# 52. Backup Restore Testing

Don't just enable backups.

Test:

```text
Create/identify recovery point
 ↓
Restore
 ↓
Validate schema/data
 ↓
Run application
 ↓
Measure restoration time
```

You need to know:

> **Can we actually restore when production is broken?**

---

# 53. Backup Retention Strategy

Think about:

```text
Operational recovery
Compliance
Business retention
Cost
Security
```

Don't keep everything forever without a reason.

Define:

```text
What?
How long?
Where?
Who can restore?
Who can delete?
```

---

# 54. Production Deployment Strategy

Avoid risky schema/data changes directly in production.

A safer approach:

```text
Development
   ↓
Testing
   ↓
Staging
   ↓
Production
```

For data migrations:

```text
Backup/PITR
   ↓
Deploy
   ↓
Migrate gradually
   ↓
Validate
```

---

# 55. Zero-Downtime Migration Concept

A common DynamoDB migration approach is:

```text
Old model
   ↓
Dual write
   ↓
New model
   ↓
Backfill
   ↓
Validate
   ↓
Switch reads
   ↓
Stop old writes
```

The exact implementation depends on the migration.

---

# 56. Dual Write Risks

Dual writes create consistency challenges.

Example:

```text
Write Old Table
    ↓
SUCCESS

Write New Table
    ↓
FAIL
```

Now:

```text
Old = updated
New = stale
```

Therefore dual-write migrations need:

```text
Retries
Idempotency
Reconciliation
Monitoring
Backfill
```

---

# 57. Advanced Production Pattern

A mature architecture might look like:

```text
                         GLOBAL USERS
                              │
                              ▼
                     Global Routing Layer
                              │
                ┌─────────────┴─────────────┐
                ▼                           ▼
           REGION A                    REGION B
                │                           │
          Application                 Application
                │                           │
                ▼                           ▼
          DynamoDB A  ◄──── Global ────► DynamoDB B
                │          Tables            │
                │                            │
                ▼                            ▼
             Streams                      Streams
                │                            │
                ▼                            ▼
             Lambda                       Lambda
                │                            │
                └───────────┬────────────────┘
                            ▼
                     Event Processing
                            │
                  ┌─────────┴─────────┐
                  ▼                   ▼
                 SQS                SNS
                  │
                  ▼
             Other Services

          Backup / PITR / Monitoring
                    │
                    ▼
                DR Strategy
```

---

# 58. Production Architecture Decision

Before selecting Global Tables, answer:

```text
1. Do we need multi-Region?
2. What is our RTO?
3. What is our RPO?
4. Do we need local writes?
5. Can multiple Regions write the same entity?
6. How are conflicts handled?
7. What dependencies need failover?
8. What is the backup strategy?
9. How do we test recovery?
10. What is the cost?
```

---

# 59. Global Tables — When to Use

Good candidates:

```text
Global user applications
Multi-Region SaaS
Low-latency global applications
Regional resilience requirements
Active-active workloads
```

Especially when users are geographically distributed.

---

# 60. When NOT to Use Global Tables

Don't introduce Global Tables merely because:

> "Multi-Region sounds better."

Avoid unnecessary complexity when:

```text
Single Region meets RTO/RPO
Users are geographically concentrated
No multi-Region requirement exists
Budget is constrained
Conflict handling would be difficult
```

Architecture should follow requirements.

---

# 61. Cost Considerations

Multi-Region architecture increases costs through factors such as:

```text
Additional regional resources
Cross-Region replication/data transfer
Additional application infrastructure
Monitoring
Backup
Operational overhead
```

Therefore compare:

```text
Business value
        vs
Additional cost
```

---

# 62. Security in Multi-Region

Every Region needs appropriate:

```text
IAM
Encryption
KMS
Network controls
CloudTrail
Monitoring
Backup
```

A common mistake:

```text
Region A secure
Region B forgotten
```

Your DR Region must meet the same security standards.

---

# 63. Compliance and Data Residency

Global replication may have regulatory implications.

Before replicating data globally, ask:

```text
Can this data legally leave the originating Region/country?
Are there residency requirements?
Are there contractual restrictions?
Which users' data can be replicated?
```

Global Tables should be evaluated against compliance requirements.

---

# 64. Production Failure Matrix

Build a matrix like:

| Failure            | Detection              | Recovery                |
| ------------------ | ---------------------- | ----------------------- |
| Item corruption    | Monitoring/application | PITR/restore            |
| Accidental delete  | CloudTrail/monitoring  | Restore                 |
| Region outage      | Health checks          | Route to healthy Region |
| Application outage | Monitoring             | Fail over compute       |
| Bad deployment     | Monitoring             | Rollback                |
| Dependency outage  | Application monitoring | Alternate dependency    |
| Replication issue  | Metrics/alerts         | Investigate/reconcile   |

This turns DR into an operational process.

---

# 65. RTO/RPO Example

Suppose business requirements are:

```text
RTO = 5 minutes
RPO = 1 minute
```

A backup-only architecture may not meet those targets.

You might require:

```text
Multi-Region application
+
Global Tables
+
Automated routing
+
Pre-deployed compute
+
Tested failover
```

The architecture must be validated against the actual targets.

---

# 66. DR Is a Business Requirement

Don't start with:

> "Should I use Global Tables?"

Start with:

> "What happens if this Region disappears?"

Then determine:

```text
Business impact
 ↓
RTO/RPO
 ↓
Architecture
 ↓
Technology choice
```

---

# 67. Common Mistake #1

### "Global Tables = Backup"

Wrong.

Global Tables provide:

```text
Replication
Multi-Region availability
```

Backup provides:

```text
Historical recovery
```

You generally need both capabilities for strong production protection.

---

# 68. Common Mistake #2

### "Multi-Region automatically means zero data loss."

Wrong.

Replication and distributed-system timing must be considered.

Define:

```text
RPO
Consistency requirements
Failure behavior
```

and test them.

---

# 69. Common Mistake #3

### "Only DynamoDB needs to be multi-Region."

Wrong.

If:

```text
DynamoDB → Region B
```

but:

```text
Application → Region A only
```

your application can still fail.

DR must cover the whole dependency chain.

---

# 70. Common Mistake #4

### "Active-active is always better."

Wrong.

Active-active introduces:

```text
Conflict resolution
Routing complexity
Operational complexity
Higher cost
```

Use it when the business benefits justify it.

---

# 71. Common Mistake #5

### "We enabled PITR, so DR testing isn't necessary."

Wrong.

You must test:

```text
Restore
Validation
Application cutover
Recovery timing
```

---

# 72. Common Mistake #6

### "Both Regions can freely update the same item."

Technically possible in appropriate Global Tables designs, but dangerous from a business-consistency perspective.

Prefer:

```text
Single writer per entity
```

when practical.

---

# 73. Common Mistake #7

### "Failover only means changing DNS."

Not enough.

Validate:

```text
Application
DynamoDB
IAM
KMS
Secrets
Queues
Streams
External dependencies
```

---

# 74. Common Mistake #8

### "Replication will fix bad data."

Replication may distribute bad data just as it distributes good data.

Therefore:

```text
Replication
+
Backup
+
Validation
```

are separate controls.

---

# 75. Top 50 Interview Questions — Lesson 9

### 1. What are DynamoDB Global Tables?

A managed multi-Region replication capability for DynamoDB.

### 2. Why use Global Tables?

For multi-Region availability, low-latency regional access, and disaster resilience.

### 3. Are Global Tables active-active?

They support multi-Region, multi-active architectures, but application-level write ownership and conflict strategy still need careful design.

### 4. Are Global Tables a backup?

No.

### 5. Why not?

Replication can replicate unwanted changes as well as desired ones.

### 6. What is RTO?

Recovery Time Objective — how quickly service must be restored.

### 7. What is RPO?

Recovery Point Objective — how much data loss is acceptable.

### 8. RTO vs RPO?

RTO = how fast; RPO = how much data.

### 9. What is active-active?

Multiple Regions actively serve production traffic.

### 10. What is active-passive?

One Region primarily serves traffic while another is prepared for failover.

### 11. Which is simpler?

Active-passive is generally simpler.

### 12. Which provides local global traffic?

Active-active generally does.

### 13. What is a Global Table replica?

A regional copy participating in the Global Table.

### 14. Can applications write in multiple Regions?

Yes, but multi-Region write conflicts must be understood and designed for.

### 15. How are conflicting writes handled?

DynamoDB Global Tables has conflict-resolution semantics based on the latest write for conflicting item updates.

### 16. How can you reduce conflicts?

Use single-writer-per-entity or Region ownership.

### 17. What is single-writer-per-entity?

Only one Region is responsible for writes to a given logical entity.

### 18. Why is it useful?

It reduces concurrent multi-Region write conflicts.

### 19. What is field-level ownership?

Different Regions own different attributes of an entity.

### 20. What is read-local/write-local?

Users read and write primarily against their nearest Region.

### 21. What is read-local/write-home?

Reads happen locally while writes are routed to the entity's home Region.

### 22. What is PITR?

Point-in-Time Recovery.

### 23. Why use PITR?

To recover from accidental corruption/deletion or other unwanted changes.

### 24. What is an on-demand backup?

An explicitly created DynamoDB backup.

### 25. Can PITR replace Global Tables?

No.

### 26. Can Global Tables replace backups?

No.

### 27. What happens during a regional failure?

Traffic can be redirected to a healthy Region when the application and data architecture support that failover.

### 28. Does DynamoDB failover automatically mean the whole application fails over?

No.

### 29. What else must fail over?

Compute, routing, secrets, dependencies, queues, event processing, and other application components.

### 30. Why is idempotency important during failover?

Requests may be retried and could otherwise execute twice.

### 31. What is an idempotency key?

A unique request identifier used to safely recognize repeated processing attempts.

### 32. Why test disaster recovery?

To verify actual RTO/RPO and discover hidden dependencies.

### 33. What is a DR game day?

A controlled exercise that simulates failures and tests recovery procedures.

### 34. Why test backup restoration?

Because having a backup does not prove the organization can successfully restore and use it.

### 35. What is a backup restore workflow?

Restore → validate → reconcile → cut over.

### 36. Why does data modeling matter in Global Tables?

It can help establish ownership boundaries and reduce conflicts.

### 37. What is a hot conflict?

Multiple Regions repeatedly updating the same logical item.

### 38. Why can active-active be difficult?

Concurrent writes and conflict resolution become more complicated.

### 39. Can Global Tables improve latency?

Yes, by allowing applications to access local regional replicas.

### 40. Does local access eliminate all global consistency concerns?

No.

### 41. Should all data be globally replicated?

Not necessarily. Compliance, residency, cost, and business requirements must be considered.

### 42. What is data residency?

Requirements controlling where certain data may be stored or processed.

### 43. Why is multi-Region KMS planning important?

Encryption keys and permissions must support the recovery architecture.

### 44. What is the biggest DR mistake?

Planning only the database failover and ignoring the rest of the application.

### 45. What is the biggest Global Tables mistake?

Allowing unrestricted multi-Region writes without a conflict strategy.

### 46. What should determine your DR architecture?

RTO, RPO, availability, latency, compliance, complexity, and cost.

### 47. What is the simplest DR option?

Backup/restore is generally simpler than maintaining a fully active multi-Region architecture.

### 48. When is Global Tables appropriate?

When the application genuinely requires multi-Region availability, local access, or regional resilience.

### 49. What should happen after a restore?

Validate data and application behavior before production cutover.

### 50. What is the most important Lesson 9 principle?

> **Replication gives availability; backups give recovery; application architecture makes regional failover actually work.**

---

# 🔥 Lesson 9 Golden Rules

1. **Global Tables = multi-Region replication and availability.**
2. **Global Tables ≠ backup.**
3. **PITR = operational recovery capability.**
4. **On-demand backup = explicit recovery point.**
5. **RTO = how fast you recover.**
6. **RPO = how much data you can lose.**
7. **Active-active provides powerful availability but increases complexity.**
8. **Active-passive can simplify write ownership and failover.**
9. **Prefer single-writer-per-entity when possible.**
10. **Design explicitly for multi-Region write conflicts.**
11. **Use idempotency for retry-safe operations.**
12. **Don't assume database failover means application failover.**
13. **Replicate only data that should be replicated.**
14. **Consider data residency before global replication.**
15. **Test backup restoration.**
16. **Test regional failover.**
17. **Measure actual RTO/RPO rather than assuming them.**
18. **Keep security controls consistent across Regions.**
19. **Design compute, routing, secrets, queues, and dependencies for DR.**
20. **Choose Global Tables because the business requires multi-Region capability—not simply because it is available.**

---

# 🧠 One-Line Memory Trick

> **Global Tables = Where → Backup/PITR = Recover → RTO = How Fast → RPO = How Much → Ownership = Who Writes → Idempotency = Safe Retry → DR Testing = Prove It Works.**

## 🎯 Lesson 9 Production Checklist

```text
☑ Define RTO
☑ Define RPO
☑ Decide single vs multi-Region
☑ Decide active-active vs active-passive
☑ Configure Global Tables if required
☑ Define write ownership
☑ Design conflict handling
☑ Enable PITR
☑ Establish on-demand backup strategy
☑ Test restore
☑ Deploy application in recovery Region
☑ Validate IAM
☑ Validate KMS
☑ Validate secrets
☑ Validate queues/events
☑ Configure global routing
☑ Implement idempotency
☑ Monitor replication/health
☑ Perform DR game days
☑ Measure actual RTO/RPO
☑ Document failover/failback procedures
```

### 🔥 Final Architecture to Remember

```text
                         GLOBAL USERS
                              │
                              ▼
                     Global Routing
                              │
             ┌────────────────┴────────────────┐
             ▼                                 ▼
         REGION A                           REGION B
             │                                 │
        Application                       Application
             │                                 │
             ▼                                 ▼
       DynamoDB A  ◄──── Global Tables ───► DynamoDB B
             │                                 │
             ▼                                 ▼
          Streams                           Streams
             │                                 │
             └──────────────┬──────────────────┘
                            ▼
                    Event Processing

                 ┌───────────────────┐
                 │ Backup / PITR      │
                 │ DR Procedures      │
                 │ Monitoring         │
                 │ Security           │
                 └───────────────────┘
```

**Next: Lesson 10 — DynamoDB Production Architecture, Cost Optimization, Observability, Performance Tuning, Migration & Real-World End-to-End Scenarios.**
