## Affected Version(s): 
All

## Issue Indicators/Example Errors
Stuck queries

## Description
When users experience hanging queries and the cluster is healthy (i.e. no unavailable ranges, network partitions, etc), a possible cause could be long-running transactions that hold intents open for (practically) unlimited durations. Other queries trying to access those rows may then have to wait for that transaction to complete (by committing or rolling back) before they can make progress. These are hard to diagnose via the stmts/txns page since contention is only reported after the conflict has been resolved (which is never, in the scenario at hand). 

Note that if the queries do return, the txn/stmt page (sorting by contention time) and/or statement bundles might be more helpful.

## Solution

```sql
-- find long-running transactions
select now()-start as dur from crdb_internal.cluster_transactions where now() - start > interval '10m' order by dur desc limit 10;
-- if lock=true (or seq > 0), this is a writing txn and its intents will block access for other txns

-- can find the client session owning this txn
select * from crdb_internal.cluster_sessions where kv_txn = <id_from_previous_query>

-- the txn or session can be canceled using CANCEL (QUERY|SESSION) X to check if that
-- resolves the problem.
```

If the above returns lots of transactions, it is often the case than a single transaction (typically with `lock=true`) is blocking others, and those may be blocking yet others. Try to look for the oldest, longest-running transaction and cancel that first – that may be sufficient to unblock all of the others.

## References
- https://github.com/cockroachlabs/support/issues/1333#issuecomment-974652838