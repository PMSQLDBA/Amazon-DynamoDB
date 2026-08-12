# Amazon DynamoDB — Top 50 Real-Time Scenario-Based Questions

## 1. Your DynamoDB table is suddenly getting throttled. What do you do?

1. Check CloudWatch for throttled requests and consumed capacity.
2. Determine whether throttling is table-wide, partition-specific, or related to a GSI.
3. Analyze partition-key distribution for hot keys.
4. Check for a recent deployment or traffic spike.
5. Verify application retries use exponential backoff and jitter.
6. Fix the underlying key/capacity problem rather than simply increasing capacity.

---

## 2. Increasing capacity didn't stop throttling. Why?

1. The workload may have a hot partition.
2. A small number of partition-key values may receive most traffic.
3. A GSI may be throttling independently.
4. Retry storms may be generating additional traffic.
5. The access pattern itself may be inefficient.
6. Increasing total capacity doesn't necessarily fix uneven traffic distribution.

---

## 3. One customer is generating massive traffic. What is your solution?

1. Identify the customer's partition-key access pattern.
2. Determine whether the customer has become a hot key.
3. Split or shard the customer's data if necessary.
4. Use a deterministic sharding strategy so reads know where to look.
5. Consider caching extremely hot read data.
6. Test the redesigned access pattern before production rollout.

---

## 4. A product suddenly becomes viral and receives millions of requests. How do you handle it?

1. Determine whether the product ID is creating a hot key.
2. Separate static product metadata from high-frequency transactional data.
3. Cache frequently read immutable or slowly changing information where appropriate.
4. Shard extremely hot write workloads when required.
5. Monitor P95/P99 latency and throttling.
6. Design specifically for hot-item traffic instead of average traffic.

---

## 5. Your application performs millions of Scan operations. What would you do?

1. Identify why the application needs Scan.
2. Determine the actual business access pattern.
3. Replace recurring scans with Query where possible.
4. Add a GSI if a different access pattern is required.
5. Use pagination for unavoidable large scans.
6. Consider whether historical/analytical data belongs in another system.

---

## 6. A Query is returning too much data and becoming slow. What do you do?

1. Review the partition-key and sort-key design.
2. Add a more restrictive sort-key condition.
3. Use projection to return only required attributes.
4. Implement pagination.
5. Avoid retrieving large unnecessary items.
6. Consider whether a GSI would provide a better access path.

---

## 7. Your DynamoDB P99 latency suddenly jumps from milliseconds to seconds.

1. Check throttling metrics first.
2. Look for hot partitions or hot keys.
3. Check recent deployments.
4. Examine application retry volume.
5. Check downstream dependencies and network behavior.
6. Compare P50, P95, and P99 to determine whether the issue is isolated to tail latency.

---

## 8. Users sometimes don't see data immediately after writing it. Why?

1. The application may be using eventually consistent reads.
2. Check whether the read requires strong consistency.
3. Use a strongly consistent read where supported and justified.
4. Verify whether the read is coming from a GSI or replicated environment.
5. Determine whether the business actually requires immediate visibility.
6. Avoid globally enabling strong consistency without a business reason.

---

## 9. Your application accidentally overwrites another user's update. How do you prevent this?

1. Implement optimistic concurrency control.
2. Store a version number with each item.
3. Read the current version before updating.
4. Use a condition that the version hasn't changed.
5. Increment the version after a successful update.
6. Retry or report a conflict when the condition fails.

---

## 10. Two users try to purchase the last item simultaneously. What do you do?

1. Store the current inventory count.
2. Use a conditional update.
3. Require inventory to be greater than zero.
4. Atomically decrement inventory.
5. Reject the second request when the condition fails.
6. Use transactions if multiple related records must change atomically.

---

# 🔥 High-Scale Production Scenarios

## 11. Your application suddenly receives 10x normal traffic.

1. Check whether the workload is evenly distributed.
2. Monitor throttling and latency.
3. Verify the selected capacity mode.
4. Identify hot partition keys.
5. Ensure retries use exponential backoff with jitter.
6. Determine whether the traffic spike is legitimate or caused by an application problem.

---

## 12. A new deployment causes DynamoDB costs to double.

1. Compare request volume before and after deployment.
2. Check whether new GSIs were introduced.
3. Look for inefficient Query or Scan patterns.
4. Check item-size changes.
5. Review retry behavior.
6. Correlate the cost increase with application changes.

---

## 13. Your DynamoDB bill increases unexpectedly every month.

1. Analyze read/write request trends.
2. Review table and GSI storage growth.
3. Check backup and PITR usage.
4. Check Global Tables replication.
5. Identify unused or unnecessary indexes.
6. Use AWS cost monitoring and CloudWatch to establish the exact cost driver.

---

## 14. A GSI is consuming significantly more capacity than expected.

1. Review the GSI's access patterns.
2. Check which attributes are projected.
3. Determine whether the GSI is actually required.
4. Review write traffic affecting the index.
5. Look for inefficient application queries.
6. Remove or redesign the index if it doesn't support a valuable access pattern.

---

## 15. You discover that one partition receives most of the traffic.

1. Identify the partition-key value responsible.
2. Determine whether the value has unusually high popularity.
3. Evaluate whether key sharding is possible.
4. Introduce a suffix/prefix or bucket strategy if appropriate.
5. Update read logic to locate the correct shard(s).
6. Load-test the new distribution.

---

## 16. You need to store millions of IoT events per minute. How would you design it?

1. Avoid using a single global partition key.
2. Distribute events across device/time buckets.
3. Use time-based sort keys where appropriate.
4. Consider write sharding for extremely high-volume devices.
5. Use Streams for downstream processing when required.
6. Move older analytical data to appropriate long-term storage.

---

## 17. An IoT device suddenly sends 100x its normal traffic.

1. Identify the device's partition-key distribution.
2. Determine whether it is creating a hot partition.
3. Apply throttling at the application/API layer if necessary.
4. Use write sharding when the data model permits it.
5. Monitor downstream processing.
6. Investigate whether the device is malfunctioning.

---

## 18. Your application needs to retrieve the latest 10 orders for a customer.

1. Use the customer ID as the partition key.
2. Store order timestamp/order ID in the sort key.
3. Query that customer's partition.
4. Sort in descending order when appropriate.
5. Limit the result to the required number of items.
6. Avoid scanning the entire table.

---

## 19. You need to retrieve all orders for a customer during a date range.

1. Use customer ID as the partition key.
2. Store an appropriate timestamp-based sort key.
3. Use a Query with a sort-key range condition.
4. Avoid Scan.
5. Paginate if the result is large.
6. Project only the attributes needed by the application.

---

## 20. You need to find orders by status across all customers.

1. The base customer key may not support this access pattern.
2. Identify whether the query is business-critical.
3. Create a GSI with status as an appropriate partition key.
4. Use a timestamp/order identifier as the GSI sort key if needed.
5. Avoid using Scan for recurring production queries.
6. Verify that the GSI doesn't create an undesirable hot key such as a single dominant status.

---

# 🔐 Security Scenarios

## 21. An application has full DynamoDB access. How would you improve security?

1. Apply least-privilege IAM policies.
2. Restrict access to only required tables.
3. Restrict actions such as GetItem, PutItem, UpdateItem, and DeleteItem.
4. Use IAM roles instead of long-lived credentials.
5. Add appropriate IAM conditions.
6. Audit access through CloudTrail.

---

## 22. Your security team requires private connectivity to DynamoDB.

1. Deploy the application inside the appropriate VPC architecture.
2. Use a DynamoDB VPC endpoint where appropriate.
3. Restrict endpoint access using policies.
4. Review route and security configuration.
5. Ensure applications don't require unnecessary public connectivity.
6. Validate connectivity from private subnets.

---

## 23. Your company requires customer-managed encryption keys.

1. Use an appropriate AWS KMS key.
2. Configure DynamoDB encryption accordingly.
3. Control KMS permissions carefully.
4. Monitor key usage.
5. Establish key rotation and lifecycle procedures.
6. Validate that applications and operational roles retain required permissions.

---

## 24. A developer accidentally deletes production data. What do you do?

1. Stop further destructive operations.
2. Determine what data was deleted and when.
3. Identify the appropriate recovery point.
4. Use PITR or backup restoration.
5. Validate the restored dataset.
6. Implement stronger IAM controls and operational safeguards afterward.

---

## 25. You suspect unauthorized DynamoDB API activity.

1. Review CloudTrail events.
2. Identify the IAM principal involved.
3. Determine which APIs were called.
4. Review the affected tables/resources.
5. Revoke or restrict compromised credentials/roles as appropriate.
6. Perform a broader security investigation and document the incident.

---

# ⚡ Transactions & Business Logic

## 26. You need to create an order and decrease inventory atomically.

1. Identify all records that must change together.
2. Use DynamoDB transactions when atomicity across multiple items is required.
3. Validate inventory conditions.
4. Create/update the order as part of the same transaction.
5. Handle transaction conflicts and retries.
6. Make the operation idempotent.

---

## 27. A payment API sends the same request three times. How do you avoid duplicate payment records?

1. Generate a unique idempotency key.
2. Store that key with the transaction state.
3. Use a conditional write to prevent duplicate creation.
4. Return the existing result for repeated requests.
5. Make retries safe.
6. Keep an audit trail for payment state transitions.

---

## 28. An order must move from `PENDING` to `PAID`, but another process already changed it to `CANCELLED`.

1. Use a conditional update.
2. Require the current status to be `PENDING`.
3. Attempt the state transition atomically.
4. If the condition fails, don't overwrite the newer state.
5. Return the conflict to the business workflow.
6. Log the state-transition failure for investigation.

---

## 29. Multiple services update the same item concurrently. How do you prevent lost updates?

1. Use conditional writes.
2. Add a version attribute.
3. Require the expected version during updates.
4. Increment the version after a successful update.
5. Handle conditional-check failures.
6. Use transactions when multiple items must remain consistent.

---

## 30. A transaction frequently fails because another process is updating the same records.

1. Determine whether contention is expected.
2. Reduce unnecessary transaction scope.
3. Keep transactions as short as possible.
4. Use conditional writes when a full transaction isn't necessary.
5. Implement bounded retries with backoff.
6. Reconsider the data model if contention is consistently high.

---

# 🌎 Global Tables & Disaster Recovery

## 31. Your application serves customers in the US, Europe, and Asia. Would you use Global Tables?

1. First determine whether local regional latency is a requirement.
2. Determine whether multi-Region availability is required.
3. Evaluate Global Tables for replicated DynamoDB data.
4. Design regional application deployments as well.
5. Define conflict-resolution behavior for multi-Region writes.
6. Test Region-level failover.

---

## 32. AWS Region A goes down. How do you recover the application?

1. Detect the regional failure.
2. Redirect application traffic to the healthy Region.
3. Ensure DynamoDB data is available there.
4. Ensure compute, networking, IAM, queues, and other dependencies are also available.
5. Validate application health.
6. Execute and measure the documented DR runbook.

---

## 33. Your company requires near-zero data loss during a regional disaster.

1. Define the actual RPO requirement.
2. Evaluate multi-Region replication.
3. Use Global Tables where appropriate.
4. Design applications for regional writes/failover.
5. Identify conflict-handling requirements.
6. Perform regular DR testing.

---

## 34. Management asks, "Do Global Tables alone give us disaster recovery?"

1. No, not for the complete application.
2. They address multi-Region DynamoDB data availability.
3. Compute must also exist in the recovery Region.
4. Networking and IAM must be ready.
5. Dependent services must also have recovery strategies.
6. End-to-end DR requires testing the entire application stack.

---

## 35. Your company requires an RTO of 15 minutes. How do you design for it?

1. Identify every component required for application recovery.
2. Pre-provision or automate recovery infrastructure.
3. Use appropriate DynamoDB recovery capabilities.
4. Automate DNS/traffic failover where appropriate.
5. Create a documented recovery runbook.
6. Perform DR exercises and measure the actual recovery time.

---

# 🔄 Streams & Event-Driven Scenarios

## 36. Every order update must trigger a notification. How would you implement it?

1. Enable DynamoDB Streams.
2. Capture relevant item changes.
3. Configure Lambda or another consumer.
4. Filter events based on the required state changes.
5. Trigger the notification workflow.
6. Make downstream processing idempotent.

---

## 37. A DynamoDB Stream consumer is failing repeatedly. What do you investigate?

1. Check Lambda/application errors.
2. Inspect failed records.
3. Determine whether a specific item is causing the failure.
4. Check consumer processing time and throughput.
5. Review retry and failure-handling behavior.
6. Ensure problematic records don't permanently block healthy processing.

---

## 38. You need to update a search index whenever DynamoDB data changes.

1. Enable DynamoDB Streams.
2. Process stream records asynchronously.
3. Transform the DynamoDB item into the search document.
4. Update the search system.
5. Make processing idempotent.
6. Monitor stream processing failures and lag.

---

## 39. You need to automatically remove sessions after 24 hours.

1. Store an expiration timestamp.
2. Configure TTL on the appropriate attribute.
3. Allow DynamoDB to remove expired records asynchronously.
4. Don't depend on deletion occurring at the exact second.
5. Ensure application authorization doesn't rely solely on physical deletion.
6. Monitor storage and session behavior.

---

## 40. You need to retain transaction history for seven years but want DynamoDB storage costs low.

1. Keep operational data in DynamoDB only as long as required.
2. Use an appropriate archival strategy.
3. Export older records to long-term storage.
4. Keep DynamoDB focused on active operational access.
5. Apply lifecycle policies to archived data.
6. Ensure the retention strategy satisfies compliance requirements.

---

# 🚨 Incident & Production Scenarios

## 41. Your DynamoDB application starts returning `ConditionalCheckFailedException`. What does it mean?

1. A conditional operation's condition evaluated to false.
2. It isn't necessarily a DynamoDB infrastructure failure.
3. Another process may have changed the item.
4. The item may already exist or not exist as expected.
5. The application should handle the condition according to business logic.
6. Investigate whether the failure rate is expected or indicates contention.

---

## 42. Your application receives many `ProvisionedThroughputExceededException` errors.

1. Check provisioned capacity consumption.
2. Look for hot partitions.
3. Check whether a GSI is constrained.
4. Verify retry/backoff behavior.
5. Consider scaling capacity if the workload genuinely increased.
6. Redesign the key if distribution is the actual problem.

---

## 43. DynamoDB works normally, but your application is still slow. What do you check?

1. Check DynamoDB latency independently from application latency.
2. Inspect application processing time.
3. Check network behavior.
4. Check downstream services.
5. Check serialization/deserialization overhead.
6. Trace the entire request using distributed tracing/observability tools.

---

## 44. A new GSI causes unexpected production impact. What do you investigate?

1. Determine the GSI's key distribution.
2. Review its capacity consumption.
3. Check write amplification.
4. Review projected attributes and item size.
5. Identify application queries using the new index.
6. Reduce, redesign, or remove the index if necessary.

---

## 45. Your application needs a query that DynamoDB doesn't support efficiently. What do you do?

1. Don't immediately resort to Scan.
2. Identify the exact business access pattern.
3. Consider changing the primary-key design.
4. Consider a GSI.
5. Consider denormalizing data.
6. If the query is analytical, consider moving it to an analytics-oriented service.

---

# 🏗️ Architecture & Migration Scenarios

## 46. You are migrating a large PostgreSQL application to DynamoDB. Where do you start?

1. Inventory the application's existing queries.
2. Identify the most important access patterns.
3. Design DynamoDB keys around those patterns.
4. Identify relationships that require denormalization.
5. Design migration and synchronization strategy.
6. Load-test before production cutover.

---

## 47. You want zero/minimal downtime during migration from PostgreSQL.

1. Perform an initial bulk data migration.
2. Establish CDC or another change-synchronization mechanism.
3. Keep DynamoDB synchronized with source changes.
4. Validate data correctness continuously.
5. Gradually move read traffic.
6. Perform controlled write cutover with rollback capability.

---

## 48. Your DynamoDB data model works in testing but fails at production scale. Why?

1. Test data may not represent production cardinality.
2. Test traffic may not represent real traffic distribution.
3. Hot keys may not appear in small datasets.
4. Production item sizes may be much larger.
5. Query patterns may differ from test assumptions.
6. Realistic load and distribution testing should occur before launch.

---

## 49. Your team wants to create a GSI for every possible future query. What do you say?

1. Don't create indexes without real access patterns.
2. Every index introduces additional storage and operational considerations.
3. Write workloads can be affected by index maintenance.
4. Excessive indexes increase architecture complexity.
5. Start with known business-critical queries.
6. Add indexes when actual requirements justify them.

---

## 50. You're asked to design a production DynamoDB system from scratch. Walk through your approach.

1. **Requirements:** Define traffic, latency, consistency, availability, RPO, and RTO.
2. **Access patterns:** Identify every critical read/write operation.
3. **Data model:** Design partition keys, sort keys, GSIs, and denormalization around those patterns.
4. **Reliability:** Add conditional writes, transactions, idempotency, backups, and DR as required.
5. **Operations:** Implement CloudWatch monitoring, alarms, CloudTrail auditing, logging, and cost monitoring.
6. **Validation:** Load-test realistic traffic, simulate failures, verify recovery, and establish production runbooks.

---

# 🔥 Top 10 Scenarios to Master First

If you're doing **DynamoDB production interviews**, prioritize these:

| Priority | Scenario                                         |
| -------- | ------------------------------------------------ |
| 🔥 1     | Hot partition / hot key                          |
| 🔥 2     | DynamoDB throttling                              |
| 🔥 3     | Query vs Scan redesign                           |
| 🔥 4     | GSI design                                       |
| 🔥 5     | Inventory concurrency                            |
| 🔥 6     | Idempotent payment/order processing              |
| 🔥 7     | DynamoDB Global Tables DR                        |
| 🔥 8     | DynamoDB cost suddenly increases                 |
| 🔥 9     | PostgreSQL → DynamoDB migration                  |
| 🔥 10    | Design complete production DynamoDB architecture |

### The interview mindset

For almost **any DynamoDB scenario**, structure your answer around:

**Access Pattern → Key Design → Distribution → Capacity → Consistency → Failure Handling → Security → Observability → Cost → DR**

That framework will let you answer most **senior-level DynamoDB production scenarios** systematically.
