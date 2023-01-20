# Runbook: Incremental cluster backup to AWS S3

This runbook covers the process and considerations for a user-instantiated incremental cluster backup where the CockroachDB data is automatically sent to an AWS S3 storage bucket.
For scheduled backups or full backups, please review the other backup runbooks for guidance.

## Overview

Incremental backups with revision history are created by finding what data has been created, deleted, or modified since the timestamp of the **_last backup_** in the chain of backups.
For the first incremental backup in a chain, this timestamp corresponds to the timestamp of the base (full) backup.
For subsequent incremental backups, this timestamp is the timestamp of the previous incremental backup in the chain.
See [Backup Collections](https://www.cockroachlabs.com/docs/stable/take-full-and-incremental-backups.html#backup-collections) for details on timestamps & folder-structures.

## Requirements

- Existing AWS cloud account
- Existing IAM access key (Runbook: [AWS IAM Access key creation](AWS-IAM-access-key.md))
- Existing target AWS S3 Bucket (Runbook: [AWS S3: Create storage bucket for backup data](AWS-create-s3-bucket.md))
- CockroachDB 22.2 with an **enterprise license**
- Existing backups where the incremental backup will apply to

## Considerations and Caveats

- **IMPORTANT NOTE: If you restored a database from a prior-backup, you MUST perform a full backup before any incremental backups. This new backup (post-restore) will be the fresh/latest baseline clsuter-backup for all future incremental backup operations.** Incremental backups against restored clusters will fail because the active CockroachDB cluster GUID will not match to the saved cluster GUID in the storage bucket.
- **Admin users** have full rights & privileges to launch a data **BACKUP** since backup & restore operations fall under typical database-admin use cases.
- **Regular users** must have **BACKUP** system privilege to launch a data restore operation. This is part of the user-privilege model that offers finer control and flexibility across user accounts.
- [External Connections](https://www.cockroachlabs.com/docs/stable/create-external-connection.html) can be created to simplify connection references and URL maintenance using logical naming.
- RPO is a defined interval of time between backups where data could be permanently lost.
  Based on the workloads and impact of data to your business continuity, define a backup-schedule that reasonably meets your recovery point objectives. 
- Performance and data-size:
  Best time to perform or schedule a backup is off-hours or during times when database activity is minimal.
  Backups are background processes that are distributed across all nodes and contributes to cluster utilization.
  This added processing could increase latency to active database query operations. 
  See also [Performance Considerations](https://www.cockroachlabs.com/docs/stable/backup.html#performance) documentation.
- This runbook demonstrates the most typical AWS S3 backup use-case, but there are many combinations parameters and settings that are available to meet specific needs related to security/privileges and your cluster properties.
  Reviewing [Backup considerations](https://www.cockroachlabs.com/docs/stable/backup.html#considerations) will provide the scope of capabilities and features that can be applied to accommodate your environment.

## Pre-Procedure for Non-Admin Users

Admin users can skip to the **Procedure** below.
For all non-admin users, the following _one-time_ steps are necessary to allow backup operations.

1. Admin must grant **cluster-level backup** privileges using the example below, where **'user'** is an existing named-account.

```sql
GRANT SYSTEM BACKUP to user;
```

## Procedure

2. Run the example Incremental BACKUP statement as a background process.

```sql
BACKUP INTO LATEST IN 's3://your-bucket-name/your-cluster-backup-name?AWS_ACCESS_KEY_ID=AKIAXXXXXXXX&AWS_SECRET_ACCESS_KEY=YYYYYYYYYYYY' AS OF SYSTEM TIME '-10s' WITH OPTIONS (DETACHED=TRUE, revision_history);
```

- **_your-bucket-name_** is the name of the bucket to send data into.
  This name must be identical to what was applied in the bucket-creation process.
- **_your-cluster-backup-name_** is the name of the backup to be saved in the bucket.  This name will be saved as a _subfolder_ in the bucket, along with a subfolder-tree that indicates the actual time of the backup process. See [Backup Collections](https://www.cockroachlabs.com/docs/stable/take-full-and-incremental-backups.html#backup-collections) for details on folder-structure.
- **_AWS_ACCESS_KEY_ID_** and **_AWS_SECRET_ACCESS_KEY_** are the access key parts required to authorize CockroachDB access to the AWS S3 services.
- **_AS OF SYSTEM TIME '-10s'_** is a technique to improve performance of the backup operation by not waiting on active/running transactions. By collecting data at-least 10 seonds ago, CockroachDB reads data thta is already committed, reducing likelihood for read-retries due to statement contention or other locks.
  This value can be changed to accommodate your worloads.  If you frequently have queries running longer than 10 seconds then a higher number would make sense, but at the cost of backup data might be stale due to the running transactions.
- **_DETACHED=TRUE_** prevents the interactive SQL session from blocking during the backup.
  Default bevahiour will block the user until the backup is complete.
  This option will let you to continue working with the database in the current session, including monitoring the status of the backup process (see the following command: SHOW JOBS);
- **_revision_history_** is an (optional) _Enterprise-only_ feature that records every change made to the cluster within the garbage collection period leading up to and including the given timestamp.
  This offers the flexibility of point-in-time restores using this backup-package.  Normally restores are based on the backup timestamp, but a point-in-time restore will deliver data as it existed at any arbitrary point in time captured by the backup.

### Monitor the status of the backup process:

3. (Optional / one-time step) Create a view to simplify the monitoring of your backup jobs.
  This view will display all the **backup** jobs that have been issued, including the current state (status) of the process.

```sql
CREATE VIEW my_backups_view AS
SELECT
    job_id,
    job_type,
    substring(description, 0, 60) AS short_description,
    status,
    created,
    finished - started AS duration,
    fraction_completed AS pct_done,
    error
FROM [SHOW JOBS]
WHERE job_type = 'BACKUP';
```

4. (Optional) Query on this view to get the latest progress-report on the BACKUP activities:

```sql
SELECT * from my_backups_view;
```

  The above query will deliver rows for each recent and active (running) BACKUP operation, for example:

```js
root@localhost:26257/defaultdb> SELECT * from my_backups_view;
        job_id       | job_type |                      short_description                      |  status   |          created           |    duration     | pct_done | error
---------------------+----------+-------------------------------------------------------------+-----------+----------------------------+-----------------+----------+--------
  831743033539067905 | BACKUP   | BACKUP INTO '/2023/01/16-194340.05' IN 's3://mz-cockroachdb | succeeded | 2023-01-16 19:43:50.050853 | 00:00:29.751453 |        1 |
  831743551603376129 | BACKUP   | BACKUP INTO '/2023/01/16-194618.15' IN 's3://mz-cockroachdb | succeeded | 2023-01-16 19:46:28.15171  | 00:00:30.121611 |        1 |
  831743667598426113 | BACKUP   | BACKUP INTO '/2023/01/16-194653.55' IN 's3://mz-cockroachdb | succeeded | 2023-01-16 19:47:03.551698 | 00:00:29.815018 |        1 |
  831745137443504129 | BACKUP   | BACKUP INTO '/2023/01/16-195422.11' IN 's3://mz-cockroachdb | running   | 2023-01-16 19:54:32.112309 | NULL            |        0 |
(4 rows)
```

  Note that the **short_description** column shown above is a computed (reduced-width) column from **description** that makes the output easy to consume.
  Normally the **description** column includes the entirety of the backup statment, including bucket/path and the AWS keys that ran the backup.

## Subsequent incremental backups

Backup data is saved in the S3 bucket, under the same subfolder as indicated by the backup name (see **_your-cluster-backup-name_** above).
Incremental backups are placed into the '/incrementals' subfolder (default name, [but can be changed](https://www.cockroachlabs.com/docs/stable/backup.html#parameters)), with additional subfolders indicating the date and timestamp IDs for easy sorting.
