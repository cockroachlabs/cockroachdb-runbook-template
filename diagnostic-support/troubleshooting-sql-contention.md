**< First Draft >**



# Troubleshooting: Workload Contention



## About Workload Contention

This section includes the *background information, examples, troubleshooting technique and remediation ideas* related to *SQL workload* contention. It provides a CockroachDB practitioner with essential knowledge and remediation points for possible contention scenarios, including:

1. Locking conflicts
2. Transaction isolation conflicts
3. Uncertainty conflicts due to a possible clock skew

To resolve any transaction contention, a database takes one of the two available actions:

- either allow the blocked transactions to *wait*,
- or *force* one of the conflicting transactions to *re-start*

Sections below detail how CockroachDB handles the most common concurrency conflicts with either of the methods above.

Contention scenario illustrations in this section include both [implicit and explicit transactions](../system-overview/tech-overview-trsansaction-implicit-explicit.md). If a sequence of SQL statements starts with a `BEGIN`, it denotes an explicit transaction. Otherwise a transaction is implicit, single-statement.




> ✅  **Minimize contention by design!**
> - Contention is not intrinsically "bad". Some kinds of contention is unavoidable, for example when it represents a reality of business requirements. Yet there is also preventable contention that can be eliminated entirely with a considerate schema and transaction design.
> - Databases are purpose-built to manage concurrent access to data. This section provides the insights into how CockroachDB manages contention.
> - Contention needs to be addressed only when it manifests itself as "bad performance", with or without transaction errors such as `40001`.
> - As long as an application requires concurrent access to data, all contention can't be eliminated. There are techniques to *minimize* the performance penalties due to contention and they are are covered in Remediation chapter. Yet the most effective *solution* to a "contention problem" will always be a data model (schema) and transaction logic design that minimizes the opportunities for contention in the first place.



Related topic: For guidance about handling hardware contention for the underlying shared computing resources, review the [Troubleshooting Hardware Resource Contention](troubleshooting-hardware-contention.md).



## 1. Locking Conflicts Explained

All locking in CockroachDB is implemented in the KV layer. The locking granularity is a key. At the SQL layer, a logical row of a relational table can be represented by one or more KV pairs, according to the number of column families defined by a table DDL. Therefore at the SQL layer, the locking in CockroachDB is more granular than row-level. However, in this section, unless specifically noted, we assume the *row-level locking*, which corresponds to the default single column family per table.

CockroachDB implements only one kind of lock - an *exclusive write lock* to manage a concurrent access by a key. Since there is only one kind of lock, we will be referring to it as just "lock".

**CockroachDB locking implementation highlights:**

- Writes acquire locks
- `SELECTs ... FOR UPDATE` acquire locks, same as writes
- Writes block reads and writes [from other transactions]
- Reads do not acquire locks
- Reads do not block reads or writes [from other transactions]
- All blocked statements are waiting indefinitely in the same [queue](https://www.cockroachlabs.com/docs/v21.2/architecture/transaction-layer.html#txnwaitqueue) until the blocking transaction releases the lock (aside from situations when a waiting transaction is forcefully disrupted externally, for example by a timeout of as a result of a closed connection)
- Locks are released when the holding transaction is closed (committed or rolled back)



#### Deadlock Detection and Resolution

Write-write conflicts may lead to deadlocks when different transactions acquire locks in different orders and then wait for each other to release the locks.

CockroachDB employs a distributed deadlock-detection algorithm that analyzes the [wait queue](https://www.cockroachlabs.com/docs/stable/architecture/transaction-layer.html#txnwaitqueue), which tracks the transactions that are blocked and transactions they are blocked by. When a closed loop is detected, one transaction from a cycle of waiters is forced to rollback and must be [retried](../system-overview/tech-overview-trsansaction-retires.md).

When transactions in a deadlock have the same priority, which transaction is aborted can not be predicted. If the priorities are different, the transaction with a lower priority is aborted.



#### Locking Contention Illustrations

The common types of contention scenarios are illustrated with easy-to-follow SQL examples. They may provide CockroachDB cluster operators with an additional method of learning the insights of contention handling, with SQL "scratch pad" experiments.

##### Setup for All Contention Illustrations

| -- One time setup                          |
| ------------------------------------------ |
| DROP   TABLE IF EXISTS  t;                 |
| CREATE TABLE t (k INT PRIMARY KEY, v INT); |
| INSERT INTO t VALUES (1,1),(2,2),(3,3);    |



##### [No] Contention Illustration 1.1.  Reads are not blocking reads or writes

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



##### Contention Illustration 1.2.  Writes are blocking reads and writes

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



##### Contention Illustration 1.3.  Deadlock (write-write closed loop conflict)

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



##### Contention Illustration 1.4.  Column families, not blocking concurrent writes on the same key

| Transaction 1 (write)                                        | Transaction 2 (write)                                |
| ------------------------------------------------------------ | ---------------------------------------------------- |
| -- Special setup for this example:<br/>CREATE TABLE t_mcf (k INT PRIMARY KEY, v1 INT, v2 INT NOT NULL,<br />FAMILY f1 (k, v1), FAMILY f2 (v2));<br/>INSERT INTO t_mcf  values (1,1,1),(2,2,2),(3,3,3); |                                                      |
|                                                              | BEGIN;                                               |
|                                                              | UPDATE t_mcf SET v2=200002 WHERE k=2;     `lock k=2` |
| BEGIN;                                                       |                                                      |
| UPDATE t_mcf SET v1=200001 WHERE k=2;     `lock k=2`         |                                                      |
| COMMIT;                                                      |                                                      |
| `success`                                                    | COMMIT;                                              |
|                                                              | `success`                                            |





## 2. Transaction Isolation Conflicts Explained

Isolation is a property implemented by a database that defines how the changes made by one transaction become visible to other transactions executing concurrently.

CockroachDB only supports `SERIALIZABLE` isolation.

The leveling language of the SQL specification defines `SERIALIZABLE` as the highest, i.e. the strongest isolation level. It guarantees that the following phenomena *will NOT* occur in a transaction:

1. Non-repeatable reads
2. Phantom reads

Legacy DBMS-s commonly use a lock-based concurrency implementation, whereby enforcing serializability requires read and write locks. CockroachDB handles serializability differently.

**CockroachDB serializable isolation implementation highlights:**

- CockroachDB is using a non-lock based optimistic concurrency control, acquiring no read locks.
- Writes proceed if there is no lock conflict, regardless of the reader's state in any transaction.
- To ensure serializability, CockroachDB *validates that the previous reads haven't changed at the commit time*. If reads are non-repeatable, the transaction in conflict is forced to restart.
- A transaction that was forced to restart, will return error `40001` to the client to be retried.



##### Contention Illustration 2.1.  Classic Serialization violation (non-repetitive reads protection)

| Transaction 1 (multi-statement)                        | Transaction 2 (write, implicit)           |
| ------------------------------------------------------ | ----------------------------------------- |
| BEGIN;                                                 |                                           |
| SELECT * FROM t WHERE v < 30000;                       |                                           |
|                                                        | UPDATE t SET v=3 WHERE k=3;    `lock k=3` |
| UPDATE t SET v=2 WHERE k=2;   `lock k=2`               |                                           |
| COMMIT;                                                |                                           |
| *`Error 40001: failed preemptive refresh (on commit)`* |                                           |
| `rollback`                                             |                                           |
| `client retry`                                         |                                           |



##### Contention Illustration 2.2.  Read-Modify-Write pattern, 2 identical contending transactions, different timing

| Transaction 1 (multi-statement)                | Transaction 2 (multi-statement)                |
| ---------------------------------------------- | ---------------------------------------------- |
| BEGIN;                                         | BEGIN;                                         |
| SELECT * FROM t;                               | SELECT * FROM t;                               |
| `modify v=28888 in the app...`                 | `modify v=29999 in the app...`                 |
|                                                | UPDATE t SET v=29999 WHERE k=2;     `lock k=2` |
| UPDATE t SET v=28888 WHERE k=2;     `lock k=2` |                                                |
| `waiting...`                                   | COMMIT;                                        |
| *`Error 40001: Write Too Old `*                | `success`                                      |
| `rollback`                                     |                                                |
| `client retry`                                 |                                                |



##### Contention Illustration 2.3.  Read-Modify-Write pattern - 2.2 Illustration, Improved Implementation 

| Transaction 1 (multi-statement)        | Transaction 2 (multi-statement)                    |
| -------------------------------------- | -------------------------------------------------- |
| BEGIN;                                 | BEGIN;                                             |
|                                        | SELECT v FROM t WHERE k=2 FOR UPDATE;   `lock k=2` |
| SELECT v FROM t WHERE k=2 FOR UPDATE;  | `modify v=29999 in the app...`                     |
| `waiting...`                           | UPDATE t SET v=29999 WHERE k=2;                    |
|                                        | COMMIT;                                            |
| `unblocked to proceed...`   `lock k=2` | `success`                                          |
| `modify v=28888 in the app...`         |                                                    |
| UPDATE t SET v=28888 WHERE k=2;        |                                                    |
| COMMIT;                                |                                                    |
| `success`                              |                                                    |





## 3. Uncertainty Conflicts due to Possible Clock Skew Explained

In CockroachDB, every transaction starts and commits at a timestamp assigned by a CockroachDB node that a client is connected to, called a gateway node. When choosing this timestamp the gateway node does not rely on synchronization with any other CockroachDB node. The gateway node uses its current time to assign a timestamp to each tuple written by this transaction.

CockroachDB uses multi-version concurrency control (MVCC) - it stores multiple value versions for each tuple, ordered by timestamp. A reader generally returns the latest tuple with a timestamp earlier than the reader's timestamp and can could ignore tuples with higher timestamps. This is when a potential clock skew needs to be considered.

If a reader's timestamp is assigned by a CockroachDB node with a clock that is behind, it might encounter tuples with higher timestamps that were, in fact, committed before the reader's transaction but their timestamps were assigned by a clock that is ahead.

Tuples with timestamps above the reader’s, but within the `max-offset`, are considered to be ambiguous as to whether they're in the past or the future of the reader. If such an uncertain tuple is encountered, the *reader transaction will generally be forced to restart* at a higher timestamp so all previously uncertain changes are definitely in the past.

For information about handling transactions that had been forced to restart, review the [transaction retries](../system-overview/tech-overview-trsansaction-retires.md) section.



##### Contention Illustration 3.1.  Uncertainty Conflicts

| Transaction 1 (read, implicit)                               | Transaction 2 (write, implicit)               |
| ------------------------------------------------------------ | --------------------------------------------- |
| -- execute in a loop, concurrently with Txn 2                | -- execute in a loop, concurrently with Txn 1 |
| SELECT * FROM t WHERE k=1;                                   | UPDATE t v=111 WHERE k=1;                     |
| *`Implict reads will practically always succeed`*            | *`Writes will always succeed`*                |
| *`Occasionally reads will take longer due to automatic retries`* |                                               |
| *`Shown in the Retry column in DB Console Transactions page`* |                                               |





## 4. Esoteric Situations that may lead to 40001 Retry Errors

A CockroachDB operator should be aware that:

-  While conflict resolution normally results in just one party of a conflict yielding execution, it's possible that a conflict may result in more than one `40001` victim, even in a 2 transaction conflict.
-  `40001` errors, that are normally associated with contention conflicts, can also occur outside of a contention situation

#### Two transactions in a deadlock can cause each other to restart (both fail with a retry error)

CockroachDB implementation is designed to not let two transactions both cancel each other, so the system would make progress on at least one of them. However a situation where both conflicting transactions fail with a retry error can't be completely rule it out. It would be difficult to make an example of that situation. It's a peculiarity of the code paths.

#### A transaction can get a retry error code spuriously

There are code paths in CockroachDB that result in a retry error outside of a transaction conflict situation.

For example, a lease transfer clears the [timestamp cache](https://www.cockroachlabs.com/docs/stable/architecture/transaction-layer.html#timestamp-cache) which is used to provide serializable guarantees (eliminate non-repetitive and phantom reads). If that happens at a specific point in a transaction, its commit will be rejected with a `40001` error because there is no way to verify the earlier reads are repeatable.





## 5. Contention Aggravating Factors

One of the main factors making a negative impact of contention on the workload more pronounced is a "loose" multi-statement transaction design, when an application is doing a measurable amount non-database work while in an open transaction. A "loose" transaction may be holding a lock longer than absolutely necessary, thus increasing the wait times of transactions that are blocked on that lock. Or increasing the time between executions of individual statements in a transaction, thus increasing a probability of isolation conflicts.

For a high level ***visual assessment***, compare the [Open SQL Transactions](https://www.cockroachlabs.com/docs/stable/ui-sql-dashboard.html#open-sql-transactions) and the [Active SQL Statements](https://www.cockroachlabs.com/docs/stable/ui-sql-dashboard.html#active-sql-statements) graphs in the the [SQL Dashboard](https://www.cockroachlabs.com/docs/stable/ui-sql-dashboard.html) side by side. If the number of active statements track the number of open transactions, it means each open transaction is executing a statement, and that would suggest a good transaction logic implementation. Conversely, if the number of active statements lags the number of open transactions, it would suggest that some open transactions are doing non-database work while keeping a transaction open, which should probably be investigated.





## 6. Contention Troubleshooting Steps

A contention investigation is practically always prompted by a performance complain, with or without `40001` error observations. The main concern is commonly a worsened response time, caused by the two attributes of contention - waits and/or retries - directly contributing to its increase.

Troubleshooting of performance issues caused by workload contention follows this path:

- Step 1. Confirm that the root cause of the performance issues is predominantly the workload contention and not various other possible causes
- Step 2. If contention is confirmed to be the principle issue, identify what specific type of conflicts (locking, isolation, or uncertainty) exist in the workload
- Step 3. Using instrumentation available for each type of conflict, identify the contending transactions and the contention point (key). With that information available, a cluster operator, in collaboration with an application developer, can act and resolve the contention issues.



### Step 1. Determine if the cluster issues are due predominantly the workload contention

Before focusing on contention troubleshooting, confirm that the current issues are not principally caused by other culprits from the [list of the most common problems](../most-common-problems/README.md) where contention is only one of (#4). There are 5 major possible causes, other than contention, to rule out first. Contention is practically always present in a cluster, yet contention may not necessarily be the first problem to focus on when resolving a cluster performance issue.

An operator needs to assess the current level of contention and its impact on the cluster performance vs other possible causes. A hot node starved of CPU by workload overload would be, for example, the first issue to resolve even if a significant workload contention is confirmed.



### Step 2. Identify the types of conflicts in the workload

Assessing the degree of contention in a cluster is aligned with determining the specific types of conflicts that make up the contention. It is effectively one task, where the overall contention level is an aggregate of locking conflicts, isolation conflicts and uncertainty conflicts.

> 👍 **Best Practice: Set the application name on every client connection**
>
> - Setting the [application name](https://www.cockroachlabs.com/docs/v22.1/map-sql-activity-to-app.html) on every application connection makes troubleshooting of contention much easier.
> - The application name connection tag is captured in the system tables and in DB Console, identifying the application components that issued conflicting transactions.



> ✅ **The guidance in this section is for the two most recent major CockroachDB versions at the time of writing - v22.1 and v21.2.**
>



#### Identifying Locking Conflicts

For a ***quick visual assessment***, observe the [Transactions Page in DB Console](https://www.cockroachlabs.com/docs/stable/ui-transactions-page.html). Select the time interval in the header that corresponds to the period of performance issues due to potential contention events. Sort the transactions by "Contention" [time] in descending order (largest contention on top). If *contention time* (the time a transaction spent waiting) is comparable or larger than the *transaction time* (the time a transaction was actively running), the locking conflicts have a significant negative impact on these transactions. Observe the execution *count* of transactions with high contention time. If it is significant, the locking conflict have a significant negative overall impact on the workload.

For a ***visual assessment how lock conflicts have been impacting the workload over time***, observe the [SQL Statement Contention](https://www.cockroachlabs.com/docs/v21.2/ui-sql-dashboard.html#sql-statement-contention) graph. It will allow to correlate workload performance issues with "concentration" of lock conflicts over time.

For more **insights into lock conflicts**, open a session with the [CockroachDB interactive SQL utility](https://www.cockroachlabs.com/docs/stable/cockroach-sql.html) and type:

```sql
-- Set the current database to the database in which the potential contention events are being investigated, e.g. 
use mydatabase;

-- In v22.1:
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

To identify the keys of a lock conflicts, run the following query against CockroachDB system tables:

```sql
-- Set the current database to the database in which the potential contention events are being investigated, e.g. 
use postgres;

-- In v22.1 and v21.2:
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



#### Identifying Transaction Isolation Conflicts

For a ***visual assessment how transaction isolation conflicts have been impacting the workload over time***, observe the [Transactions Restarts](https://www.cockroachlabs.com/docs/stable/ui-sql-dashboard.html#transaction-restarts) graph. The isolation conflict errors `40001` that result in a client side retries and the uncertainty interval conflicts that result in an automatic server side retries (see below) are combined into one graph. Hover a pointer over the graph area. The isolation conflict errors `40001`  and are reported as all lines *other than* "Read Within Uncertainty Interval". Aggregate all `40001` client retry errors to assess the impact of the transaction isolation conflicts on the the workload performance over time.

No detailed insights into transaction isolation conflicts are available in system tables via SQL interface.

To be able to troubleshoot `40001` errors expediently, application developers are encouraged to log a detailed information about contending transactions and the contention key(s) upon receiving error `40001`.



#### Identifying Uncertainty Interval Conflicts

For a ***quick visual assessment***, observe the [Transactions Page in DB Console](https://www.cockroachlabs.com/docs/stable/ui-transactions-page.html). Select the time interval in the header that corresponds to the period of performance issues due to potential contention events. Sort the transactions by "Retries" [count] in descending order (largest number of retires on top).  Observe the execution *count* of transactions with high retry count. If it is significant percentage of executed transactions is retired, the uncertainty conflict may have a measurable negative overall impact on the workload.

For a ***visual assessment how uncertainty conflicts have been impacting the workload over time***, observe the [Transactions Restarts](https://www.cockroachlabs.com/docs/stable/ui-sql-dashboard.html#transaction-restarts) graph. Hover a pointer over the graph area. The automatic transaction restarts are reported as "Read Within Uncertainty Interval" line. It allows to correlate workload performance issues with automatic uncertainty conflicts over time.

No detailed insights into uncertainty conflicts are available in system tables via SQL interface.



### Step 3. Identify the contending transactions and the point (key) of contention

Instrumentation that provides detailed actionable insights into the workload contention is essential to enable a cluster operator to swiftly act on performance issues, if caused primarily by contention.

Actionable details include:

- the definitions of 2 contending transactions at the level of statement fingerprint (normalized), and
- the contention point (key)

The detailed instrumentation of contention conflicts is a new feature in v22.1 and is available for locking conflicts only. A history of contention events is captured in the `crdb_internal.transaction_contention_events`  system table that contains transaction fingerprint IDs for both blocking and
waiting transactions, and the contention key.  [TODO: (1) discover a way to decode the txn fingerprint IDs and report transactions' statements and (2) joining with `crdb_internal.transaction_statistics` vs `crdb_internal.cluster_transaction_statistics`.]

For example, to identify lock conflicts details in the last 30 minutes, run the following query:

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





## 7. Contention Remediation

### Remediation of Locking Conflicts

Locking conflicts are a natural artifact when business requirements are calling for concurrent data changes. Realistically, locking conflicts are unavoidable. The locking conflicts, however, are resolved efficiently with regard to the underlying resource utilization. When blocked transactions are waiting on a lock, they are not consuming CPU, disk, or network resources.

Remediation is required when locking conflicts are too numerous, resulting in a significant increase in response time and/or decrease in throughput. Remediation of locking conflicts is typically about giving up some functionality in exchange for a reduction in locking contention, specifically: 



> ✅  **If using [historical](https://www.cockroachlabs.com/docs/v21.2/as-of-system-time.html) queries fits the application design**
>
> - Only if an application can use data that is 5 seconds old or older
> - Primarily benefits read-only transactions
> - Historical queries operate below [closed timestamps](https://www.cockroachlabs.com/docs/v21.2/architecture/transaction-layer#closed-timestamps) and therfore have perfect concurrency characteristics - they never wait on anything and never block anything
> - Best possible performance - served by the nearest replica



> ✅ **If &quot;fail fast&quot; fits the application design**
>
> - &quot;Fail fast&quot; could be a reasonable protective measure in the application to handle "hot update key" situations, for example when an application needs to be able to handle an arbitrary large surge of updates on the same key.
> - The most direct method of "failing fast" is using pessimistic locking with [SELECT FOR UPDATE … NOWAIT](https://www.cockroachlabs.com/docs/stable/select-for-update#wait-policies). It can reduce or prevent failures late in a transaction's life (e.g. at the commit time), by returning an error early in a contention situation if a row cannot be locked immediately.
> - A more "buffered fail fast&quot; approach would be to control the maximum length of a lock wait-queue that requests are willing to enter and wait in, with the cluster setting  [_kv.lock\_table.maximum\_lock\_wait\_queue\_length_](https://github.com/cockroachdb/cockroach/pull/66146). It can provide some level of quality-of-service with a response time predictability in a severe per-key contention. If set to a non-zero value and an existing lock wait-queue is already equal to or exceeding this length, requests will be rejected eagerly instead of entering the queue and waiting.



> ✅ **Columns families can reduce conflicts**
>
> - Conflicts happen at key level
> - Column families split a single row into multiple keys (KV pairs)
> - Transactions operating on disjoint column families will not conflict [TODO: the correct statement is more nuanced. All columns in the column families that doesn't include the PK must be NOT NULL for this statement to be true]



### Remediation of Transaction Isolation Conflicts

Transaction Isolation Conflicts may have the largest negative impact on workload performance among all types of contention because they result in a `40001` client side retry, whereby the work  done by the preceding transaction is effectively thrown away. [TODO: there are nuances wrt the previous sentence; a retry logic [can be optimized](https://www.cockroachlabs.com/docs/v21.2/savepoint.html#savepoints-for-client-side-transaction-retries) to not throw away the entire transaction work. Unclear if this is worth emphasizing.]

Several remediation techniques are available to minimize the impact of isolation conflicts, listed below in the order of a perceived positive impact, most impactful first.

##### Avoid Isolation Conflicts by Design

In a large number of situations, isolation conflicts could be avoided:

- If an application is leveraging a development framework, follow the best practices for that framework. 
- If the application's custom data access layer implements the transaction logic, pay due attention to serializable isolation realities.



> ✅ **Follow the best practices for data for development frameworks**
>
> - **<Under construction>**
> - For Spring:
>   - Use [Spring Annotations](https://blog.cloudneutral.se/spring-annotations-for-cockroachdb) to bring clarity to transaction management
>   - Eagerly fetching too much (excessive cross joins)
>   - Lazy fetching too little (excessive amounts of queries)



> ✅ **Avoid Read-Modify-Write pattern whenever possible**
>
> - Read-Modify-Write transaction design pattern is a "magnet" for isolation conflicts
> - It may be possible to avoid reading a column value, modify and write it back by pushing the expression into an SQL update statement, so the transaction becomes [implicit](../system-overview/tech-overview-trsansaction-implicit-explicit.md), which has the best possible concurrency characteristics.
> - For example,  `UPDATE t SET v=v+1 WHERE k=2;` instead of increasing a counter in the application code.



##### Use pessimistic locking

The client side `40001` retires can be avoided by using pessimistic locking early in a transaction. This approach is effectively a simple-to-implement trade-off of the "expensive" client side `40001` retires for more efficient waits on a lock. While this technique lessens the impact of isolation conflict on the workload performance, it does not eliminate the existing contention. It merely replaces the isolation conflicts with locking conflicts that are, as discussed earlier, resolved more efficiently. Users are encouraged to scrutinize the logical design beyond switching to pessimistic locking, since the isolation conflicts can often be avoided by transaction refactoring, which would be an ultimate solution.



> ✅ **If conflicts in a multi-statement transaction are unavoidable - use pessimistic locking**
>
> - Use `SELECT … FOR UPDATE` to conflict earlier in the transaction.
> - Blocking earlier on a read eliminates an opportunity for a read invalidation later, which would  result in a costly retry.
> - Note: `SELECT … FOR UPDATE` helps by letting you trade the costly client-side retries for more efficient waits, yet it does *not solve* the contention problem. Only transaction refactoring that eliminates contention by design is a *solution* for this contention problem.



##### Minimize the scope of reads

Minimizing the number of keys in scope for SELECTs reduces the probability of isolation conflicts.




> ✅ **Read as little as possible**
>
> - Design the transactions so SELECTs read the minimum number of tuples required to implement the business logic
> - Use indexes, particularly covering indexes
> - Contention is handled at the KV layer - think about conflicts at the level of keys



### Remediation of Uncertainty Interval Conflicts

By design, CockroachDB handles an inevitable clock skew by restarting reads when an uncertainty interval conflict occurs. Uncertainty conflicts *can not* be avoided, but they only add a negligible performance overheads when handled efficiently in the same transaction as an automatic retry on the server.

In rare circumstances, when an automatic server side retry is not possible and results in a `40001` client side retry, or when server side retries are numerous, operators need to take actions to reduce a negative influence of uncertainty conflicts on cluster performance. Although these conflicts are unavoidable, their probability can be reduced and the overhead of handling them can be minimized to negligible levels.




> ✅ **Reduce a probability of uncertainty conflicts **
>
> - The cluster's uncertainty window is configurable. Operators can reduce a probability of uncertainty conflicts by reducing the cluster's [--max-offset](https://www.cockroachlabs.com/docs/v21.2/cockroach-start.html#flags)  setting.
> - The `max-offset` setting can be safely reduced from the current default 500ms to 250ms or below, if the clock synchronization relies on robust networking and NTP sources & configuration.
> - Reducing the `max-offset` does not guarantee a measurable improvement. With a smaller uncertainty window, a probability of uncertainty conflicts is lower. If uncertainty retries are observed with a 500 ms  `max-offset`, it's reasonable to expect fewer retries with a 250 ms  `max-offset`.




> ✅ **Use [implicit](../system-overview/tech-overview-trsansaction-implicit-explicit.md) transactions wherever possible**
>
> - In case of a conflict, implicit transactions are retried automatically on the gateway (the node that originated the transaction)
> - Eliminate unnecessary additional client<->gateway network round trips
> - Bring additional performance benefits
>   - Hold locks for less time
>   - Allow single-range txns to use a streamlined 1-phase commit fast-path





## Useful Resources

- [Troubleshooting Overview](https://www.cockroachlabs.com/docs/stable/troubleshooting-overview.html)
- [CockroachDB Transaction Layer](https://www.cockroachlabs.com/docs/v21.2/architecture/transaction-layer.html)
- [Troubleshoot Cluster Setup](https://www.cockroachlabs.com/docs/stable/cluster-setup-troubleshooting.html)
- [Troubleshoot Statement Behavior](https://www.cockroachlabs.com/docs/stable/query-behavior-troubleshooting.html)
- [Blog: Spring Annotations for CockroachDB](https://blog.cloudneutral.se/spring-annotations-for-cockroachdb)
