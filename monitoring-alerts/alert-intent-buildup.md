# Alert: Intent Buildup

### Purpose of this Alert

This alert brings operator's attention to very large transactions that are [locking](https://www.cockroachlabs.com/docs/v22.1/architecture/transaction-layer.html#write-intents) millions of keys (rows). A common example of such transaction is a very large scope DELETE. Transactions with an excessively large scope are often inadvertent, for example due to a non-selective filter and specific data distribution that was not anticipated by an application developer.

Transactions that create a large number of intents could have a negative effect on the workload's performance:

- By creating locking contention, thus limiting concurrency, and therefore reducing the throughput, leading to stalled workloads in extreme cases
- Transactions may take non-intuitively larger amount of time to complete, with significant variations in execution latency. This would be caused by an exhaustion of the memory budget to track intents.
  The maximum number of bytes used to track locks in a transaction is configured with a cluster setting `kv.transaction.max_intents_bytes`. When a transaction modifies keys, it keeps track of the keys it has written, e.g. `a,d,g`. This allows intent resolution to know exactly which intents were written, providing a direct / point resolution.  However, if this list exceeds the memory budget, the point intent resolutions changes to range intent resolutions, which stores `a-g`, i.e. “I wrote some keys between `a` and `g`”. When another transaction  needs to resolve these intents, it has to scan everything in `a-g`, looking for those intents. Cleaning up range intents also takes a considerably larger amount of time and processing resources vs. the list of point intents.
  In v21.1 or earlier, this meant scanning all data in that range, which could take a significant amount of time, particularly when that scan encountered write intents that would block that scan.
  In v21.2, intents are stored separately from regular data, so this scan is much faster, resulting in a more limited performance overhead.

------

### Monitoring Metric

```
intentcount
```

Operators can also create custom alerts on `intentbytes` (maximum number of bytes used to track locks in transactions) to track the memory utilization against the budget (cluster setting `kv.transaction.max_intents_bytes`).

Related messages in the log:

*`... a transaction has hit the intent tracking limit (kv.transaction.max_intents_bytes); is it a bulk operation?*`*



### Alert Rule

| Tier     | Definition                                                |
| -------- | --------------------------------------------------------- |
| CRITICAL | Total cluster-wide intent count exceeds 10M for 5 minutes |
| WARNING  | Total cluster-wide intent count exceeds 10M for 2 minutes |

Operators may elect to lower the intent count threshold that triggers an alert, for tighter transaction scope scruitiny.



### Alert Response

Upon receiving this the alert, an operator can take any of the following actions:

- Identify the large scope transactions that acquire a lot of locks. Consider reducing the scope of large transactions, implementing them as several smaller scope transactions. For example, if the alert is triggered by a large scope DELETE, consider "paging" DELETEs that target thousands of records instead of millions. This is often the most effective resolution, however it generally means an application level [refactoring](https://www.cockroachlabs.com/docs/stable/bulk-update-data.html).
- As a hedge against non-intuitive SQL response time variations, adjust the cluster setting `kv.transaction.max_intents_bytes`.
  A larger value increases the memory budget for a point intent resolution list.
  In v21.1 or earlier, the default for this setting is low and can be increased to a safe limit of 4 MB.
  In v21.2 the default for this setting is [4 MB](https://github.com/cockroachdb/cockroach/issues/54029).
  Increasing this setting beyond the 21.2 default may be warranted in specific circumstances, however an operator should keep in perspective that this setting is per-transaction and therefore, if set to a high value, may lead to an out-of-memory (OOM) event in high concurrency environments.
- After reviewing the workload, an operator may conclude that a possible performance impact (discussed above) of allowing transactions to take a large number of intents is not a concern. For example, a large delete of obsolete, not-in-use data may create no concurrency implications and the elapsed time to execute that transaction may not be material. In that case, "doing-nothing" could be a valid response to this alert.
- An operator may enable a "circuit breaker" that would prevent any transaction exceeding the lock tracking memory budget. A cluster setting `kv.transaction.reject_over_max_intents_budget.enabled` allows an operator to control the behavior of CockroachDB when a transaction exceeds the memory budget for the point intents list.
  If `kv.transaction.reject_over_max_intents_budget.enabled`  is true, a SQL transaction that exceeds the intent list memory budget would be rejected with an error 53400 ("configuration limit exceeded"). 

