# Runbook: Cluster restore from AWS S3

This runbook covers the process and considerations for a CockroachDB cluster restoration where the data is located in an AWS S3 storage bucket.

## Requirements

- Existing AWS cloud account
- Existing IAM access key (Runbook: [AWS IAM Access key creation](AWS-IAM-access-key.md))
- Existing source bucket that contains a prior CockroachDB cluster backup
- CockroachDB 22.2

## Considerations and Caveats

- **Admin users** have full rights & privileges to launch a data **RESTORE** since backup & restore operations fall under typical database-admin use cases.
- **Regular users** must have **RESTORE** system privilege to launch a data restore operation. This is part of the user-privilege model that offers finer control and flexibility across user accounts.
See the **_Pre-Procedure_** steps to configure a regular user with **RESTORE** privileges.
- [External Connections](https://www.cockroachlabs.com/docs/stable/create-external-connection.html) can be created to simplify connection references and URL maintenance using logical naming.
- Restore operations are ideal into freshly created databases. This means the database must not contain any existing data, tables, views, etc.
  If you plan on deleting the contents of an existing database to accommodate an upcoming **RESTORE**, due to MVCC capabilities, there is still a 25 hour (default) period before the garbage-collector permanently deletes the data (removes tombstones).
  The only way to continue restoring into an existing database is to temporarily set the GC to a short interval to allow time for disposal of the descriptors.
- Disable access to the database until the restore operation is complete.
  This can be accomplished by restricting connections via load-balancers or by restricting the SQL ports using your network administration tools.

## Pre-Procedure for Non-Admin Users

Admin users can skip to the **Procedure** below.
For all non-admin users, the following _one-time_ steps are necessary to allow restore operations.

1. Admin must grant **cluster-level restore** privileges using the example below, where **'user'** is an existing named-account.

```sql
GRANT SYSTEM RESTORE to user;
```

## Procedure

2. Run the example BACKUP statement as a background process.

```sql
RESTORE FROM LATEST IN 's3://your-bucket-name/your-cluster-backup-name?AWS_ACCESS_KEY_ID=AKIAXXXXXXXX&AWS_SECRET_ACCESS_KEY=YYYYYYYYYYYY' WITH DETACHED;
```

- **_your-bucket-name_** is the name of the bucket to send data into.
  This name must be identical to what was applied in the bucket-creation process.
- **_your-cluster-backup-name_** is the name of the backup to restore from the bucket.
  This name is defined as the **root folder** in the bucket. To identify all the backups in a bucket, use the AWS cloud console to navigate into the S3 services and select your bucket.
  The list of root folders represent the names of backups taken in the past.
  Subfolder names within these root folders represent the timestamps of all the backups.
- **_AWS_ACCESS_KEY_ID_** and **_AWS_SECRET_ACCESS_KEY_** are the access key parts required to authorize CockroachDB access to the AWS S3 services.
- **_DETACHED=TRUE_** prevents the interactive SQL session from blocking during the restore.
  Default bevahiour will block the user until the restore is complete.
  This option will let you to continue working with the database in the current session, including monitoring the status of the restore process (see the following command: SHOW JOBS);

3. During the cluster-restore operation, the databases will be created as data is received and you will not have full visibility of the cluster until the data-restoration is complete.

```sql
select status, created, finished  from [show jobs] where job_type = 'RESTORE';
```

  The above query will deliver a row detailing the restore operation, for example:

```sql
mark@localhost:26257/defaultdb> select status, created, finished  from [show jobs] where job_type = 'RESTORE';
   status   |          created           |          finished
------------+----------------------------+-----------------------------
  succeeded | 2023-01-19 18:41:44.030241 | 2023-01-19 18:42:39.081617
(1 row)
```
