**< UNDER CONSTRUCTION >**



# Troubleshooting: Workload Contention



### About Workload Contention

This section includes the *background information, examples, troubleshooting technique and remediation ideas* related to *SQL workload* contention matters. It provides a CockroachDB practitioner with essential knowledge and remediation points about possible workload contention scenarios, including:

1. Statement (locking) conflicts
2. Transaction isolation conflicts
3. Uncertainty conflicts due to a possible clock skew

To resolve any transaction contention, a database can take one of the two available actions:

- either allow the blocked transactions to *wait*, or
- to *force* one of the conflicting transactions to *re-start*

Sections below detail how CockroachDB handles the most common concurrency conflicts with either of the methods above.

Contention scenario illustrations in this section include both [implicit and explicit transactions](../system-overview/tech-overview-trsansaction-implicit-explicit.md). If a sequence of SQL statements starts with a `BEGIN`, it denotes an explicit transaction. Otherwise a transaction is implicit, single-statement.




> 👍 **Best Practice: Design the schema and transactions that avoid contention conflicts-by-design**
> - Contention always manifests itself as "bad performance"
> - Best practices are available to alleviate the performance penalties due to contention, yet the only solution to a "contention problem" is a design that eliminates or greatly reduces the opportunities for contention.



Related topic: For guidance about handling hardware contention for the underlying shared computing resources, review the [Troubleshooting Hardware Resource Contention](troubleshooting-hardware-contention.md).



### 1. Locking Conflicts Explained

All locking in CockroachDB is implemented in the KV layer. The locking granularity is a key. At the SQL layer, a tuple can be represented by one or more KV pairs, according to the number of column families defined for a SQL table. Therefore at the SQL layer, the locking in CockroachDB is more granular than row-level. However, in this section we assume the row-level locking, which corresponds to the default single column family per table.

CockroachDB implements only one kind of lock - an *exclusive write lock* to manage concurrent access by a primary key. Since there is only one kind, we will be referring to it as just "lock".

CockroachDB locking implementation highlights:

- Writes acquire locks
- `SELECT FOR UPDATE` acquire locks
- Reads do not acquire locks
- Reads do not block reads or writes [from other transactions]
- Writes block reads and writes [from other transactions]
- All blocked statements are waiting indefinitely in the same [queue](https://www.cockroachlabs.com/docs/v21.2/architecture/transaction-layer.html#txnwaitqueue) until the blocking transaction releases the lock (aside from situations when a waiting transaction is forcefully disrupted externally)
- Locks are released when the holding transaction is closed (committed or rolled back)



#### Deadlock Detection and Resolution

Write-write conflicts may lead to deadlocks when different transactions acquire locks in different orders and then wait for each other to release the locks.

CockroachDB employs a distributed deadlock-detection algorithm that analyzes the [wait queue](https://www.cockroachlabs.com/docs/stable/architecture/transaction-layer.html#txnwaitqueue), which tracks the transactions that are blocked and transactions they are blocked by. When a closed loop is detected, one transaction from a cycle of waiters is forced to rollback and must be [retried](../system-overview/tech-overview-trsansaction-retires.md).

When transactions in a deadlock have the same priority, which transaction is aborted can not be predicted. If the priorities are different, the transaction with a lower priority is aborted.



##### Setup for All Contention Illustrations

| -- One time setup                          |
| ------------------------------------------ |
| DROP   TABLE IF EXISTS  t;                 |
| CREATE TABLE t (k INT PRIMARY KEY, v INT); |
| INSERT INTO t values (1,1),(2,2),(3,3);    |



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

| Transaction 1 (SFU/write)                            | Transaction 2 (SFU/write)                                 |
| ---------------------------------------------------- | --------------------------------------------------------- |
| BEGIN;                                               | BEGIN;                                                    |
| SELECT * FROM t WHERE k=2 FOR UPDATE;     `lock k=2` |                                                           |
|                                                      | SELECT * FROM t WHERE k=3 FOR UPDATE;     `lock k=3`      |
| SELECT * FROM t WHERE k=3 FOR UPDATE;     `lock k=3` |                                                           |
| `waiting...`                                         | SELECT * FROM t WHERE k=2 FOR UPDATE;  `<- deadlock!`     |
| `aborted`  *Error 40001, Txn 1 chosen randomly*      | `unblocked to proceed...`   *Txn 2 could've gotten 40001* |
| COMMIT;                                              | COMMIT;                                                   |
| `rollback`                                           | `success`                                                 |
| `client retry`                                       |                                                           |



##### Contention Illustration 1.4.  Column families, not blocking concurrent writes on the same key

| Transaction 1 (write)      | Transaction 2 (write)      |
| -------------------------- | -------------------------- |
| ***`UNDER CONSTRUCTION`*** | ***`UNDER CONSTRUCTION`*** |
| BEGIN;                     | BEGIN;                     |
| COMMIT;                    | COMMIT;                    |
| `success`                  | `success`                  |



> ✅ **Remedy: If conflicts in a multi-statement transaction is unavoidable - conflict early**
> - Use `SELECT … FOR UPDATE` to conflicts earlier in the transaction
> - Block earlier, before reads that could be invalidated later and result in a costly retry
> - Note: `SELECT … FOR UPDATE` can really help by letting you trade the costly client-side retries for more efficient waits, but it does not *solve* the contention problem. Only transaction refactoring that eliminates contention by design is a *solution* for the contention problem.



> ✅ **Remedy: Columns families can eliminate conflicts**
> - Contention happens at the key level
> - Column families split a single row into multiple keys (KV pairs)
> - Transactions operating on disjoint column families will not conflict



> ✅ **Remedy: Use [historical](https://www.cockroachlabs.com/docs/v21.2/as-of-system-time.html) queries whenever possible**
> - Historical queries never wait on anything and never block anything
> - Best possible performance - served by the nearest replica
> - Only if the application can use data that is 5 second old or older





### 2. Transaction Isolation Conflicts Explained

Isolation is a property implemented by a database that defines how the changes made by one transaction become visible to other transactions executing concurrently.

CockroachDB only supports `SERIALIZABLE` isolation.

The leveling language of the SQL specification defines `SERIALIZABLE` as the highest, i.e. the strongest isolation level. It guarantees that the following phenomena *will NOT* occur in a transaction:

1. Non-repeatable reads
2. Phantom reads

Legacy DBMS-s commonly use a lock-based concurrency implementation, whereby enforcing serializability requires read and write locks.

CockroachDB is using a non-lock based optimistic concurrency control, acquiring no read locks. To ensure serializability, CockroachDB *validates that the previous reads haven't changed at the commit time*. If reads are non-repeatable, the transaction in conflict is forced to restart.



##### Contention Illustration 2.1.  Classic Serialization violation (non-repetitive reads protection)

| Transaction 1 (multi-statement)                        | Transaction 2 (write, implicit)           |
| ------------------------------------------------------ | ----------------------------------------- |
| BEGIN;                                                 |                                           |
| SELECT * FROM t WHERE v < 300000;                      |                                           |
|                                                        | UPDATE t SET v=2 WHERE k=3;    `lock k=3` |
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
| *`Error 40001: Write Too Old error`*           | `success`                                      |
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




> ✅ **Remedy: Read as little as possible**
> - Design the transactions so SELECTs read the minimum number of tuples required to implement the business logic
> - Use indexes, particularly covering indexes
> - Contention is handled at the KV layer - think about conflicts at the level of keys



> ✅ **Remedy: Avoid Read-Modify-Write pattern whenever possible**
>
> - It may be possible to avoid reading a column value, modify and write it back by pushing the expression into an SQL update statement, so the transaction becomes [implicit](../system-overview/tech-overview-trsansaction-implicit-explicit.md), which has the best possible concurrency characteristics.
> - For example,  `UPDATE t SET v=v+1 WHERE k=2;` instead of increasing a counter in the application code.





### 3. Uncertainty Conflicts due to Possible Clock Skew Explained

In CockroachDB, every transaction starts and commits at a timestamp assigned by some CockroachDB node that receives the SQL.
When choosing this timestamp the node does not rely on any particular synchronization with any other CockroachDB node. This becomes the timestamp of each tuple written by the aforementioned transaction.

CockroachDB uses multi-version concurrency control (MVCC) - it stores multiple values for each tuple ordered by timestamp. A reader generally returns the latest tuple with a timestamp earlier than the reader's timestamp and can could ignore tuples with higher timestamps. But this is when a clock skew needs to be considered.

If a reader's timestamp is assigned by a CockroachDB node with a clock that is behind, it might encounter tuples with higher timestamps that were, in fact, committed before the reader's transaction but their timestamps were assigned by a clock that is ahead.

Tuples with timestamps above the reader’s but within the `max-offset` are considered to be ambiguous as to whether they're in the past or the future of the reader. If such an uncertain tuple is encountered, the *reader transaction will generally be forced to restart* at a higher timestamp so all previously uncertain changes are definitely in the past.

For information about handling transactions that had been forced to restart, review the [transaction retries](../system-overview/tech-overview-trsansaction-retires.md) section.



##### Contention Illustration 3.1.  Uncertainty Conflicts

| Transaction 1 (read, singleton or multi) | Transaction 2 (write)                                  |
| ---------------------------------------- | ------------------------------------------------------ |
| *`This scenario is event timing driven`* | *`Write needs to occur within max-offset of the read`* |
| *`Not trivial to reproduce`*             | *`work in progress`*                                   |




> ✅ **Remedy: Use [implicit](../system-overview/tech-overview-trsansaction-implicit-explicit.md) transactions wherever possible**
>
> - In case of a conflict, retried automatically on the gateway node (that originated the transaction)
> - Eliminates an unnecessary additional client<->gateway network round trips
> - Allows single-range txns to use a streamlined 1-phase commit fast-path
> - Hold locks for less time





### Identifying Contending Transactions

#### Using DB Console to Identify most contending Transaction 

***`UNDER CONSTRUCTION`***

[Transactions Page in DB Console](https://www.cockroachlabs.com/docs/cockroachcloud/transactions-page.html)

#### Using system tables (views)

***`UNDER CONSTRUCTION`***

[Transaction Contention](https://www.cockroachlabs.com/docs/v21.2/performance-best-practices-overview#find-transaction-contention)

```sql
SELECT DISTINCT
    t.database_name
  , t.schema_name
  , t.name
  , ti.index_name
  , ce.num_contention_events
  , ce.cumulative_contention_time
FROM
    crdb_internal.cluster_contention_events ce
    INNER JOIN crdb_internal.table_indexes ti
        ON  ce.index_id = ti.index_id AND ce.table_id = ti.descriptor_id
    INNER JOIN crdb_internal.tables t
        ON ce.table_id = t.table_id
ORDER BY
    ce.num_contention_events DESC
;
```





### Esoteric Situations that may lead to 40001 Retry Errors

The following is included for completeness, so operators are aware that

-  `40001` errors, that are normally associated with contention conflicts, can also occur outside of a contention situation
- While conflict resolution normally results in one party of a conflict yielding excution, it's possible that a conflict may result in more than one victim

#### Two transactions in a deadlock can cause each other to restart (both fail with a retry error)

CockroachDB implementation is designed to not let two transactions both cancel each other, so the system would continue making progress on at least one of them. However it can't be completely rule it out. It would be difficult to make an example of that situation. It's esoteric and a peculiarity of the code paths.  is possible", as a friendly FYI.

#### A transaction can get a retry error code spuriously

There are code paths that result in a retry error outside of a transaction conflict situation.

For example, a lease transfer clears the [timestamp cache](https://www.cockroachlabs.com/docs/stable/architecture/transaction-layer.html#timestamp-cache) which is used to provide serializable guarantees (eliminate non-repetitive and phantom reads). If that happens at a specific point in a transaction's life, its commit will be rejected because there is no way to verify the earlier reads are repetitive.




### Useful Resources

- [Troubleshooting Overview](https://www.cockroachlabs.com/docs/stable/troubleshooting-overview.html)
- [CockroachDB Transaction Layer](https://www.cockroachlabs.com/docs/v21.2/architecture/transaction-layer.html)
- [Troubleshoot Cluster Setup](https://www.cockroachlabs.com/docs/stable/cluster-setup-troubleshooting.html)
- [Troubleshoot Statement Behavior](https://www.cockroachlabs.com/docs/stable/query-behavior-troubleshooting.html)
- [Blog: What is Database Contention](https://www.cockroachlabs.com/blog/what-is-database-contention/)

