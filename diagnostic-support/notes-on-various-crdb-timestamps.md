Cockroach has a number of different timestamps but they fall into 3 categories:
Transaction Timestamp:
- now() forces the transaction to observe its commit timestamp and thus commit at that time
- current_timestamp()
- transaction_timestamp()
These all return the timestamp of when the transaction starts.
This is updated if the transaction gets pushed and returns the new start time.
Note that you can’t use this time for AS OF SYSTEM TIME queries as this is the read time when the transaction starts,
not the timestamp when it was written.From our docs:
The value is based on a timestamp picked when the transaction starts and which stays constant throughout the transaction. This timestamp has no relationship with the commit order of concurrent transactions.
So these should never be used if strict ordering is required.
1. Statement Timestamp:
- statement_timestamp()
This, when used within an explicit transaction, is just the time for the statement and will always be higher than the transaction timestamp mentioned above. It should only be used for observing the order of statements within a single transaction.
2. Cluster Timestamp:
- cluster_logical_timestamp()
This timestamp forces the transaction to observe its commit timestamp and thus commit at that time. When used, cockroach cannot push the transaction. This is the only timestamp that can be used for guaranteeing the order of a transaction.
This means the transaction cannot be auto-retried. So there must be a mechanism in the app to re-issue the transaction if it fails with a 40001 error.
See https://www.cockroachlabs.com/docs/stable/transaction-retry-error-reference.html for all the transaction retry errors.3. crdb_internal_mvcc_timestamp
We have implemented a way to retrieve the mvcc timestamps in SQL. This column ensures the correct ordering as it contains the actual time that it was committed:
https://github.com/cockroachdb/cockroach/pull/51494It can be retrieved like so:
SELECT crdb_internal_mvcc_timestamp FROM t WHERE x = 2;
______________________________________________________________
example:
```
create table entity(
entity_key UUID PRIMARY KEY,
text STRING);create table outbox(
event_key UUID PRIMARY KEY,
entity_key UUID NOT NULL);
```
In session 1 on node 1:
```
begin transaction
insert into entity (1234, 'ABC');
insert into outbox(5678, 1234);
commit;
end transaction;
```
In session 2 on node 2:
```
begin transaction;
update entity
set text = 'DEF'
where entity_key = 1234;
insert into outbox(9876, 1234);
commit;
end transaction;
```
