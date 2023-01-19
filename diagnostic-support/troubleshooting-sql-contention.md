# Troubleshooting: Workload Contention



## 1. About Workload Contention

This section includes the *background information, examples, troubleshooting technique and remediation ideas* related to *SQL workload* contention. A related topic of contention for the underlying shared computing resources is discussed in the article [Troubleshooting Hardware Resource Contention](troubleshooting-hardware-contention.md). It provides a CockroachDB practitioner with essential knowledge and remediation points for possible SQL workload contention conflicts, including:

1. **[Locking](#3.-locking-conflicts)** conflicts
2. **[Serializable isolation](#4.-serializable-isolation-conflicts)** conflicts
3. **[Uncertainty](#5.-uncertainty-conflicts)** conflicts due to a possible clock skew

Databases are purpose-built to manage concurrent access to data. Contention is not intrinsically "bad". Some form of contention is unavoidable when it represents, for example, a reality of business requirements. 

Yet there is also preventable contention that can even be eliminated entirely with a considerate schema and transaction design.

As long as an application requires concurrent access to data, all contention will not be eliminated. Contention needs to be addressed only when it manifests itself as "bad performance". There are techniques to *minimize* the performance penalties due to contention and they are covered in the Remediation chapters below.

> ✅ The most effective *solution* to a "contention problem" will always be a data model (schema) and transaction logic design that minimizes the opportunities for contention in the first place.



##### Contention Aggravating Factors

One of the main factors making a negative impact of all types of contention on the workload more pronounced is a "loose" multi-statement transaction design, when an application is doing a measurable amount non-database work while in an open transaction. A "loose" transaction may be holding locks longer than absolutely necessary, thus increasing the wait times of other transactions that are blocked on one of these locks. Or increasing the time between executions of individual statements in a transaction, thus increasing the probability of isolation conflicts.

For a quick assessment of the transactions' design, compare the [Open SQL Transactions](https://www.cockroachlabs.com/docs/stable/ui-sql-dashboard.html#open-sql-transactions) and the [Active SQL Statements](https://www.cockroachlabs.com/docs/stable/ui-sql-dashboard.html#active-sql-statements) graphs in the the [SQL Dashboard](https://www.cockroachlabs.com/docs/stable/ui-sql-dashboard.html) side by side. If the number of active statements tracks the number of open transactions, it means each open transaction is executing a statement, and that would suggest a good transaction logic implementation. Conversely, if the number of active statements lags the number of open transactions, it would suggest that some open transactions are doing non-database work while keeping a transaction open, which should probably be investigated.



##### Database Conflicts Resolution Methods

To resolve any transaction contention, a database can take one of only two available actions:

- either allow the blocked transactions to *wait*,
- or *force* one of the conflicting transactions to *re-start*

Sections below detail how CockroachDB handles the different types of concurrency conflicts with either of the methods above.





## 2. Contention Troubleshooting and Resolution Actions

A contention investigation is practically always prompted by a performance complaint, with or without `40001` error observations. The main concern is commonly a worsened response time, caused by the two attributes of contention - waits and/or retries - directly contributing to its increase.

Troubleshooting of performance issues caused by workload contention follows this path:

- **A**. Confirm that the root **cause of the performance issues is predominantly the workload contention** and not various other possible causes
- **B**. If contention is confirmed to be the principal performance issue, identify the **most impactful contention events** between transactions.
- **C**. Using the contention instrumentation data, and in collaboration with application developers, **resolve** the contention issue(s).



##### (A) Determine if the cluster issues are due predominantly the workload contention

Before focusing on contention troubleshooting, confirm that the current issues are not principally caused by other culprits from the [list of the most common problems](../most-common-problems/README.md) where contention is only one of (#4). There are 5 major possible causes, other than contention, to rule out first. Contention is practically always present in a cluster, yet contention may not necessarily be the first problem to focus on when resolving a cluster performance issue.

An operator needs to assess the current level of contention and its impact on the cluster performance vs other possible causes. A hot node starved of CPU by workload overload would be, for example, the first issue to resolve even if a significant workload contention is confirmed.

Assessing the overall level of contention in a cluster needs to account for all types of conflicts that contribute to contention - it is an aggregate of locking conflicts, isolation conflicts and uncertainty conflicts.



##### (B) Identify the most impactful contention events between transactions

> 👍 **Best Practice: Set the application name on every client connection**
>
> - Setting the [application name](https://www.cockroachlabs.com/docs/v22.1/map-sql-activity-to-app.html) on every application connection makes troubleshooting of contention much easier.
> - The application name connection tag is captured in the system tables and in DB Console, identifying the application components that issued conflicting transactions.

After contention is confirmed to have a principal performance impact (A), the next step towards the resolution of contention issues is getting detailed actionable insights into the conflicting transactions. Specifically:

- The definitions of 2 contending transactions at the level of a normalized statement fingerprint (application->connection->transaction->statement), and
- The conflict point (key)

Along with an identification of the conflict type. Visual clues  - [locking](#3.2-identifying-locking-conflicts), [isolation](#4.2-identifying-serializable-isolation-conflicts), [uncertainty](#5.2-identifying-uncertainty-conflicts) or [other](#6.-esoteric-situations-that-may-lead-to-40001-errors).

Instrumentation that provides actionable insights into the workload contention is essential to enable a cluster operator to swiftly act on performance issues caused by contention.



##### (C). Resolve the Contention Issues

Remediation techniques are available for each type of conflict -  [locking](#3.3-remediation-of-locking-conflicts), [isolation](#4.3-remediation-of-serializable-isolation-conflicts), [uncertainty](#5.3-remediation-of-uncertainty-conflicts) or [other](#remediation-of-the-closed-timestamp-related-retry-errors).





## 3. Locking Conflicts

### 3.1 Locking Conflicts Explained

All locking in CockroachDB is implemented in the KV layer and the *locking granularity is a key*.

At the SQL layer, a logical row of a relation can be represented by one or more KV pairs, according to the number of column families  and secondary indexes defined by a table DDL. Therefore at the SQL layer, the locking in CockroachDB is more granular than row-level - column families and secondary indexes can have separate locking scopes from the primary keys.

To provide concise actionable guidance, in this section we imply *row-level locking* unless specifically noted otherwise.

CockroachDB implements only one kind of lock - an *exclusive write lock* to manage concurrent access by a key. Since there is only one kind of lock, we will be referring to it as just "lock".

The implementation of write locks in CockroachDB is leveraging the system of "[write intents](https://www.cockroachlabs.com/docs/stable/architecture/transaction-layer.html#write-intents)". Therefore, a "write lock" is synonymous to "write intent", or "intent" for short.

##### **CockroachDB locking implementation highlights:**

- Writes acquire locks
- `SELECTs ... FOR UPDATE` acquire locks, same as writes
- Writes block reads and writes [from other transactions]
- Reads do not acquire locks
- Reads do not block reads or writes [from other transactions]
- All blocked statements are waiting indefinitely in the same [wait queue](https://www.cockroachlabs.com/docs/stable/architecture/transaction-layer.html#txnwaitqueue) until the blocking transaction releases the lock (aside from situations when a waiting transaction is forcefully disrupted externally, for example by a timeout of as a result of a closed connection)
- Locks are released when the holding transaction is closed (committed or rolled back)



#### Deadlock Detection and Resolution

Write-write conflicts may lead to deadlocks when different transactions acquire locks in different orders and then wait for each other to release the locks.

CockroachDB employs a distributed deadlock-detection algorithm that analyzes the [wait queue](https://www.cockroachlabs.com/docs/stable/architecture/transaction-layer.html#txnwaitqueue), which tracks the transactions that are blocked and transactions they are blocked by. When a closed loop is detected, one transaction from a cycle of waiters is forced to rollback and must be [retried](../system-overview/tech-overview-trsansaction-retires.md).

When transactions in a deadlock have the same priority, which transaction is aborted can not be predicted. If the priorities are different, the transaction with a lower priority is aborted.



#### Locking Contention Illustrations

The common types of contention scenarios are illustrated with easy-to-follow SQL examples. They may provide CockroachDB cluster operators with an additional method of learning the insights of contention handling, with SQL "scratch pad" experiments.

> Contention scenario illustrations in this section include both [implicit and explicit transactions](../system-overview/tech-overview-trsansaction-implicit-explicit.md). If a sequence of SQL statements starts with a `BEGIN`, it denotes an explicit transaction. Otherwise a transaction is implicit, single-statement.

##### Setup for All Contention Illustrations

| -- One time setup                          |
| ------------------------------------------ |
| DROP   TABLE IF EXISTS  t;                 |
| CREATE TABLE t (k INT PRIMARY KEY, v INT); |
| INSERT INTO t VALUES (1,1),(2,2),(3,3);    |



##### [No] Contention Illustration 3.1.  Reads are not blocking reads or writes

| Transaction 1 (read) | Transaction 2 (read)       | Transaction 3 (write)        |
| -------------------- | -------------------------- | ---------------------------- |
| BEGIN;               |                            |                              |
| SELECT * FROM t;     | BEGIN;                     |                              |
|                      | SELECT * FROM t WHERE k=2; | BEGIN;                       |
|                      |                            | UPDATE t SET v=21 WHERE k=2; |
|                      |                            | COMMIT;                      |
|                      | COMMIT;                    | `success`                    |
| COMMIT;              | `success`                  |                              |
| `success`            |                            |                              |



##### Contention Illustration 3.2.  Writes are blocking reads and writes

| Transaction 1 (write)          | Transaction 2 (read)       | Transaction 3 (write)           |
| ------------------------------ | -------------------------- | ------------------------------- |
| BEGIN;                         |                            |                                 |
| UPDATE t SET v=2012 WHERE k=2; | BEGIN;                     |                                 |
| `lock k=2`                     | SELECT * FROM t WHERE k=2; | BEGIN;                          |
|                                | `waiting...`               | UPDATE t SET v=2032  WHERE k=2; |
|                                |                            | `waiting...`                    |
| COMMIT;                        | `unblocked to proceed...`  | `unblocked to proceed...`       |
| `success, kv=2,2012`           | COMMIT;                    | COMMIT;                         |
|                                | `success`                  | `success, kv=2,2032`            |



##### Contention Illustration 3.3.  Deadlock (write-write closed loop conflict)

| Transaction 1 (SFU, same locking as write)           | Transaction 2 (SFU, same locking as write)            |
| ---------------------------------------------------- | ----------------------------------------------------- |
| BEGIN;                                               | BEGIN;                                                |
| SELECT * FROM t WHERE k=2 FOR UPDATE;     `lock k=2` |                                                       |
|                                                      | SELECT * FROM t WHERE k=3 FOR UPDATE;     `lock k=3`  |
| SELECT * FROM t WHERE k=3 FOR UPDATE;     `lock k=3` |                                                       |
| `waiting...`                                         | SELECT * FROM t WHERE k=2 FOR UPDATE;  `<- deadlock!` |
| `aborted`  *Error 40001, Txn 1 chosen randomly*      | `unblocked to proceed...`   *Txn 2 won a coin toss!*  |
| COMMIT;                                              | COMMIT;                                               |
| `rollback`                                           | `success`                                             |
| `client retry`                                       |                                                       |



##### Contention Illustration 3.4.  Column families, not blocking concurrent writes on the same key

| Transaction 1 (write)                                        | Transaction 2 (write)                                   |
| ------------------------------------------------------------ | ------------------------------------------------------- |
| -- Special setup for this example:<br/>CREATE TABLE t_mcf (k INT PRIMARY KEY, v1 INT, v2 INT NOT NULL,<br />FAMILY f1 (k, v1), FAMILY f2 (v2));<br/>INSERT INTO t_mcf  values (1,1,1),(2,2,2),(3,3,3); |                                                         |
|                                                              | BEGIN;                                                  |
|                                                              | UPDATE t_mcf SET v2=200002 WHERE k=2;     `-- lock k=2` |
| BEGIN;                                                       |                                                         |
| UPDATE t_mcf SET v1=200001 WHERE k=2;     `-- lock k=2`      |                                                         |
| COMMIT;                                                      |                                                         |
| `success`                                                    | COMMIT;                                                 |
|                                                              | `success`                                               |

> Illustration 3.4 caveats:
>
> - A second col family would have a separate locking scope from the primary in a point lookup if at least one column in the second family is NOT NULL. This is because if all cols in the secondary family are NULL, there will be no key pair, so the PK access is needed to distinguish between “*all columns are NULL*” and “*no row exists*” cases.
> - In other words, if all columns in a second family are NULL-able, a secondary family key lock from a single row SQL by PK will escalate to an entire row.
> - In the current implementation of multi-row scans, CockroachDB don’t constrain locking to a single column family, even if it were possible as an optimization.



### 3.2 Identifying Locking Conflicts

##### Visual Assessment

For *at-a-glance visual assessment*, observe the [Transactions Page in DB Console](https://www.cockroachlabs.com/docs/stable/ui-transactions-page.html). Select the time interval in the header that corresponds to the period of performance issues due to potential contention events. Sort the transactions by "Contention" [time] in descending order (largest contention on top). If *contention time* (the time a transaction spent waiting) is comparable or larger than the *transaction time* (the time a transaction was actively running), the locking conflicts have a significant negative impact on these transactions. Observe the execution *count* of transactions with high contention time. If it is significant, the locking conflicts have a significant negative overall impact on the workload.

For a *visual assessment how lock conflicts have been impacting the workload over time*, observe the [SQL Statement Contention](https://www.cockroachlabs.com/docs/v21.2/ui-sql-dashboard.html#sql-statement-contention) graph. It will allow correlating workload performance issues with "concentration" of lock conflicts over time.

##### Conflict Insights via System Tables

A detailed instrumentation of contention conflicts is a new feature in v22.1 and is available **for locking conflicts only**. A history of contention events is captured in the `crdb_internal.transaction_contention_events`  system table that contains transaction fingerprint IDs for both blocking and waiting transactions, and the contention key. 

For example, to identify lock conflicts details in the last 30 minutes, run the following:

```sql
-- Set the current database to the database in which the potential contention events are being investigated, e.g. 
use mydatabase;

-- In v22.1:
SELECT
    collection_ts
  , contention_duration
  , sw.app_name as waiting_app_name
  , sb.app_name as blocking_app_name
  , waiting_txn_fingerprint_id
  , blocking_txn_fingerprint_id
  , t.name as table_name
  , i.index_name
  , key
FROM (
        SELECT
            collection_ts,
            contention_duration,
            waiting_txn_fingerprint_id,
            blocking_txn_fingerprint_id,
            key_parts[2]::INT AS table_id,
            key_parts[3]::INT AS index_id,
            key
        FROM (
            SELECT
                collection_ts,
                contention_duration,
                waiting_txn_fingerprint_id,
                blocking_txn_fingerprint_id,
                regexp_split_to_array(crdb_internal.pretty_key(contending_key, 0), '/') AS key_parts,
                crdb_internal.pretty_key(contending_key, 0) as key
            FROM
                crdb_internal.transaction_contention_events
            )
        ) e
    JOIN crdb_internal.table_indexes i
         ON e.index_id = i.index_id AND e.table_id = i.descriptor_id
    JOIN crdb_internal.tables t
         ON e.table_id = t.table_id
    LEFT JOIN crdb_internal.cluster_transaction_statistics sw
         ON e.waiting_txn_fingerprint_id = sw.fingerprint_id
    LEFT JOIN crdb_internal.cluster_transaction_statistics sb
         ON e.blocking_txn_fingerprint_id = sb.fingerprint_id
WHERE
    t.database_name = current_database()
AND collection_ts >= NOW() - INTERVAL '30 MINUTES'
;
```

For more insights into lock conflicts in the current database, run the following:

```sql
-- Set the current database to the database in which the potential contention events are being investigated, e.g. 
use mydatabase;

-- Available in release v22.1 and later:
SELECT * FROM crdb_internal.cluster_contended_indexes WHERE database_name = current_database();

-- An equivalent query in v21.2:
SELECT 
    t.database_name
  , t.schema_name
  , t.name
  , i.index_name
  , c.num_contention_events
FROM     crdb_internal.cluster_contention_events c
    JOIN crdb_internal.table_indexes i
         ON c.index_id = i.index_id AND c.table_id = i.descriptor_id
    JOIN crdb_internal.tables t
         ON c.table_id = t.table_id
WHERE
    t.database_name = current_database()
GROUP BY
    t.database_name
  , t.schema_name
  , t.name
  , i.index_name
  , c.num_contention_events
ORDER BY
    c.num_contention_events DESC
;
```

The above query shows the *cumulative number of lock conflicts* `num_contention_events` by table and index, that occurred since the cluster started. This view does not contain timeline information. The information in this table is most useful if the query is executed periodically to track changes in the `num_contention_events` by table and index.

To identify the keys of a lock conflicts, run the following:

```sql
-- Set the current database to the database in which the potential contention events are being investigated, e.g. 
use mydatabase;

-- Available in release v21.2, v22.1 and later:
SELECT 
    t.database_name
  , t.schema_name
  , t.name as table_name
  , i.index_name
  , crdb_internal.pretty_key(c.key, 0) as key
  , sum(count) as key_contention_events
FROM     crdb_internal.cluster_contention_events c
    JOIN crdb_internal.table_indexes i
         ON c.index_id = i.index_id AND c.table_id = i.descriptor_id
    JOIN crdb_internal.tables t
         ON c.table_id = t.table_id
WHERE
    t.database_name = current_database()
GROUP BY
    t.database_name
  , t.schema_name
  , t.name
  , i.index_name
  , c.key
;
```

The output example:

```
  database_name | schema_name |    table_name    |      index_name       |    key     | key_contention_events
----------------+-------------+------------------+-----------------------+------------+------------------------
  postgres      | public      | t_with_conflicts | t_with_conflicts_pkey | /104/1/1/0 |                    10
  postgres      | public      | t_with_conflicts | t_with_conflicts_pkey | /104/1/2/0 |                    14
  postgres      | public      | t_with_conflicts | t_with_conflicts_pkey | /104/1/3/0 |                    11
```

The values in the `key` column are formatted as `/<table id>/<index id>/<key value...>`. `key_contention_events` is a cumulative number of lock conflicts. This information would show the "hot" keys in the cluster.



### 3.3 Remediation of Locking Conflicts

Locking conflicts are a natural artifact when business requirements are calling for concurrent data changes. Realistically, locking conflicts are unavoidable. The locking conflicts, however, are resolved efficiently with regard to the underlying resource utilization. When blocked transactions are waiting on a lock, they are not consuming CPU, disk, or network resources.

Remediation is required when locking conflicts are too numerous, resulting in a significant increase in response time and/or decrease in throughput. Remediation of locking conflicts is typically about giving up some functionality in exchange for a reduction in locking contention, specifically: 



> ✅  **If using [historical](https://www.cockroachlabs.com/docs/v21.2/as-of-system-time.html) queries fits the application design**
>
> - Only if an application can use data that is 5 seconds old or older
> - Primarily benefits read-only transactions
> - Historical queries operate below [closed timestamps](https://www.cockroachlabs.com/docs/v21.2/architecture/transaction-layer#closed-timestamps) and therefore have perfect concurrency characteristics - they never wait on anything and never block anything
> - Best possible performance - served by the nearest replica



> ✅ **If &quot;fail fast&quot; fits the application design**
>
> - &quot;Fail fast&quot; could be a reasonable protective measure in the application to handle "hot update key" situations, for example when an application needs to be able to handle an arbitrary large surge of updates on the same key.
> - The most direct method of "failing fast" is using pessimistic locking with [SELECT FOR UPDATE … NOWAIT](https://www.cockroachlabs.com/docs/stable/select-for-update#wait-policies). It can reduce or prevent failures late in a transaction's life (e.g. at the commit time), by returning an error early in a contention situation if a row cannot be locked immediately.
> - A more "buffered fail fast&quot; approach would be to control the maximum length of a lock wait-queue that requests are willing to enter and wait in, with the cluster setting  [_kv.lock\_table.maximum\_lock\_wait\_queue\_length_](https://github.com/cockroachdb/cockroach/pull/66146). It can provide some level of quality-of-service with a response time predictability in a severe per-key contention. If set to a non-zero value and an existing lock wait-queue is already equal to or exceeding this length, requests will be rejected eagerly instead of entering the queue and waiting.



> ✅ **Columns families can reduce conflicts**
>
> - Conflicts happen at key level.
> - Column families split a single row into multiple keys (KV pairs).
> - Transactions operating on disjoint column families will not conflict (except an edge case when *all* columns in a non-primary column family are NULL-able).



> ✅ **Covered secondary indexes can reduce conflicts (or can aggravate them!)**
>
> - The main reason to use a covered index is to improve SQL performance by avoiding an index join.
> - Since secondary indexes have separate locking scopes, storing additional columns in an index also affects lock contention in both directions - potentially helping or aggravating PK contention!
> - Storing enough columns to avoid an index join can prevent contention on the PK, but storing too many columns can cause index scans to block on locks that may not be necessary.





## 4. Serializable Isolation Conflicts

### 4.1 Serializable Isolation Conflicts Explained

Isolation is a property implemented by a database that defines how the changes made by one transaction become visible to other transactions executing concurrently.

CockroachDB only supports `SERIALIZABLE` isolation.

The SQL specification defines `SERIALIZABLE` as the highest, i.e. the strongest isolation level that guarantees that the effect of SQL transactions that overlap in time is the same as had they not overlapped in time.

 `SERIALIZABLE` isolation guarantees that the following phenomena *will NOT* occur in a transaction:

1. Non-repeatable reads
2. Phantom reads

Legacy DBMS-s commonly use a lock-based concurrency implementation, whereby enforcing serializability requires read and write locks. CockroachDB handles serializability differently.

**CockroachDB serializable isolation implementation highlights:**

- CockroachDB is using a non-lock based optimistic concurrency control, acquiring no read locks.
- Writes proceed if there is no lock conflict.
- To ensure serializability, CockroachDB *validates that the previous reads haven't changed at the commit time*. If reads are non-repeatable, the transaction in conflict is forced to restart.
- When a transaction is forced to restart, CockroachDB will make the best effort to automatically retry it on the server side, provided the conflict can be discovered in the first statement of a transaction. If autoretry is not possible, a transaction returns a `40001` error for a client side retry.



##### Contention Illustration 4.1.  Read-Modify-Write pattern, serialization violation (non-repetitive read protection)

| Transaction 1 (multi-statement)                              | Transaction 2 (multi-statement)                |
| ------------------------------------------------------------ | ---------------------------------------------- |
| BEGIN;                                                       |                                                |
| SELECT * FROM t;                                             |                                                |
|                                                              | BEGIN;                                         |
|                                                              | SELECT * FROM t;                               |
| UPDATE t SET v=222 WHERE k=2;   `-- lock k=2`                |                                                |
|                                                              | UPDATE t SET v=333 WHERE k=3;    `-- lock k=3` |
| COMMIT;                                                      |                                                |
| *`Error 40001: RETRY_SERIALIZABLE - failed preemptive refresh (on commit)`* | COMMIT;                                        |
| `rollback`                                                   | `success`                                      |
| `client retry`                                               |                                                |



##### Contention Illustration 4.2.  Read-Modify-Write pattern, serialization violation (phantom read protection)

| Transaction 1 (multi-statement)                 | Transaction 2 (implicit, single statement) |
| ----------------------------------------------- | ------------------------------------------ |
| BEGIN;                                          |                                            |
| SELECT * FROM t;                                |                                            |
|                                                 | `delete an existing key=2...`              |
| `modify key=2 in the app...`                    | DELETE FROM t WHERE k=2;                   |
| UPDATE t SET v=288 WHERE k=2;     `-- lock k=2` | `success`                                  |
| `waiting...`                                    |                                            |
| *`Error 40001: Write Too Old `*                 |                                            |
| `rollback`                                      |                                            |
| `client retry`                                  |                                            |



##### Contention Illustration 4.3.  Read-Modify-Write pattern, same logic as previous example, improved Implementation 

| Transaction 1 (multi-statement)                              | Transaction 2 (implicit, single statement) |
| ------------------------------------------------------------ | ------------------------------------------ |
| BEGIN;                                                       |                                            |
| SELECT v FROM t WHERE k=2 FOR UPDATE;                        |                                            |
| `k=2 is locked, txn is proceeding under a lock protection...` | `another txn is about to modify key=2...`  |
|                                                              | UPDATE t SET v=299 WHERE k=2;              |
| `modify key=2 in the app...`                                 |                                            |
| UPDATE t SET v=288 WHERE k=2;                                | `waiting...`                               |
| COMMIT;                                                      |                                            |
| `success`                                                    | `success`                                  |

> Takeaways from the illustration 4.3:
>
> - Error 40001 (client side retry) is eliminated!
> - The isolation conflict was effectively traded for a locking conflict. It's a good trade because a locking conflict is resolved by a wait, which is more efficient than a more resource consuming client side retry.
> - But must be careful about limiting the scope of SFU - only lock the key(s) that will be modified. E.g. do not `SELECT * FROM t FOR UPDATE` or SFU becomes counterproductive making the situation worse than the original - locking many keys actually increases the probability of a conflict.



### 4.2 Identifying Serializable Isolation Conflicts

##### Visual Assessment

To assess *how serializable isolation conflicts have been impacting the workload over time*, observe the [Transactions Restarts](https://www.cockroachlabs.com/docs/stable/ui-sql-dashboard.html#transaction-restarts) graph. The isolation conflict errors `40001` that result in a client side retries and the uncertainty interval conflicts that result in an automatic server side retries (see below) are combined into one graph. Hover a pointer over the graph area. The isolation conflict errors are `40001`  and are reported as all lines *other than* "Read Within Uncertainty Interval". Aggregate all `40001` client retry errors to assess the impact of the transaction isolation conflicts on the workload performance over time.

No detailed insights into transaction isolation conflicts are available in system tables via SQL interface.

To be able to troubleshoot `40001` errors expediently, application developers are encouraged to log detailed information about contending transactions and the contention key(s) upon receiving error `40001`.

##### Conflict Insights via System Tables

*Serializable isolation conflicts are currently not captured and therefore not reported via system tables in any `crdb_internal` (only locking conflicts are currently captured and reported via system tables). This is known limitation and CRL is working on resolving this observability gap.*



### 4.3 Remediation of Serializable Isolation Conflicts

Serializable Isolation Conflicts may have the largest negative impact on workload performance among all types of contention because they result in a `40001` client side retry, whereby the work  done by the preceding statements is effectively thrown away. 

Several remediation techniques are available to minimize the impact of isolation conflicts, listed below in the order of a perceived positive impact, most impactful first.



> ✅ **Avoid Isolation "Conflicts by Design"! Follow the best practices for development frameworks**
>
> - Serializable isolation conflicts can be largely avoided!
> - If an application is leveraging a development framework, **follow the best practices for the framework** of choice.
> - Recommended Best Practices Resources:
>   -  [JPA Best Practices Cookbook](https://blog.cloudneutral.se/series/jpa-and-crdb) - a series of blogs that outlines best practices for JPA and Hibernate with CockroachDB
>   - [Spring Annotations](https://blog.cloudneutral.se/spring-annotations-for-cockroachdb) - a guide to using annotations and aspects in Spring to implement a robust transaction scope management.



> ✅ **Avoid Read-Modify-Write pattern whenever possible**
>
> - Read-Modify-Write transaction design pattern is a "magnet" for isolation conflicts
> - It may be possible to avoid reading a column value, modify and write it back by pushing the expression into an SQL update statement, so the transaction becomes [implicit](../system-overview/tech-overview-trsansaction-implicit-explicit.md), which has the best possible concurrency characteristics.
> - For example,  `UPDATE t SET v=v+1 WHERE k=2;` instead of increasing a counter in the application code.




> ✅ **Minimize the scope of reads**
>
> - Minimizing the number of keys in scope for SELECTs reduces the probability of isolation conflicts.
> - Read as little as possible
> - Design the transactions so SELECTs read the minimum number of tuples required to implement the business logic
> - Use indexes, particularly covering indexes
> - Contention is handled at the KV layer - think about conflicts at the level of keys



> ✅ **If conflicts in a multi-statement transaction are unavoidable - use pessimistic locking**
>
> - The client side `40001` retires can be avoided by using pessimistic locking early in a transaction. Acquiring an early lock on a read with  `SELECT … FOR UPDATE` (SFU) eliminates an opportunity for read invalidation (and a costly retry) later.
> - While SFU can prevent serializable conflicts by locking early, this technique can also be counterproductive if the lock has a large scope, thus creating more contention. As a rule of thumb - if using SFU, only lock the key(s) that will be subsequently updated in the same transaction. Otherwise avoid SFU, or it may create more locks than necessary.
> - SFU is effectively a simple-to-implement trade-off of the "expensive" client side `40001` retires for more efficient waits on a lock. While this technique may lessen the impact of isolation conflicts on the workload performance, it does not eliminate the existing contention. It merely replaces the isolation conflicts with locking conflicts that are, as discussed earlier, resolved more efficiently.
> - Only transaction refactoring that eliminates contention by design is a true *solution* for this contention problem.





## 5. Uncertainty Conflicts

### 5.1 Uncertainty Conflicts due to Possible Clock Skew Explained

In CockroachDB, every transaction starts and commits at a timestamp assigned by a CockroachDB node that a client is connected to, called a gateway node. When choosing this timestamp the gateway node does not rely on communication with any other CockroachDB node. The gateway node uses its current time to assign a timestamp to each tuple written by this transaction.

CockroachDB uses multi-version concurrency control (MVCC) - it stores multiple value versions for each tuple, ordered by timestamp. A reader generally returns the latest tuple with a timestamp earlier than the reader's timestamp and ignores tuples with higher timestamps. This is when a potential clock skew needs to be considered.

If a reader's timestamp is assigned by a CockroachDB node with a clock that is behind, it might encounter tuples with higher timestamps that were, in fact, committed before the reader's transaction but their timestamps were assigned by a clock that is ahead.

Tuples with timestamps above the reader’s, but within the `max-offset`, are considered to be ambiguous as to whether they're in the past or the future of the reader. If such an uncertain tuple is encountered, the *reader transaction will generally be forced to restart* at a higher timestamp so all previously uncertain changes are definitely in the past.

For information about handling transactions that had been forced to restart, review the [transaction retries](../system-overview/tech-overview-trsansaction-retires.md) section.



##### Contention Illustration 5.1.  Uncertainty Conflicts

| Transaction 1 (read, implicit)                               | Transaction 2 (write, implicit)                              |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| -- execute in a loop, concurrently with Txn 2                | -- execute in a loop, concurrently with Txn 1                |
| SELECT * FROM t WHERE k=1;                                   | UPDATE t v=111 WHERE k=1;                                    |
| *`Implicit reads will succeed nearly 100% of the time due to automatic retries`* | *`Implicit writes will succeed nearly 100% of the time`*     |
| *`(except an edge case when the returned result set is very large)`* | *` (except an edge case when an internal read that preceeds an UPDATE sees an uncertainty-related restart)`* |
| *`Automatic retries shown in the Retry column in DB Console' Transactions page`* |                                                              |



### 5.2 Identifying Uncertainty Conflicts

##### Visual Assessment

Observe the [Transactions Page in DB Console](https://www.cockroachlabs.com/docs/stable/ui-transactions-page.html). Select the time interval in the header that corresponds to the period of performance issues due to potential contention events. Sort the transactions by "Retries" [count] in descending order (largest number of retires on top).  Observe the execution *count* of transactions with high retry count. If a significant percentage of executed transactions is retired, the uncertainty conflict may have a measurable negative overall impact on the workload.

To asses *how uncertainty conflicts have been impacting the workload over time*, observe the [Transactions Restarts](https://www.cockroachlabs.com/docs/stable/ui-sql-dashboard.html#transaction-restarts) graph. Hover a pointer over the graph area. The automatic transaction restarts are reported as the "Read Within Uncertainty Interval" line. It allows correlating workload performance issues with automatic uncertainty conflicts over time.

No detailed insights into uncertainty conflicts are available in system tables via SQL interface.

##### Conflict Insights via System Tables

*Uncertainty conflicts are currently not captured and therefore not reported via system tables in any `crdb_internal` (only locking conflicts are currently captured and reported via system tables). This is known limitation and CRL is working on resolving this observability gap.*



### 5.3 Remediation of Uncertainty Conflicts

By design, CockroachDB handles an inevitable clock skew by restarting reads when an uncertainty interval conflict occurs. Uncertainty conflicts *can not* be avoided, but they only add a negligible performance overheads when handled efficiently in the same transaction as an automatic retry on the server.

In rare circumstances, when an automatic server side retry is not possible and results in a `40001` client side retry, or when server side retries are numerous, operators need to take actions to reduce a negative influence of uncertainty conflicts on cluster performance. Although these conflicts are unavoidable, their probability can be reduced and the overhead of handling them can be minimized to negligible levels.




> ✅ **Reduce a probability of uncertainty conflicts**
>
> - The cluster's uncertainty window is configurable. Operators can reduce the probability of uncertainty conflicts by reducing the cluster's [--max-offset](https://www.cockroachlabs.com/docs/v21.2/cockroach-start.html#flags)  setting.
> - The `max-offset` setting can be safely reduced from the current default 500ms to 250ms or below, if the clock synchronization relies on robust networking and NTP sources & configuration.
> - Reducing the `max-offset` does not guarantee a measurable improvement. With a smaller uncertainty window, the probability of uncertainty conflicts is lower. If uncertainty retries are observed with a 500 ms  `max-offset`, it's reasonable to expect fewer retries with a 250 ms  `max-offset`.




> ✅ **Use [implicit](../system-overview/tech-overview-trsansaction-implicit-explicit.md) transactions wherever possible**
>
> - In case of a conflict, implicit transactions are retried automatically on the gateway (the node that originated the transaction)
> - Eliminate unnecessary additional client<->gateway network round trips
> - Bring additional performance benefits
>   - Hold locks for less time
>   - Allow some single-range txns to use a streamlined 1-phase commit fast-path (specifically - UPSERTs, which only touch a single row and specify values for all columns.)





## 6. Esoteric Situations that may lead to 40001 Errors

A CockroachDB operator should be aware that:

-  While conflict resolution normally results in just one party of a conflict yielding execution, it's possible that a conflict may result in more than one `40001` victims, even in a 2 transaction conflict.
-  `40001` errors, that are normally associated with contention conflicts, can also occur outside of a contention situation

#### Two transactions in a deadlock can cause each other to restart (both fail with a retry error)

CockroachDB implementation is designed to not let two transactions both cancel each other, so the system would make progress on at least one of them. However a situation where both conflicting transactions fail with a retry error can't be completely ruled out. It would be difficult to make an example of that situation. It's a peculiarity of the code paths.

#### A transaction can get a retry error code spuriously

There are code paths in CockroachDB that result in a retry error outside of a transaction conflict situation.

For example, a lease transfer clears the [timestamp cache](https://www.cockroachlabs.com/docs/stable/architecture/transaction-layer.html#timestamp-cache) which is used to provide serializable guarantees (eliminate non-repetitive and phantom reads). If that happens at a specific point in a transaction, its commit will be rejected with a `40001` error because there is no way to verify the earlier reads are repeatable.

#### Closed Timestamp related errors

Long running transactions may experience `40001` errors due to the [closed timestamp](https://www.cockroachlabs.com/docs/stable/architecture/transaction-layer#closed-timestamps) cluster setting. The closed timestamp is defined by the cluster setting `kv.closed_timestamp.target_duration`, 3 seconds by default. The conditions for this esoteric error are:

- A long-running transaction (duration exceeds `kv.closed_timestamp.target_duration`) T1 reads records. The reads are optimistic, CockroachDB keeps the span information to check at commit time if later writes happened (same logic is used to ensure serializable isolation guarantees).
- A concurrent transaction T2 writes to the table that was read from by T1.
- The long-running transaction T1 performs write(s). The timestamp for the writes gets pushed by the continuously advancing closed timestamp, below which writes cannot be performed.
- T1 finishes and attempts to commit. As part of committing, it has to advance its read timestamp to match its pushed write timestamp. Because of the concurrent write by T2, the T1 reads cannot be advanced, and T1 fails with a retry error.

> Although reported as `RETRY_SERIALIZABLE`, this error, in fact, does *not* occur due to a serializable isolation conflict !

##### Remediation of the Closed Timestamp related retry errors

- `SELECT FOR UPDATE` pessimistic locking in the beginning of a long-running transaction can prevent concurrent writers from invalidating its reads by locking the table. *Carefully evaluate the side effects of this change!* It will keep locks for the duration of the long-running transaction.
- Increase `kv.closed_timestamp.target_duration` so that a long-running transaction doesn't get its write timestamp continuously bumped. *Carefully evaluate the side effects of this change!* It can impact the lag in changefeeds and the latency of follower reads.

##### Contention Illustration 6.1.  Esoteric error due to closed timestamp property

| Transaction 1 (multi-statement)                              | Transaction 2 (write, implicit)                      |
| ------------------------------------------------------------ | ---------------------------------------------------- |
| SHOW CLUSTER SETTING kv.closed_timestamp.target_duration;    |                                                      |
| *`Default closed timestamp period is 3 seconds`*             |                                                      |
| BEGIN;                                                       |                                                      |
| SELECT * FROM t;                                             |                                                      |
|                                                              | UPDATE t SET v=333 WHERE k=3;    `-- lock k=3`       |
| *`Interactively switching windows and typing takes a user more than 3 seconds`* |                                                      |
| UPDATE t SET v=222 WHERE k=2;   `-- lock k=2`                |                                                      |
| COMMIT;                                                      |                                                      |
| *`Error 40001: RETRY_SERIALIZABLE - failed preemptive refresh (on commit)`* | `There is no serializable conflict, why this error?` |
| `rollback`                                                   |                                                      |
| `client retry`                                               |                                                      |



##### Contention Illustration 6.2.  Esoteric error due to closed timestamp property (error eliminated)

| Transaction 1 (multi-statement)                              | Transaction 2 (write, implicit)                |
| ------------------------------------------------------------ | ---------------------------------------------- |
| SET  CLUSTER SETTING kv.closed_timestamp.target_duration='10m'; |                                                |
| BEGIN;                                                       |                                                |
| SELECT * FROM t;                                             |                                                |
|                                                              | UPDATE t SET v=333 WHERE k=3;    `-- lock k=3` |
| UPDATE t SET v=222 WHERE k=2;   `-- lock k=2`                |                                                |
| COMMIT;                                                      |                                                |
| `success`                                                    |                                                |





## 7. FAQs

**Q:** *CRDB workload contention vernacular distinguishes the 3 types of conflicts - locking, isolation, and uncertainty. Aren't isolation conflicts the same as locking conflicts? Why is it helpful to consider these 2 conflict types separately?*

**A:**  Locking and isolation conflicts differ in the following essential ways:

- Locking conflicts (outside of a deadlock condition) are resolved through waits. Isolation conflicts are resolved by forcing one of the transactions to restart.
- Locking conflicts are handled within the conflicting statements. Isolation conflicts can be resolved at commit time, after a conflicting statement was successfully executed.
- In the real world, the locking conflicts are predominantly an attribute of business requirements calling for concurrent data changes, unavoidable. Isolation conflicts can often be eliminated with serializable-isolation-aware transaction implementation.
- The available instrumentation means for locking and isolation conflicts are different.
- The main remediation choices for locking and isolation conflicts are different. It’s helpful to differentiate for clarity of remediation guidance.



**Q:** *CRDB workload contention vernacular distinguishes the 3 types of conflicts - locking, isolation, and uncertainty. Uncertainty conflicts are intuitively different from other conflict types. Is it even a "conflict"?  In theory, uncertainty interval related restarts can occur even if a user only ever runs one transaction at a time.*

**A:**  Referring to uncertainty related restarts as "conflicts" is a useful since they only arise when there are near-concurrent reads and writes on the same keys. A "conflict" is an attribute of incompatibility. It's broader than just a locking conflict, for example. Since a reader would not be forced to restart if there were no writes, these reads and writes are not compatible under the circumstances, and therefore are conflicted. This conflict is a direct consequence of a contention on a key (which we agree is a "conflict"), but since not directly related to locking, it's useful to view them as a different type of "conflict".
Also note that the consequence of this type of incompatibility is essentially the same as in the case of an isolation conflict. A transaction in this type of conflict is forced to restart and may return the same error `40001`. If isolation related incompatibility is a "conflict" that results in a `40001` error, it would be confusing to users if the uncertainty related incompatibility is not referred to as a "conflict".



## Resources

- [CockroachDB Transaction Layer](https://www.cockroachlabs.com/docs/stable/architecture/transaction-layer.html)
- [Troubleshooting Overview](https://www.cockroachlabs.com/docs/stable/troubleshooting-overview.html)
- [Troubleshoot Statement Behavior](https://www.cockroachlabs.com/docs/stable/query-behavior-troubleshooting.html)
- [Blog: Spring Annotations for CockroachDB](https://blog.cloudneutral.se/spring-annotations-for-cockroachdb)

