# ⚡ Amazon DynamoDB — Lesson 8

## Security, IAM, Encryption, VPC Endpoints, Fine-Grained Access & DynamoDB Protection

This lesson focuses strictly on **securing DynamoDB in production**.

The core security model is:

```text
Application
     │
     ▼
IAM Authentication / Authorization
     │
     ▼
DynamoDB
     │
 ┌───┼───────────────┐
 ▼   ▼               ▼
TLS Encryption   Encryption at Rest   Resource Policies
     │
     ▼
VPC Endpoint / Private Connectivity
     │
     ▼
Logging + Monitoring + Protection
```

---

# 1. DynamoDB Security — Big Picture

DynamoDB security has several layers:

```text
1. Identity
2. Authorization
3. Encryption
4. Network access
5. Data-level permissions
6. Monitoring
7. Protection against accidental/destructive actions
```

Think:

> **Who are you? → What can you do? → How is data protected? → Where can requests come from? → How do we detect misuse?**

---

# 2. Shared Responsibility Model

AWS secures the underlying infrastructure.

You are responsible for properly configuring:

```text
IAM permissions
Encryption configuration
Application credentials
Resource policies
Network access
Data protection
Logging
Backup/recovery
```

A secure DynamoDB deployment therefore isn't just:

> "AWS encrypts DynamoDB."

You still need correct authorization and operational controls.

---

# 3. IAM — The Primary Authorization Mechanism

DynamoDB integrates deeply with:

> **AWS Identity and Access Management (IAM)**

IAM determines whether a principal can perform actions such as:

```text
GetItem
PutItem
UpdateItem
DeleteItem
Query
Scan
BatchGetItem
BatchWriteItem
TransactWriteItems
```

---

# 4. IAM Users vs Roles

For production applications, prefer:

```text
Application
    ↓
IAM Role
    ↓
DynamoDB
```

rather than embedding long-term AWS access keys inside application code.

For AWS workloads such as:

```text
EC2
Lambda
ECS
EKS
```

use appropriate IAM roles/workload identities.

---

# 5. Least Privilege

One of the most important security principles:

> **Grant only the permissions the application actually needs.**

Bad:

```text
Action:
dynamodb:*
```

on every table.

Better:

```text
Allow:
dynamodb:GetItem
dynamodb:PutItem
dynamodb:UpdateItem

Resource:
specific table
```

And if possible, restrict access further.

---

# 6. Example — Read-Only Application

Suppose an application only needs to read products.

Don't give it:

```text
PutItem
UpdateItem
DeleteItem
```

Give only:

```text
GetItem
Query
```

and perhaps:

```text
BatchGetItem
```

if required.

This reduces the blast radius if the application is compromised.

---

# 7. IAM Actions

Common DynamoDB IAM actions include:

```text
dynamodb:GetItem
dynamodb:PutItem
dynamodb:UpdateItem
dynamodb:DeleteItem

dynamodb:Query
dynamodb:Scan

dynamodb:BatchGetItem
dynamodb:BatchWriteItem

dynamodb:TransactGetItems
dynamodb:TransactWriteItems
```

Administrative actions include operations such as:

```text
CreateTable
DeleteTable
UpdateTable
DescribeTable
```

These should generally be restricted to infrastructure/admin roles.

---

# 8. Separate Application and Administrative Permissions

A strong production model is:

```text
                 IAM
                  │
        ┌─────────┴─────────┐
        ▼                   ▼
 Application Role       Admin/Deployment Role
        │                   │
        ▼                   ▼
 Data operations       Table configuration
```

Application:

```text
Get
Put
Update
Query
```

Deployment/admin:

```text
CreateTable
UpdateTable
DeleteTable
UpdateContinuousBackups
etc.
```

This separation prevents applications from accidentally modifying infrastructure.

---

# 9. IAM Resource-Level Permissions

You can restrict permissions to specific DynamoDB resources.

Conceptually:

```text
Application Role
       ↓
Allow GetItem
       ↓
Table: Products
```

Instead of:

```text
All DynamoDB tables
```

This is a major least-privilege technique.

---

# 10. Table-Level Permissions

Suppose your account contains:

```text
Customers
Orders
Payments
Employees
```

A customer-facing application should not automatically access:

```text
Employees
Payments
```

You can restrict its IAM policy to only the tables it needs.

Example:

```text
Customer API
    ↓
Orders table
    ↓
GetItem / Query / UpdateItem
```

---

# 11. IAM Conditions

IAM policies can also include conditions.

This enables more sophisticated authorization.

Examples can involve:

```text
Source VPC endpoint
Principal tags
Resource tags
Request context
Leading keys
```

depending on the authorization requirement and supported DynamoDB/IAM condition keys.

---

# 12. Fine-Grained Access Control

Sometimes table-level authorization isn't enough.

Requirement:

> User A should only access User A's records.

Instead of creating:

```text
One table per user
```

you can use IAM policy conditions to restrict access based on key values in appropriate designs.

A common concept is:

> **LeadingKeys**

---

# 13. LeadingKeys Concept

Suppose your partition key is:

```text
UserId
```

Data:

```text
UserId = U100
UserId = U200
UserId = U300
```

You may design authorization so a principal can access only a particular leading key.

Conceptually:

```text
User U100
   ↓
Allowed:
UserId = U100

Denied:
UserId = U200
UserId = U300
```

This is a powerful fine-grained authorization pattern.

---

# 14. Why Data Modeling Matters for Security

Fine-grained access is easier when your keys align with authorization boundaries.

For example:

```text
PK = USER#U100
```

can make ownership boundaries explicit.

Then authorization can be designed around:

```text
USER#U100
```

This demonstrates an important DynamoDB principle:

> **Data modeling and security modeling are closely connected.**

---

# 15. Encryption at Rest

DynamoDB encrypts data at rest.

This protects stored data such as:

```text
Table data
Indexes
Backups
Streams
```

The encryption helps protect data stored on AWS infrastructure.

---

# 16. AWS-Owned Keys vs Customer Managed Keys

DynamoDB supports encryption using:

```text
AWS owned encryption keys
```

and customer-managed AWS KMS keys for supported configurations.

Customer-managed keys provide additional control over:

```text
Key policies
Permissions
Auditing
Key lifecycle
```

---

# 17. AWS KMS

When using a customer-managed KMS key, DynamoDB integrates with:

> **AWS Key Management Service (KMS)**

Conceptually:

```text
DynamoDB
   │
   ▼
AWS KMS
   │
   ▼
Customer-managed KMS key
```

This can be useful when organizational security requirements require customer control over encryption keys.

---

# 18. Why Use a Customer-Managed KMS Key?

Common reasons:

```text
Regulatory requirements
Centralized key management
Key policy control
Auditing
Separation of duties
Organizational encryption standards
```

But don't assume customer-managed keys are automatically "more secure" for every workload.

They introduce additional key-management responsibilities.

---

# 19. KMS Key Policy

If using a customer-managed KMS key, the key policy and IAM permissions must allow the appropriate services/principals to use the key.

A common mistake is:

```text
DynamoDB configured for KMS
       ↓
KMS permissions incorrect
       ↓
Operation fails
```

Therefore encryption configuration must be tested carefully.

---

# 20. Encryption in Transit

DynamoDB API communication should use:

> **TLS/HTTPS**

Conceptually:

```text
Application
    │
    │ encrypted TLS connection
    ▼
DynamoDB
```

This protects data while it travels between the client and DynamoDB service endpoints.

---

# 21. Don't Disable TLS

Production applications should use AWS SDKs and standard HTTPS endpoints.

Avoid custom insecure transport implementations.

The normal AWS SDK communication model already supports secure TLS connections.

---

# 22. VPC and DynamoDB

A common misconception:

> "DynamoDB runs inside my VPC."

It doesn't.

DynamoDB is an AWS managed service.

Your VPC communicates with DynamoDB through AWS networking mechanisms.

One important option is:

> **VPC Gateway Endpoint for DynamoDB**

---

# 23. DynamoDB Gateway Endpoint

A DynamoDB VPC gateway endpoint allows resources in a VPC to access DynamoDB without requiring internet/NAT connectivity for that path.

Conceptually:

```text
Private Subnet
      │
      ▼
VPC Route Table
      │
      ▼
DynamoDB Gateway Endpoint
      │
      ▼
DynamoDB
```

This is an important production networking pattern.

---

# 24. Why Use a VPC Endpoint?

Benefits include:

```text
No NAT dependency for DynamoDB traffic
Private connectivity path
Better network control
Potential NAT cost reduction
Security boundary through endpoint policies
```

---

# 25. Gateway Endpoint vs Interface Endpoint

For DynamoDB, the relevant VPC endpoint type is:

> **Gateway endpoint**

This differs from interface endpoints, which use AWS PrivateLink and elastic network interfaces.

Remember:

```text
DynamoDB
   ↓
Gateway Endpoint
```

---

# 26. Endpoint Policy

A VPC endpoint can have a policy controlling which resources/actions can be accessed through the endpoint.

Conceptually:

```text
VPC
 ↓
Endpoint Policy
 ↓
Allowed DynamoDB resources
```

This adds another authorization layer.

---

# 27. IAM + Endpoint Policy

Security can be layered:

```text
Application
     ↓
IAM policy
     ↓
VPC endpoint policy
     ↓
DynamoDB resource policy
     ↓
DynamoDB
```

A request needs to satisfy the applicable authorization controls.

This is defense in depth.

---

# 28. DynamoDB Resource-Based Policies

DynamoDB supports resource-based policies for supported resources/features.

These can help control access at the resource level and support cross-account access patterns.

Conceptually:

```text
Account A
Application
     │
     ▼
DynamoDB Table
     ▲
     │
Resource Policy
     │
Account B
```

This can be useful for controlled cross-account architectures.

---

# 29. Cross-Account Access

Suppose:

```text
Account A
Application

Account B
DynamoDB
```

You need an explicit authorization design.

Possible approaches include:

```text
IAM roles
Resource-based policies
Cross-account trust
```

depending on the architecture.

Never assume:

> "If the network can reach it, the application can access it."

Network connectivity and authorization are separate concerns.

---

# 30. Authentication vs Authorization

Important distinction:

### Authentication

> **Who are you?**

### Authorization

> **What are you allowed to do?**

In AWS:

```text
IAM
```

primarily handles identity and authorization.

For DynamoDB API access, AWS request signing and IAM-based authorization establish who is making the request and what actions are permitted.

---

# 31. Temporary Credentials

For workloads running on AWS, prefer temporary credentials obtained through roles rather than long-lived access keys.

Example:

```text
Lambda
  ↓
Execution Role
  ↓
Temporary Credentials
  ↓
DynamoDB
```

This is safer than:

```text
Hard-coded AWS_ACCESS_KEY_ID
Hard-coded AWS_SECRET_ACCESS_KEY
```

---

# 32. Never Hard-Code Credentials

Bad:

```text id="ndgl7p"
const accessKey = "AKIA...";
const secretKey = "...";
```

Also don't put AWS credentials in:

```text
Git repositories
Docker images
Source code
Frontend JavaScript
```

Use IAM roles and appropriate secret/credential management mechanisms.

---

# 33. Browser Access to DynamoDB

Avoid giving a browser direct broad DynamoDB permissions.

Bad:

```text
Browser
   ↓
Powerful DynamoDB IAM permissions
```

Better:

```text
Browser
   ↓
API
   ↓
Authorization
   ↓
DynamoDB
```

or use carefully designed temporary credentials/fine-grained access when there is a specific justified architecture.

---

# 34. Application-Level Authorization

IAM doesn't replace application authorization.

Example:

```text
User A
   ↓
API
   ↓
"Can this user modify Order O100?"
```

Your application may need to verify:

```text
User owns order
OR
User has admin role
```

Then DynamoDB is accessed using an appropriately restricted service role.

---

# 35. IAM Is Not Your Business Authorization System

Suppose your application has:

```text
Customer
Manager
Admin
Auditor
```

You shouldn't necessarily create a new IAM principal for every customer.

Instead:

```text
Application identity
      ↓
Business authorization
      ↓
DynamoDB
```

IAM controls AWS resource access.

Application authorization controls business permissions.

---

# 36. Fine-Grained Access Example

Suppose:

```text
PK = CUSTOMER#100
```

Customer 100 requests:

```text
CUSTOMER#100
```

but tries:

```text
CUSTOMER#200
```

A secure design should ensure:

```text
Customer 100
     ↓
Allowed: CUSTOMER#100
Denied: CUSTOMER#200
```

This can involve:

```text
Application authorization
+
IAM conditions
+
DynamoDB key design
```

---

# 37. Protection Against Accidental Deletes

One major production risk is:

```text
DeleteTable
```

A developer accidentally executes:

```text
aws dynamodb delete-table ...
```

You should restrict administrative permissions.

Separate:

```text
Developer
Application
Deployment
Production Admin
```

roles.

---

# 38. Point-in-Time Recovery

Security isn't just confidentiality.

It also includes:

> **Recoverability**

DynamoDB supports:

> **Point-in-Time Recovery (PITR)**

This provides continuous backups for supported tables and allows restoration to a point in time within the supported recovery window.

---

# 39. Why PITR Matters

Imagine:

```text
10:00 AM → Data correct
10:30 AM → Bad deployment
10:31 AM → Many records corrupted
```

Without recovery:

```text
Data damaged
```

With PITR:

```text
Restore table
to appropriate earlier point
```

This can dramatically reduce recovery time.

---

# 40. On-Demand Backups

DynamoDB also supports:

> **On-demand backups**

Useful when you want explicit backup points for situations such as:

```text
Migration
Major deployment
Compliance
Archival
Testing
Operational safety
```

---

# 41. PITR vs On-Demand Backup

### PITR

```text
Continuous backup capability
```

Useful for:

```text
Operational recovery
Accidental corruption
Recovery to a selected point in time
```

### On-demand backup

```text
Explicit backup
```

Useful for:

```text
Migration
Long-term retention strategy
Major change points
```

---

# 42. Global Tables and Security

If using DynamoDB Global Tables:

```text
Region A
   ↕
Region B
   ↕
Region C
```

security considerations apply across all participating regions.

You need to think about:

```text
IAM
Encryption
KMS
Data residency
Resource policies
Backup/recovery
Cross-region operations
```

---

# 43. Encryption and Global Tables

If customer-managed KMS keys are involved, carefully plan key configuration across participating regions.

Don't assume:

```text
One regional KMS key
```

automatically covers every regional resource configuration.

Encryption architecture should be designed together with your multi-region strategy.

---

# 44. CloudTrail

For API-level auditing, use:

> **AWS CloudTrail**

CloudTrail can record AWS API activity such as:

```text
Who performed an action?
What API was called?
When?
From where?
What AWS resource was involved?
```

This is essential for security investigations and governance.

---

# 45. CloudTrail vs DynamoDB Streams

Don't confuse them.

### DynamoDB Streams

Captures:

```text
DynamoDB item changes
```

### CloudTrail

Captures:

```text
AWS API activity
```

Example:

```text
Application:
PutItem
     ↓
DynamoDB Stream:
Item changed

CloudTrail:
Who called PutItem?
```

They answer different questions.

---

# 46. CloudWatch Monitoring

Use CloudWatch for operational monitoring.

Examples:

```text
ThrottledRequests
ConsumedReadCapacityUnits
ConsumedWriteCapacityUnits
SystemErrors
UserErrors
SuccessfulRequestLatency
```

Exact metrics and dimensions depend on the DynamoDB feature/workload.

---

# 47. Security Monitoring Architecture

A production setup can look like:

```text
                 DynamoDB
                    │
        ┌───────────┼────────────┐
        ▼           ▼            ▼
   CloudTrail   CloudWatch    Streams
        │           │            │
        ▼           ▼            ▼
   Audit/SIEM    Alerts       Lambda
```

Each provides a different type of visibility.

---

# 48. AWS Config / Security Governance

Organizations may use AWS governance services to detect configuration drift or enforce security standards.

For example:

```text
Required backup configuration
Required encryption settings
Required tagging
```

The exact governance implementation depends on your organization.

---

# 49. DynamoDB Access Logging

A mature security program should answer:

```text
Who accessed the data?
Who modified it?
Which API was used?
Was the request allowed?
Was the request denied?
Where did it originate?
```

Combine:

```text
CloudTrail
IAM
CloudWatch
Application logs
```

as appropriate.

---

# 50. Deny by Default

A strong security philosophy:

```text
No permission
     ↓
Explicitly grant required permission
```

Avoid:

```text
Allow everything
     ↓
Try to restrict later
```

Least privilege should be designed from the beginning.

---

# 51. IAM Policy Example — Conceptual

Suppose an application only needs:

```text
GetItem
Query
UpdateItem
```

on:

```text
Orders
```

Conceptually:

```text
ApplicationRole
    │
    ├── GetItem
    ├── Query
    └── UpdateItem
          │
          ▼
       Orders
```

It doesn't receive:

```text
DeleteTable
CreateTable
Scan every table
Access Payments
```

unless genuinely required.

---

# 52. Protecting Sensitive Data

DynamoDB may contain:

```text
Customer information
Payment metadata
Authentication state
Business secrets
Personal information
```

Encryption is only one control.

Also consider:

```text
Least privilege
Data minimization
Access logging
Application authorization
Backup protection
Key management
Data retention
```

---

# 53. Don't Store Secrets Carelessly

Avoid storing:

```text
API keys
Passwords
Private credentials
Long-lived secrets
```

in DynamoDB simply because it is encrypted.

Use appropriate secret-management services such as:

```text
AWS Secrets Manager
AWS Systems Manager Parameter Store
```

when appropriate.

Encryption at rest does not make poor secret-management practices safe.

---

# 54. DynamoDB Injection / Expression Safety

Applications dynamically construct:

```text
KeyConditionExpression
FilterExpression
UpdateExpression
```

Be careful with user-controlled values.

Use AWS SDK parameterization mechanisms rather than blindly concatenating untrusted strings into expressions.

Conceptually:

```text
User Input
    ↓
Validated parameter
    ↓
SDK expression values
    ↓
DynamoDB
```

---

# 55. Scan Security Consideration

A `Scan` reads every item that matches the operation scope.

Even if IAM allows it, unrestricted scanning can create:

```text
High read consumption
Performance problems
Large data exposure
```

Prefer:

```text
Query
```

when the access pattern allows it.

---

# 56. Security and Query Design

Suppose users should access only:

```text
PK = USER#123
```

A well-designed key model allows targeted queries.

Bad:

```text
Scan entire table
Filter UserId = 123
```

Better:

```text
Query:
PK = USER#123
```

Security and performance can reinforce each other.

---

# 57. VPC Endpoint Security Architecture

A production private application might look like:

```text
Private EC2/ECS/Lambda Networking
          │
          ▼
     VPC Route Table
          │
          ▼
DynamoDB Gateway Endpoint
          │
          ▼
       DynamoDB
```

Then layer:

```text
IAM
+
Endpoint Policy
+
DynamoDB Resource Policy
```

where applicable.

---

# 58. Important: VPC Endpoint ≠ IAM

A VPC endpoint controls the network path.

IAM controls authorization.

So:

```text
Network access
      ≠
Data permission
```

Both need to be designed.

---

# 59. Real-Time Scenario — Production API

Requirement:

> Customer API runs in private subnets and accesses DynamoDB.

Architecture:

```text
Internet
   ↓
Load Balancer/API Gateway
   ↓
Application
   ↓
Private VPC
   ↓
DynamoDB Gateway Endpoint
   ↓
DynamoDB
```

Security:

```text
IAM Role
+
Endpoint Policy
+
TLS
+
Encryption at Rest
+
CloudTrail
+
CloudWatch
```

---

# 60. Real-Time Scenario — Multi-Tenant SaaS

Suppose:

```text
Tenant A
Tenant B
Tenant C
```

share one DynamoDB table.

Partition key:

```text
TENANT#<tenantId>
```

Example:

```text
TENANT#A
TENANT#B
TENANT#C
```

Authorization should ensure:

```text
Tenant A
   ↓
Only Tenant A data
```

This requires careful application and IAM design.

---

# 61. Real-Time Scenario — Employee/Admin Separation

Production roles:

```text
Application Role
   ↓
Normal CRUD

Developer Role
   ↓
Limited development access

Deployment Role
   ↓
Infrastructure changes

Production Admin
   ↓
Break-glass/admin capabilities
```

Don't give every developer:

```text
dynamodb:*
```

against production.

---

# 62. Real-Time Scenario — Sensitive Customer Data

Requirement:

> Protect customer information.

Controls:

```text
1. Encryption at rest
2. TLS in transit
3. Least-privilege IAM
4. Application authorization
5. CloudTrail auditing
6. Backup/recovery controls
7. Data retention policies
```

Security is layered.

---

# 63. Real-Time Scenario — Accidental Data Deletion

Requirement:

> Recover from accidental deletion.

Use:

```text
PITR
+
On-demand backups
+
Restricted DeleteItem/DeleteTable permissions
+
CloudTrail
```

If the issue is detected:

```text
Investigate
   ↓
Identify recovery point
   ↓
Restore
   ↓
Validate
   ↓
Cut over/reconcile
```

---

# 64. Real-Time Scenario — Cross-Account Access

Requirement:

```text
Account A application
        ↓
Account B DynamoDB
```

Design:

```text
IAM / resource-based authorization
+
explicit cross-account trust
+
least privilege
```

Don't solve cross-account security purely through networking.

---

# 65. Real-Time Scenario — Compromised Application

Suppose an application role is compromised.

If it has:

```text
dynamodb:*
```

across all tables:

```text
Blast radius = huge
```

If it only has:

```text
GetItem
UpdateItem
```

on:

```text
Orders
```

the blast radius is significantly reduced.

This is exactly why least privilege matters.

---

# 66. Security Architecture — Recommended Mental Model

Think in layers:

```text
                 USER
                   │
                   ▼
             Authentication
                   │
                   ▼
        Application Authorization
                   │
                   ▼
              IAM Role
                   │
                   ▼
            VPC Connectivity
                   │
                   ▼
           Endpoint Policy
                   │
                   ▼
        DynamoDB Resource Policy
                   │
                   ▼
              DynamoDB
             /        \
            ▼          ▼
      Encryption     Logging
        at Rest      / Audit
```

Not every workload requires every layer, but the model helps you identify security gaps.

---

# 67. Top 40 Interview Questions — Lesson 8

### 1. How does DynamoDB authorization work?

Primarily through AWS IAM, with resource policies and other controls where applicable.

### 2. What is least privilege?

Granting only the permissions required to perform the intended task.

### 3. Should applications use long-lived AWS access keys?

Prefer IAM roles and temporary credentials for AWS workloads.

### 4. Is DynamoDB encrypted at rest?

Yes.

### 5. Can DynamoDB use customer-managed KMS keys?

Yes, for supported encryption configurations.

### 6. What is KMS used for?

Encryption key management.

### 7. Is DynamoDB traffic encrypted in transit?

AWS SDK/API communication uses TLS.

### 8. Does DynamoDB run inside a customer's VPC?

No.

### 9. How can a VPC privately access DynamoDB?

Through a DynamoDB VPC gateway endpoint.

### 10. What type of endpoint is used for DynamoDB?

A gateway endpoint.

### 11. Does a VPC endpoint replace IAM?

No.

### 12. What is an endpoint policy?

A policy controlling access through a VPC endpoint.

### 13. What is fine-grained access control?

Restricting access to specific data/resources rather than granting broad table-level access.

### 14. What is `dynamodb:LeadingKeys` commonly used for?

Restricting access based on partition-key values in suitable authorization designs.

### 15. Why does key design affect security?

Keys can define tenant/user ownership boundaries.

### 16. What is a DynamoDB resource-based policy?

A policy attached to a supported DynamoDB resource to help control access.

### 17. Can DynamoDB support cross-account access?

Yes, with appropriate authorization design.

### 18. What is CloudTrail used for?

Auditing AWS API activity.

### 19. What are DynamoDB Streams used for?

Capturing item-level changes for downstream processing.

### 20. CloudTrail vs Streams?

CloudTrail = API activity.

Streams = item changes.

### 21. What is PITR?

Point-in-Time Recovery for recovering a DynamoDB table to a selected earlier point within the supported recovery window.

### 22. Why enable PITR?

To protect against accidental deletion/corruption and improve operational recovery.

### 23. What are on-demand backups?

Explicit DynamoDB backups created for operational, migration, archival, or recovery purposes.

### 24. Is encryption alone enough for security?

No.

### 25. What else is required?

IAM, least privilege, application authorization, network controls, logging, backup/recovery, and operational governance.

### 26. Should a browser receive broad DynamoDB permissions?

Generally no.

### 27. Where should business authorization occur?

Typically in the application/service layer, with IAM enforcing AWS resource permissions.

### 28. Why avoid `dynamodb:*`?

It creates excessive permissions and a large blast radius.

### 29. Why prefer Query over Scan?

Query is targeted and generally much more efficient when the key design supports it.

### 30. Can Scan create security concerns?

Yes. It can expose/read large amounts of data and consume substantial capacity if broadly permitted.

### 31. What is a gateway endpoint?

A VPC routing mechanism that provides private access to supported AWS services such as DynamoDB.

### 32. Does a VPC endpoint make data automatically authorized?

No. Authorization still applies.

### 33. Why separate application and admin IAM roles?

To prevent applications from modifying infrastructure or performing unnecessary privileged operations.

### 34. How do you protect against accidental DeleteTable?

Restrict administrative permissions and separate production deployment/admin roles.

### 35. How do you protect multi-tenant DynamoDB data?

Use tenant-aware key design plus application authorization and appropriately scoped IAM controls.

### 36. Should secrets be stored directly in DynamoDB?

Not merely because DynamoDB is encrypted. Use appropriate secret-management solutions.

### 37. What is defense in depth?

Using multiple independent security controls rather than relying on one mechanism.

### 38. Can encryption protect against an authorized malicious application?

No. If the application is legitimately authorized to decrypt/access the data, encryption at rest doesn't replace authorization controls.

### 39. What is the biggest IAM mistake?

Granting broad permissions such as `dynamodb:*` when only a few actions/resources are required.

### 40. What is the most important DynamoDB security principle?

> **Least privilege + strong identity + encryption + network controls + auditing + recoverability.**

---

# 🔥 Lesson 8 Golden Rules

1. **Use IAM roles rather than hard-coded AWS credentials.**
2. **Apply least privilege to DynamoDB actions and resources.**
3. **Separate application permissions from administrative permissions.**
4. **Use TLS for data in transit.**
5. **DynamoDB encrypts data at rest.**
6. **Use customer-managed KMS keys when organizational requirements justify the additional control and management.**
7. **DynamoDB is a managed AWS service, not a resource deployed inside your VPC.**
8. **Use a DynamoDB Gateway VPC Endpoint for private VPC-to-DynamoDB connectivity.**
9. **Remember: network connectivity and authorization are different security layers.**
10. **Use fine-grained access patterns when users/tenants need restricted data access.**
11. **Design partition keys with authorization boundaries in mind.**
12. **Use CloudTrail for API auditing.**
13. **Use CloudWatch for operational monitoring and alerting.**
14. **Enable PITR for important production tables.**
15. **Use on-demand backups where explicit backup points are useful.**
16. **Don't give applications `dynamodb:*` unless there is an exceptional, justified requirement.**
17. **Don't expose broad DynamoDB permissions directly to browsers.**
18. **Don't assume encryption replaces authorization.**
19. **Don't use Scan when a properly designed Query can satisfy the access pattern.**
20. **Security should be designed together with DynamoDB data modeling.**

---

# 🧠 One-Line Memory Trick

> **IAM = Who can do what → KMS = How data is protected at rest → TLS = Protection in transit → VPC Endpoint = Private network path → Fine-Grained Access = Which records → CloudTrail = Who did what → PITR/Backup = Recover when something goes wrong.**

### Lesson 8 Production Checklist

```text
☑ IAM roles
☑ Least privilege
☑ Resource-level permissions
☑ Fine-grained authorization where needed
☑ TLS
☑ Encryption at rest
☑ KMS where required
☑ VPC Gateway Endpoint
☑ Endpoint policy
☑ Resource policies where applicable
☑ CloudTrail
☑ CloudWatch
☑ PITR
☑ On-demand backups
☑ Tenant-aware data model
☑ No hard-coded credentials
☑ Separate admin/application roles
```

**Next: Lesson 9 — DynamoDB Global Tables, Multi-Region Architecture, Disaster Recovery, Backup/Restore & Advanced Production Design.**
