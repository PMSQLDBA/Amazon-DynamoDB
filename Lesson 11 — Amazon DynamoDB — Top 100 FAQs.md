# Amazon DynamoDB — Top 100 FAQs

## 1. What is Amazon DynamoDB?

1. DynamoDB is a fully managed, serverless NoSQL database service from AWS.
2. It provides single-digit-millisecond performance at scale.
3. It supports key-value and document data models.
4. It automatically handles infrastructure, patching, and much of the operational management.
5. It can scale horizontally to very large workloads.
6. It is designed for highly available, low-latency applications.

---

## 2. When should I choose DynamoDB instead of a relational database?

1. Choose DynamoDB when predictable low-latency access at massive scale is important.
2. It is particularly useful when access patterns are known in advance.
3. It works well for high-volume transactional workloads.
4. It avoids traditional relational database scaling bottlenecks.
5. It is less suitable when applications require many arbitrary joins and ad-hoc SQL queries.
6. The decision should be based primarily on workload and access patterns.

---

## 3. What are the main components of DynamoDB?

1. Tables contain DynamoDB items.
2. Items contain attributes.
3. Partition keys determine how data is distributed.
4. Sort keys organize related items within a partition key.
5. GSIs and LSIs provide alternative access patterns.
6. Streams, backups, TTL, transactions, and Global Tables provide additional capabilities.

---

## 4. What is an item in DynamoDB?

1. An item is similar conceptually to a row in a relational database.
2. It is identified by its primary key.
3. An item can contain multiple attributes.
4. Different items in the same table can have different attributes.
5. Item size affects capacity consumption.
6. Keeping items reasonably small is important for performance and cost.

---

## 5. What is an attribute?

1. An attribute is a named piece of data within an item.
2. It is conceptually similar to a column value.
3. DynamoDB supports scalar, document, and set data types.
4. Items do not need to have identical attributes.
5. Attributes can be nested when using document types.
6. Attribute design should support the application's access patterns.

---

# Primary Keys & Data Modeling

## 6. What is a partition key?

1. A partition key uniquely identifies an item when used as a simple primary key.
2. DynamoDB uses it to determine the logical partitioning of data.
3. A good partition key distributes requests across the table.
4. High-cardinality values are generally preferable for high-scale workloads.
5. Poor key design can create hot partitions.
6. Partition-key design is one of the most important DynamoDB decisions.

---

## 7. What is a composite primary key?

1. A composite primary key consists of a partition key and sort key.
2. Multiple items can share the same partition-key value.
3. The sort key differentiates those items.
4. It enables efficient range and ordered queries.
5. It is useful for hierarchical and one-to-many relationships.
6. It is fundamental to many DynamoDB single-table designs.

---

## 8. What is a sort key?

1. A sort key organizes items sharing the same partition key.
2. Items can be queried using sort-key conditions.
3. It can represent timestamps, entity types, versions, or relationships.
4. It enables range queries.
5. It can support ascending or descending retrieval.
6. Good sort-key design can eliminate the need for additional queries or indexes.

---

## 9. What is single-table design?

1. Single-table design stores multiple logical entities in one physical table.
2. Items are distinguished using carefully designed keys.
3. It is based on access patterns rather than normalized entities.
4. It can reduce the need for joins.
5. It can provide highly efficient application queries.
6. It requires more up-front data-modeling discipline.

---

## 10. Why is DynamoDB data modeling different from SQL modeling?

1. Relational databases commonly start with entities and relationships.
2. DynamoDB commonly starts with access patterns.
3. DynamoDB doesn't provide traditional joins between arbitrary tables.
4. Data is often intentionally duplicated.
5. Denormalization is frequently beneficial.
6. The objective is efficient predictable access rather than normalization.

---

## 11. What makes a good partition key?

1. It should distribute traffic effectively.
2. It should generally have sufficient cardinality.
3. It should avoid concentrating most requests on one value.
4. It should support the application's most important queries.
5. It should remain useful as data volume grows.
6. Key design should be validated using realistic workload testing.

---

## 12. What is a hot partition?

1. A hot partition occurs when disproportionate traffic targets a particular partition.
2. A common cause is a poor partition-key design.
3. Low-cardinality or highly popular keys can cause this problem.
4. Hot partitions can result in throttling and latency.
5. Increasing overall capacity may not completely solve a hot-key problem.
6. Redesigning the access pattern or key distribution is often required.

---

## 13. How do you prevent hot partitions?

1. Use high-cardinality partition-key values.
2. Spread requests across many partition-key values.
3. Avoid concentrating traffic on a single popular key.
4. Consider sharding or bucketing for extreme workloads.
5. Analyze actual request distribution rather than only data distribution.
6. Load-test the design before production.

---

## 14. What is a composite sort key pattern?

1. A sort key can combine multiple pieces of information.
2. Example: `ORDER#2026-08-11#1001`.
3. This allows hierarchical organization.
4. Prefixes can represent entity types.
5. Timestamps can enable chronological queries.
6. Composite keys can support multiple related access patterns efficiently.

---

## 15. What is an access pattern?

1. An access pattern describes how an application retrieves or modifies data.
2. Examples include "get customer by ID."
3. Another example is "get a customer's latest orders."
4. Another is "find orders by status."
5. DynamoDB tables should be designed around these requirements.
6. Access patterns should be identified before finalizing keys and indexes.

---

# Read/Write Operations

## 16. What is GetItem?

1. GetItem retrieves a single item by its primary key.
2. It is one of the most direct DynamoDB operations.
3. It provides predictable access when the key is known.
4. Strongly consistent reads can be requested where appropriate.
5. Projection expressions can limit returned attributes.
6. It is generally preferable to scanning when the key is available.

---

## 17. What is PutItem?

1. PutItem creates or replaces an item.
2. The item is identified by its primary key.
3. Conditional expressions can prevent unwanted overwrites.
4. It can be used as an idempotent operation when designed correctly.
5. It consumes write capacity based partly on item size.
6. It is commonly used for inserts and complete item replacement.

---

## 18. What is UpdateItem?

1. UpdateItem modifies specific attributes of an item.
2. It can add, remove, or modify attributes.
3. It can use conditional expressions.
4. It avoids replacing the entire item when only a few attributes change.
5. Atomic counters can be implemented using updates.
6. It is commonly used for incremental state changes.

---

## 19. What is DeleteItem?

1. DeleteItem removes an item identified by its primary key.
2. Conditional expressions can prevent accidental deletion.
3. It consumes write capacity.
4. It can be used as part of transactional operations.
5. Streams can capture deletion events when enabled.
6. Applications should consider idempotency when retrying deletes.

---

## 20. What is Query?

1. Query retrieves items using a specific partition-key value.
2. It can optionally apply sort-key conditions.
3. It is generally much more targeted than Scan.
4. Results can be ordered by sort key.
5. Pagination is supported.
6. Good data modeling makes Query the primary read pattern.

---

## 21. What is Scan?

1. Scan examines items across a table or index.
2. It can be expensive for large datasets.
3. It is generally slower and less efficient than targeted Query operations.
4. Filters do not necessarily eliminate the underlying read work.
5. Repeated production scans should be carefully reviewed.
6. A frequent Scan requirement can indicate a poor data model.

---

## 22. Query vs Scan — which should I use?

1. Use Query when the partition key is known.
2. Use Scan when you genuinely need to examine a broad dataset.
3. Query is generally more efficient.
4. Scan can consume significant read capacity.
5. Filtering a Scan doesn't magically turn it into a targeted lookup.
6. For recurring business queries, consider redesigning keys or adding an index.

---

## 23. What is pagination in DynamoDB?

1. DynamoDB operations can return results in pages.
2. The response can contain a `LastEvaluatedKey`.
3. The application uses that key to retrieve the next page.
4. Pagination prevents huge responses from being returned at once.
5. Applications should correctly handle continuation tokens.
6. Pagination is essential for large result sets.

---

# Consistency

## 24. What is eventual consistency?

1. Eventually consistent reads may return slightly older data.
2. They provide lower-cost read behavior in applicable situations.
3. DynamoDB generally uses eventual consistency by default for many reads.
4. Changes become visible across replicas within a short period.
5. Many workloads don't require immediate read-after-write visibility.
6. Applications should choose consistency based on business requirements.

---

## 25. What is a strongly consistent read?

1. A strongly consistent read returns the most recent committed value available for the item.
2. It can be requested for supported read operations.
3. It can consume more read capacity than an eventually consistent read.
4. It is useful when stale data could cause business problems.
5. Not every DynamoDB feature supports strong consistency in every context.
6. Use it selectively rather than everywhere.

---

## 26. When should I use strong consistency?

1. Use it when immediate read-after-write correctness matters.
2. Inventory and critical state transitions may require it.
3. Financial workflows may have stricter consistency requirements.
4. Many dashboards and read-only views can use eventual consistency.
5. Evaluate business impact before enabling it broadly.
6. Strong consistency should solve a real requirement rather than being used by default.

---

# Capacity & Scaling

## 27. What is provisioned capacity?

1. You specify expected read and write capacity.
2. DynamoDB allocates capacity according to that configuration.
3. Auto scaling can adjust provisioned capacity.
4. It works well for predictable workloads.
5. Capacity planning becomes important.
6. It can provide cost advantages for stable workloads.

---

## 28. What is on-demand capacity?

1. On-demand mode automatically accommodates changing traffic without manual capacity planning.
2. It is useful for unpredictable workloads.
3. It is convenient for new applications.
4. It can handle significant traffic variation.
5. Cost depends on request usage and other applicable charges.
6. The choice should be based on workload characteristics rather than assumptions.

---

## 29. Provisioned vs On-Demand — which should I choose?

1. Use provisioned when workload patterns are predictable.
2. Use on-demand when traffic is unpredictable or highly variable.
3. Provisioned can be attractive for stable high-volume workloads.
4. On-demand reduces capacity-planning overhead.
5. Auto scaling can make provisioned capacity more adaptive.
6. Benchmark your real workload before making a long-term decision.

---

## 30. What is DynamoDB Auto Scaling?

1. Auto scaling adjusts provisioned capacity based on utilization.
2. It helps accommodate changing workloads.
3. It reduces manual capacity management.
4. Scaling isn't necessarily instantaneous.
5. Sudden spikes should be considered separately from gradual traffic growth.
6. Monitor scaling behavior rather than assuming it solves every capacity issue.

---

## 31. What causes throttling?

1. Traffic can exceed available capacity.
2. A hot partition can cause localized throttling.
3. A GSI can also experience capacity pressure.
4. Sudden traffic spikes can trigger throttling.
5. Retry storms can amplify the problem.
6. Investigate the workload and key distribution before simply increasing capacity.

---

## 32. How should an application handle throttling?

1. Implement retries.
2. Use exponential backoff.
3. Add jitter to avoid synchronized retries.
4. Keep retry limits reasonable.
5. Make operations idempotent where appropriate.
6. Investigate and correct the underlying capacity or key-design problem.

---

## 33. What is exponential backoff?

1. Retry delays increase after successive failures.
2. This reduces pressure on DynamoDB.
3. It prevents immediate repeated requests.
4. Jitter adds randomness to retry timing.
5. AWS SDKs provide retry mechanisms, but application behavior should still be understood.
6. Poor retry design can turn a small incident into a large outage.

---

# Indexes

## 34. What is a GSI?

1. GSI stands for Global Secondary Index.
2. It provides an alternative partition/sort-key structure.
3. It supports access patterns that the base table key cannot efficiently handle.
4. It can span the full table.
5. GSIs have their own capacity and storage implications.
6. They should be created only for meaningful access patterns.

---

## 35. What is an LSI?

1. LSI stands for Local Secondary Index.
2. It uses the same partition key as the base table.
3. It provides an alternate sort key.
4. It must be defined when the table is created.
5. It has different constraints from GSIs.
6. Its use should be based on specific query requirements.

---

## 36. GSI vs LSI?

1. A GSI can have a different partition key.
2. An LSI shares the base table's partition key.
3. GSIs can be added after table creation.
4. LSIs must be created with the table.
5. Their consistency and capacity characteristics differ.
6. GSIs are generally more flexible for new access patterns.

---

## 37. Why shouldn't I create too many GSIs?

1. Each GSI consumes additional storage.
2. Writes to the base table can propagate to relevant indexes.
3. Additional indexes increase operational complexity.
4. Index projections can increase storage and capacity usage.
5. Unused indexes create unnecessary cost.
6. Every index should correspond to a real application access pattern.

---

## 38. What is index projection?

1. Projection determines which attributes appear in an index.
2. `KEYS_ONLY` keeps only key attributes.
3. `INCLUDE` stores selected additional attributes.
4. `ALL` stores all attributes.
5. Smaller projections can reduce storage requirements.
6. Choose the projection based on actual query requirements.

---

# Streams, TTL & Events

## 39. What is DynamoDB Streams?

1. Streams capture item-level changes.
2. Events can represent inserts, updates, and deletes.
3. They are commonly integrated with Lambda.
4. They enable event-driven architectures.
5. They can support downstream processing and auditing use cases.
6. Streams are separate from the application's normal table reads.

---

## 40. What is TTL?

1. TTL stands for Time to Live.
2. It automatically marks expired items for deletion.
3. It is useful for temporary data.
4. Common examples include sessions and temporary records.
5. TTL deletion is asynchronous rather than an exact-time deletion mechanism.
6. Applications should not depend on deletion occurring at the exact expiration timestamp.

---

## 41. Where should TTL be used?

1. Sessions are a common use case.
2. Temporary tokens can use TTL.
3. Expiring caches can use TTL.
4. Short-lived event records may use TTL.
5. Historical data requiring guaranteed retention should be handled differently.
6. TTL should support lifecycle management rather than business-critical timing.

---

## 42. How does DynamoDB support event-driven architecture?

1. DynamoDB Streams capture item changes.
2. Lambda can process stream records.
3. Events can trigger downstream workflows.
4. Applications can update search indexes or analytics systems.
5. Events can be routed into broader AWS event architectures.
6. This reduces tight coupling between producers and consumers.

---

# Transactions & Concurrency

## 43. What are DynamoDB transactions?

1. Transactions allow multiple operations to be treated atomically.
2. They can coordinate changes across multiple items.
3. They are useful for maintaining business invariants.
4. They can combine reads and writes depending on the transaction operation.
5. Transactions add overhead compared with simple operations.
6. Use them when atomicity is genuinely required.

---

## 44. What is a conditional write?

1. A conditional write executes only if a specified condition is true.
2. It helps protect business invariants.
3. It is useful for optimistic concurrency.
4. It can prevent duplicate operations.
5. It can prevent updates to invalid state.
6. It is one of the most important DynamoDB correctness mechanisms.

---

## 45. How do you implement optimistic locking?

1. Store a version number with the item.
2. Read the current version.
3. Update only if the version still matches.
4. Increment the version during a successful update.
5. A failed condition indicates another writer changed the item.
6. The application can then retry or report a conflict.

---

## 46. How do you prevent duplicate payment processing?

1. Generate a unique idempotency key.
2. Store the request/result associated with that key.
3. Use conditional writes to prevent duplicate processing.
4. Treat repeated requests as the same logical operation.
5. Make retries safe.
6. Combine this with transactional logic where necessary.

---

# Security

## 47. How is DynamoDB secured?

1. IAM controls API access.
2. Encryption protects data at rest.
3. TLS protects data in transit.
4. VPC endpoints can provide private connectivity from VPC environments.
5. CloudTrail provides API auditing.
6. Fine-grained IAM policies can restrict access.

---

## 48. What is the role of IAM with DynamoDB?

1. IAM determines who can call DynamoDB APIs.
2. Policies can allow or deny specific actions.
3. Access can be scoped to particular tables.
4. Conditions can further restrict permissions.
5. Applications should use least privilege.
6. Avoid broad permissions such as unrestricted DynamoDB access.

---

## 49. How is DynamoDB encrypted?

1. DynamoDB encrypts data at rest.
2. AWS-managed encryption options are available.
3. Customer-managed KMS keys can be used where required.
4. Encryption should be considered as part of compliance requirements.
5. Data in transit is protected using TLS.
6. Key-management permissions must also be carefully controlled.

---

## 50. Why use VPC endpoints for DynamoDB?

1. They can provide private connectivity from VPC resources.
2. Traffic does not need to traverse the public internet.
3. They can simplify network security requirements.
4. They can be useful for private application architectures.
5. Endpoint policies can add another layer of access control.
6. They are especially useful for workloads operating inside private subnets.

---

# Global Tables & DR

## 51. What are DynamoDB Global Tables?

1. Global Tables provide multi-Region replicated DynamoDB tables.
2. They support geographically distributed applications.
3. Applications can access local regional replicas.
4. They improve regional resilience.
5. They require careful consideration of multi-Region writes.
6. They are useful when global availability or low-latency regional access is required.

---

## 52. Why use Global Tables?

1. To provide multi-Region data availability.
2. To reduce latency for geographically distributed users.
3. To support regional disaster recovery.
4. To build globally distributed applications.
5. To reduce dependence on a single AWS Region.
6. The entire application stack must also be designed for regional resilience.

---

## 53. What happens during a Region failure?

1. Traffic should be redirected to a healthy Region.
2. The application must be capable of operating there.
3. DynamoDB Global Tables can provide replicated data.
4. IAM, networking, queues, compute, and dependencies must also be available.
5. Failover procedures should be tested.
6. A multi-Region database alone does not create a complete DR solution.

---

## 54. What is RPO?

1. RPO means Recovery Point Objective.
2. It represents acceptable data loss measured in time.
3. A lower RPO generally requires stronger replication/recovery mechanisms.
4. Different workloads can have different RPO requirements.
5. Global replication can support very low recovery-point objectives.
6. RPO must be defined as a business requirement.

---

## 55. What is RTO?

1. RTO means Recovery Time Objective.
2. It represents how quickly the system must recover.
3. A low RTO requires fast and tested recovery procedures.
4. Automated failover can reduce recovery time.
5. Manual recovery usually increases RTO.
6. RTO must be tested rather than assumed.

---

# Backup & Restore

## 56. What is Point-in-Time Recovery?

1. PITR allows recovery of a table to a point within the supported recovery window.
2. It protects against accidental deletion or corruption.
3. It should be enabled for important production tables.
4. Recovery is separate from normal application operations.
5. Restored data should be validated before production use.
6. PITR is an important component of operational resilience.

---

## 57. Backup vs Global Tables?

1. Backups are primarily for data recovery.
2. Global Tables provide multi-Region replicated availability.
3. Backup does not automatically provide real-time application failover.
4. Global Tables do not replace all backup requirements.
5. Production systems often need both.
6. Choose each capability based on its specific recovery objective.

---

# Cost Optimization

## 58. What drives DynamoDB cost?

1. Read and write request usage or provisioned capacity.
2. Table and index storage.
3. Global replication.
4. Backups and recovery features.
5. Streams and related workloads.
6. Data transfer and surrounding architecture can also contribute.

---

## 59. How do you reduce DynamoDB cost?

1. Eliminate unnecessary GSIs.
2. Avoid inefficient scans.
3. Reduce unnecessarily large items.
4. Choose the right capacity mode.
5. Optimize access patterns.
6. Monitor storage, indexes, backups, and replication continuously.

---

## 60. How do large items affect cost?

1. Larger items require more capacity per operation.
2. They also increase storage requirements.
3. Large responses increase network transfer.
4. Unnecessary attributes should be removed.
5. Large files should generally be stored in object storage instead.
6. DynamoDB should primarily store structured operational data.

---

## 61. Why should large files be stored in S3 instead of DynamoDB?

1. S3 is designed for object storage.
2. Large binaries can make DynamoDB items unnecessarily large.
3. DynamoDB is optimized for low-latency operational data access.
4. Storing metadata in DynamoDB keeps items smaller.
5. The DynamoDB item can contain an S3 object key.
6. This architecture separates metadata from large payload storage.

---

# Observability

## 62. How do you monitor DynamoDB?

1. Use CloudWatch metrics.
2. Monitor throttling.
3. Monitor latency.
4. Monitor errors and consumed capacity.
5. Monitor application-level business metrics.
6. Build dashboards and alarms around production requirements.

---

## 63. Which DynamoDB metrics are important?

1. Consumed read capacity.
2. Consumed write capacity.
3. Throttled requests.
4. Request latency.
5. System and user errors.
6. Additional metrics should be selected according to the capacity mode and workload.

---

## 64. Why monitor P95 and P99 latency?

1. Averages can hide slow requests.
2. P95 shows the experience of slower users.
3. P99 exposes extreme latency.
4. Distributed applications often care about tail latency.
5. A low average with high P99 can still indicate a serious problem.
6. Production SLOs should consider tail latency.

---

## 65. What is CloudTrail used for?

1. CloudTrail records AWS API activity.
2. It helps identify who performed administrative operations.
3. It supports security investigations.
4. It assists with compliance and auditing.
5. It can help correlate configuration changes with incidents.
6. It complements CloudWatch operational monitoring.

---

# Troubleshooting

## 66. DynamoDB is throttling. What do you check first?

1. Determine whether the problem is table-level or index-related.
2. Look for hot partition-key values.
3. Check traffic changes.
4. Check recent deployments.
5. Review retry behavior.
6. Determine whether capacity mode is appropriate.

---

## 67. Increasing capacity didn't solve throttling. Why?

1. The workload may have a hot partition.
2. Traffic may be concentrated on a small number of keys.
3. A GSI may be experiencing its own constraint.
4. Application retries may be amplifying traffic.
5. The access pattern may be inefficient.
6. Capacity alone cannot fix fundamentally poor key distribution.

---

## 68. DynamoDB latency suddenly increased. What do you investigate?

1. Check throttling.
2. Check P95/P99 latency.
3. Look for hot keys.
4. Review recent application deployments.
5. Investigate retries and downstream dependencies.
6. Compare current traffic with normal traffic.

---

## 69. Why can retries make an outage worse?

1. A failed request triggers another request.
2. Multiple clients can retry simultaneously.
3. This increases traffic during an already overloaded period.
4. Without backoff, the system can enter a retry storm.
5. Jitter spreads retries over time.
6. Retry limits and idempotency are critical.

---

## 70. What should you do when a Scan is causing performance issues?

1. Identify why the Scan exists.
2. Determine the actual business access pattern.
3. Consider a Query-based design.
4. Add an appropriate GSI if justified.
5. Reduce unnecessary attributes and result size.
6. Avoid turning a frequently used Scan into a permanent production architecture.

---

# Migration

## 71. How do you migrate from PostgreSQL to DynamoDB?

1. Identify application access patterns.
2. Redesign the relational model for DynamoDB.
3. Create the target table and indexes.
4. Perform an initial data load.
5. Synchronize ongoing changes where required.
6. Validate and perform a controlled cutover.

---

## 72. Why can't I simply convert SQL tables to DynamoDB tables?

1. SQL schemas are optimized around relational modeling.
2. DynamoDB is optimized around access patterns.
3. SQL joins don't translate directly.
4. Normalized structures may cause inefficient DynamoDB access.
5. Denormalization may be required.
6. Migration is therefore both a data and application redesign exercise.

---

## 73. What is dual-write migration?

1. The application writes to both old and new systems.
2. The old system may remain the source of truth initially.
3. DynamoDB is populated and validated.
4. Read traffic can gradually move to DynamoDB.
5. Eventually DynamoDB can become the primary system.
6. Dual writes require reconciliation and failure-handling logic.

---

## 74. What is CDC in migration?

1. CDC means Change Data Capture.
2. It captures changes occurring in the source system.
3. Those changes are transformed for DynamoDB.
4. It keeps the target closer to source state during migration.
5. It can reduce application downtime during cutover.
6. CDC pipelines need monitoring and error handling.

---

## 75. What should be validated during migration?

1. Record counts.
2. Key correctness.
3. Attribute correctness.
4. Duplicate and missing records.
5. Application query behavior.
6. Performance and business functionality.

---

# Production Architecture

## 76. What does a production-grade DynamoDB architecture look like?

1. Well-designed partition and sort keys.
2. Minimal and purposeful indexes.
3. Appropriate capacity mode.
4. Strong security and IAM controls.
5. Backup and disaster-recovery mechanisms.
6. CloudWatch, CloudTrail, alerts, and operational runbooks.

---

## 77. How do you design DynamoDB for high availability?

1. Use DynamoDB's managed multi-AZ architecture.
2. Avoid application dependencies on a single instance.
3. Use appropriate backup capabilities.
4. Consider Global Tables for multi-Region requirements.
5. Design the application for retry and failover.
6. Test recovery rather than assuming it works.

---

## 78. How do you design DynamoDB for millions of users?

1. Choose highly distributed partition keys.
2. Model queries around predictable access patterns.
3. Avoid hot keys.
4. Select capacity mode according to workload.
5. Monitor throttling and latency.
6. Load-test at realistic scale before production.

---

## 79. How do you design DynamoDB for global users?

1. Determine whether users need local data access.
2. Evaluate Global Tables.
3. Define regional write/read behavior.
4. Design conflict handling for multi-Region writes.
5. Build regional failover for the application layer.
6. Test Region-level failure scenarios.

---

## 80. How do you design DynamoDB for a highly transactional application?

1. Identify operations requiring atomicity.
2. Use conditional writes for state validation.
3. Use transactions when multiple items must change atomically.
4. Implement idempotency.
5. Handle retries carefully.
6. Build strong audit and observability capabilities.

---

# Real-Time Scenarios

## 81. How would you design a shopping cart?

1. Use the customer or cart identifier as the partition key.
2. Use product/item identifiers as sort keys.
3. Update quantities with targeted operations.
4. Use conditional logic where inventory or state validation is needed.
5. Use TTL for temporary carts if appropriate.
6. Keep large product metadata outside the cart item when unnecessary.

---

## 82. How would you design an order-management system?

1. Model customer-to-order access directly.
2. Use composite keys to represent orders and related entities.
3. Use GSIs for additional required queries.
4. Use conditional writes for valid state transitions.
5. Use Streams for asynchronous downstream processing.
6. Design backup and DR according to business requirements.

---

## 83. How would you design inventory management?

1. Store inventory using a well-distributed product key.
2. Use conditional updates to prevent negative inventory.
3. Use atomic counters where appropriate.
4. Use transactions when multiple inventory changes must be atomic.
5. Make reservation operations idempotent.
6. Monitor hot products that receive extremely high traffic.

---

## 84. How would you design a user-session store?

1. Use user/session identifiers as keys.
2. Store session state in compact items.
3. Apply TTL to expire sessions.
4. Use conditional updates for security-sensitive state.
5. Monitor session lookup latency.
6. Consider availability and failover requirements.

---

## 85. How would you design an IoT workload?

1. Expect very high write volumes.
2. Avoid using a single device or global value as an overloaded key.
3. Consider time bucketing and sharding.
4. Design queries around the required time windows.
5. Use Streams for downstream processing when appropriate.
6. Archive historical data to specialized storage when necessary.

---

## 86. How would you design a social-media feed?

1. Identify whether reads or writes are the dominant workload.
2. Consider precomputed feeds.
3. Use denormalization to avoid expensive dynamic joins.
4. Consider fan-out strategies.
5. Watch for celebrity/high-follower hot keys.
6. Use asynchronous processing for feed generation where appropriate.

---

## 87. How would you handle a sudden 10x traffic spike?

1. Determine whether the table uses on-demand or provisioned capacity.
2. Check whether traffic is evenly distributed.
3. Look for hot partition keys.
4. Monitor throttling and latency.
5. Ensure retries use exponential backoff and jitter.
6. Identify whether the spike is legitimate or caused by an application bug.

---

## 88. What would you do if a production deployment caused throttling?

1. Compare metrics before and after deployment.
2. Identify new access patterns.
3. Check for new scans or inefficient queries.
4. Inspect partition-key distribution.
5. Roll back if the deployment is the likely cause.
6. Fix and load-test the new access pattern before redeployment.

---

## 89. What would you do if the DynamoDB bill suddenly doubled?

1. Review request volume.
2. Review table and GSI storage.
3. Check for newly created indexes.
4. Check Global Tables or replication changes.
5. Check backup and Streams usage.
6. Correlate the cost increase with deployments and traffic changes.

---

## 90. What would you do after accidental deletion of production data?

1. Stop additional destructive activity.
2. Identify exactly what data was affected.
3. Determine the required recovery point.
4. Use the appropriate backup/PITR mechanism.
5. Restore and validate data.
6. Reconcile restored data with application state before returning to normal operations.

---

## 91. What would you do if P99 latency suddenly reached seconds?

1. Check DynamoDB throttling.
2. Check hot partitions.
3. Check recent application changes.
4. Examine retry volume.
5. Check downstream services.
6. Compare P50, P95, and P99 to isolate tail-latency behavior.

---

## 92. What would you do if one customer causes most DynamoDB traffic?

1. Identify the customer's partition key.
2. Determine whether that key is becoming hot.
3. Measure read/write distribution.
4. Consider sharding or bucketing the customer's data.
5. Update application access patterns accordingly.
6. Load-test the redesigned key strategy.

---

## 93. How would you handle a very popular product?

1. A single product may become a hot key.
2. Reads can potentially be cached where appropriate.
3. Writes may require sharding depending on workload.
4. Product metadata can be separated from high-frequency transactional data.
5. Monitor request distribution.
6. Design specifically for extreme popularity rather than average traffic.

---

## 94. How would you design a payment system using DynamoDB?

1. Use a unique idempotency key.
2. Use conditional writes to prevent duplicate processing.
3. Use transactions when multiple state changes require atomicity.
4. Store payment state transitions carefully.
5. Maintain audit information.
6. Never rely on client-side prevention of duplicate requests.

---

## 95. How would you design DynamoDB for disaster recovery?

1. Define RPO and RTO first.
2. Enable appropriate backup/PITR capabilities.
3. Consider Global Tables for multi-Region requirements.
4. Replicate or recreate dependent infrastructure.
5. Automate recovery procedures where possible.
6. Perform regular DR exercises.

---

# Advanced Interview Questions

## 96. What is the most common DynamoDB design mistake?

1. Designing the table like a relational database.
2. Starting with entities instead of access patterns.
3. Using poor partition keys.
4. Creating excessive indexes.
5. Depending heavily on Scan.
6. Failing to consider traffic distribution at production scale.

---

## 97. What is the most important DynamoDB performance principle?

1. Design efficient access patterns first.
2. Use appropriate partition keys.
3. Avoid hot partitions.
4. Prefer Query over Scan.
5. Keep items appropriately sized.
6. Monitor actual production behavior.

---

## 98. What is the most important DynamoDB cost principle?

1. Avoid unnecessary reads and writes.
2. Avoid unnecessary indexes.
3. Keep item sizes under control.
4. Choose capacity mode based on workload.
5. Monitor storage and replication.
6. Optimize the data model before optimizing individual API calls.

---

## 99. What is the most important DynamoDB production principle?

1. Treat data modeling as an application architecture decision.
2. Design around access patterns.
3. Build for retries and concurrency.
4. Monitor continuously.
5. Have backup and recovery procedures.
6. Test failure scenarios before production incidents occur.

---

# 100. How would you explain your complete DynamoDB strategy in an interview?

1. **Model:** "I start with access patterns and design partition/sort keys accordingly."
2. **Scale:** "I ensure traffic is distributed and avoid hot partitions."
3. **Performance:** "I favor efficient Query operations and carefully designed indexes."
4. **Reliability:** "I use conditional writes, transactions, idempotency, backups, and appropriate DR architecture."
5. **Operations:** "I monitor throttling, latency, errors, cost, and application-level metrics."
6. **Production:** "For migration or major redesigns, I use controlled rollout, validation, observability, and a tested rollback strategy."

---

# 🔥 100-Question DynamoDB Revision Map

For quick revision, remember the progression:

```text
01–05   Fundamentals
06–15   Keys & Data Modeling
16–23   CRUD, Query, Scan & Pagination
24–26   Consistency
27–33   Capacity & Scaling
34–38   GSI / LSI / Indexes
39–42   Streams / TTL / Events
43–46   Transactions / Concurrency
47–50   Security
51–57   Global Tables / DR / Backup
58–61   Cost Optimization
62–65   Monitoring / Observability
66–70   Troubleshooting
71–75   Migration
76–80   Production Architecture
81–95   Real-Time Scenarios
96–100  Advanced Interview Questions
```

## 🏆 The 10 DynamoDB Questions You Absolutely Must Master

If you're preparing for an **AWS/DynamoDB production interview**, make these automatic:

1. **How do you design a partition key?**
2. **How do you prevent hot partitions?**
3. **Query vs Scan?**
4. **GSI vs LSI?**
5. **How do you model a one-to-many relationship?**
6. **How do conditional writes and transactions work?**
7. **How do you troubleshoot DynamoDB throttling?**
8. **How do Global Tables and DR work?**
9. **How do you optimize DynamoDB cost?**
10. **How would you design DynamoDB for a real production workload from scratch?**

These 100 FAQs cover the complete progression from **DynamoDB fundamentals → data modeling → performance → scaling → security → Global Tables → DR → cost → observability → migration → production troubleshooting → real-world architecture**.
