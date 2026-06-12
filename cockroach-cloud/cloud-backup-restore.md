# CockroachDB Cloud — Backup & Restore


## Overview

This runbook covers backup and restore operations for CockroachDB Cloud clusters. CockroachDB Cloud provides two complementary backup mechanisms:

1. **Managed Backups** — Automated backups taken and stored by Cockroach Labs on behalf of the customer. Enabled by default on all tiers.
2. **Self-Managed Backups** — Customer-initiated backups stored in the customer's own cloud storage (S3, GCS, Azure Blob). Supported on all tiers.

Both mechanisms use CockroachDB's native backup engine, which produces consistent, distributed backups without cluster downtime.


## Concepts & Definitions

| Term | Definition |
|---|---|
| **Full Backup** | A complete snapshot of all data at a given timestamp. Baseline for incremental backups. |
| **Incremental Backup** | A backup of only the data that changed since the last full or incremental backup. Reduces storage and backup time. |
| **Backup Collection** | A full backup plus all its dependent incremental backups stored in a single storage path. |
| **Managed Backup** | Automated backup managed by Cockroach Labs, stored in Cockroach Labs-owned cloud storage. |
| **Self-Managed Backup** | Customer-initiated backup stored in customer-owned cloud storage. |
| **PITR (Point-in-Time Recovery)** | Ability to restore a cluster to any timestamp within the backup's revision history window. |
| **Protected Timestamp** | A mechanism that prevents garbage collection of MVCC data that a backup or schedule depends on. |
| **Revision History** | The window of time within which PITR is possible, determined by `revision_history` backup option. |
| **GC TTL** | Garbage collection time-to-live; data deleted before GC TTL has expired can be included in PITR. |

<div style="margin-bottom: 50px;"></div>

## Managed Backups

### Configuration

Managed backup settings can be configured in **Cloud Console → Backup & Restore**:

| Setting | Options | Recommendation |
|---|---|---|
| Frequency | Hourly, Daily | Daily full + Hourly incremental for most production workloads |
| Retention | 2–365 days | 30 days minimum for production; align with RPO requirements |
| Point-in-Time Recovery | Enabled/Disabled | Enable for production clusters | 

<div style="margin-bottom: 20px;"></div>

>Cockroach Labs retains managed backups for the configured retention period. After cluster deletion, backups are retained for the configured retention time (max 30 days if agreement terminated).

<div style="margin-bottom: 50px;"></div>

### Restoring from a Managed Backup

**Via Cloud Console:**
1. Navigate to your cluster → **Backup & Restore**
2. Select the backup or point-in-time to restore from
3. Choose restore target:
   - **Restore into this cluster** (in-place restore — will overwrite current data)
   - **Restore into a new cluster** (recommended for production; preserves current cluster)
5. Click **Restore**
6. Monitor restore progress in the **Jobs** view

> In-place restores are destructive. Always prefer restoring to a new cluster for validation before decommissioning the original.

**To restore a deleted cluster:**
Contact Cockroach Labs Support. Managed backups from deleted clusters are available for the configured retention window.

<div style="margin-bottom: 50px;"></div>


## Self-Managed Backups

### Supported Storage Backends

| Provider | URI Format |
|---|---|
| AWS S3 | `s3://bucket-name/path?region=us-east-1` |
| Google Cloud Storage | `gs://bucket-name/path` |
| Azure Blob Storage | `azure-blob://container-name/path?AZURE_ACCOUNT_NAME=...` |

>Enable object lock / immutable storage on your backup bucket to protect against accidental deletion or ransomware.

<div style="margin-bottom: 50px;"></div>

### Authentication

**Recommended: IAM Roles (implicit auth)**
```sql
-- AWS: Instance profile / IAM role attached to cluster nodes
BACKUP INTO 's3://my-bucket/crdb-backups?AUTH=implicit&region=us-east-1';

-- GCP: Workload Identity
BACKUP INTO 'gs://my-bucket/crdb-backups?AUTH=implicit';
```

**Alternative: Explicit credentials**
```sql
BACKUP INTO 's3://my-bucket/crdb-backups'
  WITH AWS_ACCESS_KEY_ID = '...', AWS_SECRET_ACCESS_KEY = '...';
```

### Full Cluster Backup

```sql
-- Full cluster backup (recommended baseline)
BACKUP INTO 's3://my-bucket/crdb-backups?AUTH=implicit&region=us-east-1'
  AS OF SYSTEM TIME '-10s'
  WITH revision_history;
```

### Incremental Backup (into existing collection)

```sql
BACKUP INTO LATEST IN 's3://my-bucket/crdb-backups?AUTH=implicit&region=us-east-1'
  AS OF SYSTEM TIME '-10s'
  WITH revision_history;
```

### Database-Level Backup

```sql
BACKUP DATABASE my_database
  INTO 's3://my-bucket/crdb-backups/my_database?AUTH=implicit&region=us-east-1'
  AS OF SYSTEM TIME '-10s'
  WITH revision_history;
```

### Table-Level Backup

```sql
BACKUP TABLE my_database.my_table
  INTO 's3://my-bucket/crdb-backups/my_table?AUTH=implicit&region=us-east-1'
  AS OF SYSTEM TIME '-10s';
```

### Scheduled Backups (Recommended for Self-Managed)

```sql
-- Full backup weekly, incrementals daily
CREATE SCHEDULE daily_backups
  FOR BACKUP INTO 's3://my-bucket/crdb-backups?AUTH=implicit&region=us-east-1'
    WITH revision_history
  RECURRING '@daily'
  FULL BACKUP '@weekly'
  WITH SCHEDULE OPTIONS first_run = 'now';

-- Verify the schedule
SHOW SCHEDULES;
```

<div style="margin-bottom: 50px;"></div>


## Restore Procedures

### Validate a Backup Before Restoring

```sql
-- List available backups in a collection
SHOW BACKUPS IN 's3://my-bucket/crdb-backups?AUTH=implicit&region=us-east-1';

-- Inspect a specific backup
SHOW BACKUP LATEST IN 's3://my-bucket/crdb-backups?AUTH=implicit&region=us-east-1';

-- Validate backup integrity (non-destructive)
RESTORE FROM LATEST IN 's3://my-bucket/crdb-backups?AUTH=implicit&region=us-east-1'
  WITH verify_backup_table_data;
```
<div style="margin-bottom: 50px;"></div>

### Full Cluster Restore

>Full cluster restores overwrite all data on the target cluster. Only restore to a new, empty cluster unless performing emergency disaster recovery.

```sql
-- Restore to a specific point in time (PITR)
RESTORE FROM LATEST IN 's3://my-bucket/crdb-backups?AUTH=implicit&region=us-east-1'
  AS OF SYSTEM TIME '2026-06-01 12:00:00';
```
<div style="margin-bottom: 50px;"></div>

### Database Restore

```sql
-- Restore a database (to the same cluster or a new cluster)
RESTORE DATABASE my_database
  FROM LATEST IN 's3://my-bucket/crdb-backups/my_database?AUTH=implicit&region=us-east-1';

-- Restore with a name change (avoids conflict with existing database)
RESTORE DATABASE my_database
  FROM LATEST IN 's3://my-bucket/crdb-backups/my_database?AUTH=implicit&region=us-east-1'
  WITH new_db_name = 'my_database_restored';
```
<div style="margin-bottom: 50px;"></div>

### Table Restore

```sql
-- Restore individual tables
RESTORE TABLE my_database.my_table
  FROM LATEST IN 's3://my-bucket/crdb-backups/my_table?AUTH=implicit&region=us-east-1'
  WITH into_db = 'my_database';
```
<div style="margin-bottom: 50px;"></div>

## Monitoring Backup Jobs

```sql
-- Check status of running and recent backup/restore jobs
SELECT job_id, job_type, status, created, finished,
       fraction_completed,
       description
FROM crdb_internal.jobs
WHERE job_type IN ('BACKUP', 'RESTORE', 'CREATE SCHEDULE FOR BACKUP')
ORDER BY created DESC
LIMIT 20;

-- Check scheduled backup health
SELECT name, state, next_run, error
FROM crdb_internal.schedules
WHERE label LIKE '%backup%';
```

**Cloud Console:** Navigate to **Jobs** to see backup and restore job status with progress indicators.

<div style="margin-bottom: 50px;"></div>

## Backup Schedule & Retention Design

| RPO Target | Recommended Configuration |
|---|---|
| 24 hours | Daily full backups, 30-day retention |
| 4 hours | Daily full + 4-hour incrementals, 30-day retention |
| 1 hour | Daily full + hourly incrementals + PITR enabled |
| 15 minutes | PITR with hourly incrementals; GC TTL ≥ 1 hour |

>Enable `revision_history` on all scheduled backups to support PITR within the backup window.

<div style="margin-bottom: 50px;"></div>

## Post-Restore Verification

```sql
-- Verify cluster/database came up cleanly
SELECT node_id, is_live FROM crdb_internal.kv_node_status;

-- Spot check row counts on critical tables
SELECT count(*) FROM my_database.orders AS OF SYSTEM TIME follower_read_timestamp();
SELECT count(*) FROM my_database.users AS OF SYSTEM TIME follower_read_timestamp();

-- Check for any pending schema change jobs post-restore
SELECT job_type, status FROM crdb_internal.jobs
WHERE status NOT IN ('succeeded', 'canceled')
ORDER BY created DESC;

-- Verify sequences are correct (especially if partial restore)
SELECT sequence_name, last_value
FROM information_schema.sequences
WHERE sequence_schema = 'public';
```

**Application verification:**
- Application connectivity to restored cluster confirmed
- Application smoke tests pass against restored data
- Sequence values are not behind current maximum IDs
- Foreign key constraints pass (`VALIDATE CONSTRAINT` if needed)

<div style="margin-bottom: 50px;"></div>

## Troubleshooting

| Symptom | Likely Cause | Action |
|---|---|---|
| Backup job failing with `protected timestamp` error | GC TTL too short for backup window | Increase `gc.ttlseconds` on affected tables or increase backup frequency |
| Restore fails with version mismatch | Backup taken on newer version | RESTORE only supports N → N+1 version jumps; upgrade cluster first |
| Restore job paused/stuck | Transient cloud storage error | Resume with `RESUME JOB <job_id>`; check cloud storage permissions |
| Missing tables after restore | Table-level backup taken without dependent types/sequences | Use database-level backup for completeness |
| Schedule stopped running | Schedule paused due to consecutive failures | Check `error` in `crdb_internal.schedules`; fix root cause then `RESUME SCHEDULES` |



## Related Resources

- [Backup & Restore in CockroachDB Cloud](https://www.cockroachlabs.com/docs/cockroachcloud/backup-and-restore-overview)
- [Managed Backups](https://www.cockroachlabs.com/docs/cockroachcloud/managed-backups)
- [BACKUP Statement Reference](https://www.cockroachlabs.com/docs/stable/backup)
- [RESTORE Statement Reference](https://www.cockroachlabs.com/docs/stable/restore)
- [Scheduled Backups](https://www.cockroachlabs.com/docs/stable/manage-a-backup-schedule)
- [Point-in-Time Recovery](https://www.cockroachlabs.com/docs/stable/restore#point-in-time-restore)
