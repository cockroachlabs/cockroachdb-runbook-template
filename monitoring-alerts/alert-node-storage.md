

> 
>
> ✅  **Clarification about `write stall` and `disk stall` conditions**
>
> There are 2 distinctly different conditions referring to "stalls" in I/O operations. Operators don't need to create alerts for either condition, in spite of their alarming names.
>
> ----
>
> `write stall` is an internal Pebble concept caused by too many memtables' flushes, which usually indicates an excessive write rate by a client. `write stall` messages in logs do not require operator's immediate attention. CockroachDB node can log write stall messages when the disk is churning through a large amount of data ingestion (flushing memtables).
>
> Examples when high write rates can cause `write stall` log messages:
>
> \- A sharp increase (spike) in insert throughput from user workload or some job
>
> \- A large scope delete, such as a TTL job running on a very large set of data or on a large table, triggering a large influx of memtable flushes
>
> A slow disk is less often the culprit. However, if potential causes above pointing to behavior within the database causing the `write stall`s are ruled out, the customer may want to investigate the health and IO capacity of their disks.
>
> ----
>
> `disk stall` is a condition that occurs when a disk IO system call time takes more than a configurable time deadline. The deadline for CockroachDB data IO is configurable via a cluster setting `storage.max_sync_duration` for the max allowable time an IO operation can be waiting in the storage layer. `COCKROACH_LOG_MAX_SYNC_DURATION` environment variable (there is no equivalent cluster setting) sets the max allowable time an IO operation can be waiting in the message logging framework.
>
> Disk stalls are covered in the broader [data availability](../system-overview/data-availability.md#disk-stalls) article.
>
> 



---

# Alert: Node Storage Capacity

### Purpose of this Alert

CockroachDB node will not able to operate if there is no free disk space on CockroachDB store volume.

------

### Monitoring Metric

```
capacity.available
```



### Alert Rule

| Tier     | Definition                                         |
| -------- | -------------------------------------------------- |
| WARNING  | Node free disk space is less than 30% for 24 hours |
| CRITICAL | Node free disk space is less than 10% for 1 hour   |




### Alert Response

Increase the size of CockroachDB node storage capacity.  CockroachDB  storage volumes should not be utilized more than 60% (40% free space). In an "disk full" situation, operator may be able to get a node "unstuck" by removing the [ballast file](https://www.cockroachlabs.com/docs/stable/cluster-setup-troubleshooting#automatic-ballast-files).



-----

# Alert: Node Storage Performance

### Purpose of this Alert

Under-configured or under-provisioned disk storage is a common root cause of inconsistent CockroachDB cluster performance and could also lead to cluster instability. Review the "Insufficient Disk IO performance" Section of [the common problems experienced by CockroachDB users](../most-common-problems/README.md).

------

### Monitoring Metric

```
sys.host.disk.iopsinprogress (storage device average queue length)
```



### Alert Rule

| Tier     | Definition                                                   |
| -------- | ------------------------------------------------------------ |
| WARNING  | Storage device average queue length is greater than 10 for 10 seconds |
| CRITICAL | Storage device average queue length is greater than 20       |



### Alert Response

See Resolution in the "Insufficient Disk IO performance" Section of [the common problems experienced by CockroachDB users](../most-common-problems/README.md).



