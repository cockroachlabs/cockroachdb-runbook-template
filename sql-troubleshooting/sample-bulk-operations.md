# Bulk Operations within CRDB
With **READ-COMITTED** single-server databases, it is typical to run various BULK operations directly from the SQL command line.  These need to be re-coded to properly operate in a **SERIALIZABLE** distribute SQL database.  


## Historical Bulk Operations

### CREATE TABLE AS SELECT (CTAS)

```sql
CREATE TABLE backup_table as SELECT * from source_table;
```

### INSERT INTO... SELECT (IAS)
```sql
INSERT INTO new_table 
SELECT * FROM source_table
WHERE ts < '2021-07-01'
```

### DELETE

```sql
DELETE FROM big_bloated_table
WHERE ts < '2021-01-01';
```

### MANUAL ARCHIVE

```sql
BEGIN;
    INSERT INTO archive_table
    SELECT * FROM source_table
    WHERE ts < '2021-04-01';

    DELETE FROM source_table
    WHERE ts < '2021-04-01';
COMMIT;
```

## BULK operations in CRDB

These historical BULK operations are not practical on large data sets spread across a distributed architecture.  This is made more complicated by **SERIALIZABLE** transactions.  The key to coding transactions with SERIALIZABLE is to keep the TXN size as small as possible.  This allows you to more effectivly scale without *RETRIES* or timeouts.

**So what is the way around it?**

Typically, these operations do **NOT** need to be done as one huge batch.  Breaking these operations into smaller batches and iterating is the best approach.  This arroach uses the SQL `LIMIT` operato and has you code the repitition of operations until finshed.

### DELETE with Limit

Using the `cockroach` or `crdb` binary and the `LIMIT` clause allows you to create a simple script to clean up a table.  The `cockroach` CLI  example has the [`--watch`](https://www.cockroachlabs.com/docs/stable/cockroach-sql.html#repeat-a-sql-statement) option which essentially repeats the command over and over.  The example below deletes in batches of 100 rows until there are no more rows to delete and the CLI returns `DELETE 0`.

```bash
cockroach sql --insecure --format csv --execute """
  DELETE FROM mybigtable
  WHERE my_timestamp < '2021-01-01'
  LIMIT 100
""" --watch 0.0001s |
while read d
do
  echo $d
  if [[ "$d" == "DELETE 0" ]]; then
     echo "DONE"
     exit
  fi
done
```

This is typically run from a shell on the `bastion` server.


### BULK MOVE data within the same cluster

This use case has you moving ALL rows from one table to another.  The initial use case has one smaller table ~1TB that needed to move rows to a table that had 20TB.  To make this happen, you can use the `DELETE` clause with `RETURNING` inside of a CTE.  Again, this can use the `--watch` option to easily script the move and since these are all SERIALIZABLE transactions, this is safe to interrupt and multi-thread.  The example below is just one of multiple threads that were run based on a `UUID` range of the primary key.

```bash
crdb sql --database revenue_platform_prod --execute """set disallow_full_table_scans=false;
WITH copybatch AS (
    DELETE FROM source_table
    WHERE 1=1 and
    id between '10000000-0000-0000-0000-000000000000' and '20000000-0000-0000-0000-000000000000'
    LIMIT 100
    RETURNING source_table.*
)
INSERT INTO destination_table
SELECT DISTINCT * FROM copybatch;
""" --watch 1s |
while read d
do
  echo $d
  if [[ "$d" == "INSERT 0" ]]; then
     echo "DONE"
     exit
  fi
done
}
```

### BULK INSERT as SELECT
It is best to AVOID using "create table as select" (CTAS) for BULK table backup operations.  It is better to create a version of the new table first and then use a program to read data from the source and `UPSERT` in controlled batches.  The [copier](https://github.com/doordash/crdb-operator/tree/main/cmd/copier) program was created efficiently copy data with CockroachDB.  This tool can copy some or all of the a table between clusters.  You can copy some or all of the columns and limit the input by supplying a *predicate* to the initial SELECT portion.

This code is under development, so please provide feedback via github issues.  At the moment, there are a fair number of command-line parameters and environment variables needed.  The example below uses the [journals_saved_triggers_20210713.sh](journals_saved_triggers_20210713.sh) script and [journals_saved_triggers_20210713.ini](journals_saved_triggers_20210713.ini) file to copy some outlier data into the `saved_triggers_20210713` table.


### BULK UPDATE of DATA
Finally, the above bulk `INSERT` operation was used to copy some errant rows before performing a bulk update operation.  This errant unindexed data is ~200K rows from a 1.2 Billion row table.  So, to retrieve this data, the entire table must be scanned.  Once the errant rows were written into the `saved_triggers_20210713` table, we are nearly ready to proceed with the update.

To clearly keep track of rows that have been updated, it is best to add a `BOOLEAN` to the `saved_triggers_20210713` table.  

```sql
ALTER TABLE saved_triggers_20210713
ADD COLUMN done BOOLEAN DEFAULT FALSE;
```

Once the `done` column was added, it can be use to assist to account for the BATCH updates.  The `journals` table was the targeted for this update operation using the `saved_triggers_20210713` table to drive and `LIMIT` the `UPDATE` size.

```bash
cockroach sql --format csv --execute """
WITH upd as (
  UPDATE journals SET trigger_time = (trigger_time::INT // 1000)::TIMESTAMP
  FROM saved_triggers_20210713
  WHERE journals.id = saved_triggers_20210713.id AND
        saved_triggers_20210713.done is FALSE
  LIMIT 1000
RETURNING journals.id
)
UPDATE saved_triggers_20210713
SET done = TRUE
WHERE saved_triggers_20210713.id in (SELECT id from upd)
""" --watch 0.0001s |
while read d
do
  echo $d
  if [[ "$d" == "UPDATE 0" ]]; then
     echo "DONE"
     exit
  fi
done
```