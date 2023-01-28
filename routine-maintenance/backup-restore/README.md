
# Backup & Restore Procedures

## Overview

This collection of procedures covers CockroachDB **backup** and **restore** operations.
You will find end-to-end implementations of typical use-cases, technical considerations, caveats, and examples of SQL to launch backups and restores of CockroachDB clusters.
These procedures will continue to grow as we evolve the database, integrate with other technologies, and explore trending customer use-cases that need attention.

## How often should you run a backup?

This depends on the nature of your workloads, and the **acceptable risk** tolerance for a **_quantifiable volume_** of data-loss since your last backup.
This quantifiable volume of data the **maximum** amount of data, measured by time of the last backup that is lost when the database was restored from the last backup-point.
The elapsed time since your last backup is your **RPO** (_recovery-point-objective_) and is defined by a backup schedule where you set a time-interval.
This time-interval should have a candence such that if a disaster occurs, the volume of data that is lost will have minimal or manageable impact to your business operations.

### Running backup outside of the schedule

Workloads can include large batched blocks of inserts or update queries which may be transient by nature, or alternatively they can be relatively small but critical changes to your data that's tied directly to you business success.
These out-of-band database updates are good candidates for manually triggered backup operations to protect these significant data events.

### Understand your risk tolerance to help define a backup schedule

Regular backups are necessary to mitigate risk of data-loss even with a fault-tolerant and resilient database such as CockroachDB.

Backup procedures are part of a due-diligence process where action-plans are in place to restore data to known points in time specified by your RPO goals.

### Factors to consider when defining your schedule

There is no calculator to define a schedule since many variables need to be explored, ranging from business-continuity requirements to technical considerations that may affect the backup process.
The below list will help identify focus-areas and testing strategies to consider as your schedule is developed.

 - **_Volume/recency of transactions_** that need restoration due to committments through service level agreements or priority of transactions.
 - **_Size of data_** that is backed up.
 The more data on your platform, the longer the backup process takes and the increase in latency may be a burden to active users that are connected to the database.
 **Note that a prior backup can be running when a new backup is triggered:** Your candence must be long-enough to allow the existing backup to complete.
 - **_Hardware or service limitations_** may limit the ingress of your backup files, whether it's by physical size, network bandwidth consumed, or quota-driven cloud requests.
 - **_Peak vs quiet hours of operation:_** Automated backups are best scheduled during non-peak hours of business operations, typically outside of typical business hours to help reduce overall latency for client connections and preserve expected query peformance.
 - **_What to backup_** because CockroachDB lets you backup the entire cluster, an individual database in the cluster, or an individual table in a database. 

## Full and incremental cluster-wide backups

Depending on the cadence of backups (eg: hourly, daily), CockroachDB by default assumes an incremental backup, and creates a second schedule with a larger candence that relative to the user-specified schedule. This is specifically documented with pre-defined candence rules tied to the [FULL BACKUP crontab](https://www.cockroachlabs.com/docs/v22.2/create-schedule-for-backup#parameters) parameters.
While you can force a full-backup for each scheduled trigger, in most cases this is not necessary because it generates undue workload on the cluster and consumes large volumes of storage.
- The backup process reads the entire cluster. From a byte-size perspective it can exceed GB or event TB sizes), and the backup process 
- Both full and incremental backups create independent packages and folder structures
- Incremental backups rely on the previous full (or incremental) backup data to perform a backup from the previous point in time.

## Backup & Restore operations using AWS S3

This set of guides define integration points between CockroachDB and AWS S3 for the data storage medium, including SQL syntax that specifically enables data flow between CockroachDB and AWS S3 storage buckets.
A successful solution will require experience with AWS technologies to take advantage of enhanced security capabilities or other long-term storage requirements that you may have.
Here you will find the most typical use-cases for S3 integration since it is not possible to outline every combination or scenario.

* **One-time** AWS configurations that are required to perform backups/restores using the integrated S3 storage capabilities of CockroachDB
  * [AWS IAM: Access key creation for CockroachDB integration](AWS-IAM-access-key.md)
  * [AWS S3: Create storage bucket for backup data](AWS-create-s3-bucket.md)

* Cluster level backups & restores using AWS S3
  * [Full cluster backup to AWS S3](full-cluster-backup-to-s3.md)
  * [Incremental cluster backup to AWS S3](incremental-cluster-backup-to-s3.md)
  * [Cluster restore from AWS S3](cluster-restore-from-s3.md)

## References & documentation

Cockroach Labs offers documentation into backup and restore operations that cover operational configurations, technical considerations, and usage examples.
Below is a focused subset of shortcuts that are used across these runbooks.

* [BACKUP: Home page with syntax, examples, techical considerations, and use-cases](https://www.cockroachlabs.com/docs/stable/backup.html)
* [RESTORE: Home page with syntax, examples, technical considerations, and use-cases](https://www.cockroachlabs.com/docs/stable/restore.html)
* [Take Full and Incremental Backups](https://www.cockroachlabs.com/docs/stable/take-full-and-incremental-backups.html)
* [SHOW BACKUP](https://www.cockroachlabs.com/docs/stable/show-backup.html)
* [ALTER BACKUP (Enterprise)](https://www.cockroachlabs.com/docs/stable/alter-backup.html)
* [CREATE SCHEDULE FOR BACKUP](https://www.cockroachlabs.com/docs/stable/create-schedule-for-backup.html)
* [ALTER BACKUP SCHEDULE](https://www.cockroachlabs.com/docs/stable/alter-backup-schedule.html)
* [CREATE EXTERNAL CONNECTION](https://www.cockroachlabs.com/docs/stable/create-external-connection.html)
* [SHOW CREATE EXTERNAL CONNECTION](https://www.cockroachlabs.com/docs/stable/show-create-external-connection.html)
* [DROP EXTERNAL CONNECTION](https://www.cockroachlabs.com/docs/stable/drop-external-connection.html)
* [Restoring Backups Across Database Versions](https://www.cockroachlabs.com/docs/stable/restoring-backups-across-versions.html)
