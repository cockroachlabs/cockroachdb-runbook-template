# Alert: Version Mismatch

### Purpose of this Alert

All CockroachDB cluster nodes are running exactly the same executable (with identical build label). This warning is a safety catch to guard against an operational error when some node(s) were not upgraded.

------

### Monitoring Metric

```
build.timestamp
```



### Alert Rule

| Tier    | Definition                                                   |
| ------- | ------------------------------------------------------------ |
| WARNING | Non-uniform executable Build version across cluster nodes for more than 4 hours. |



### Alert Response

Ensure all cluster nodes are running exactly the same CockroachDB version, including the patch release version number.



-----

# Alert: Major Upgrade Un-finalized

### Purpose of this Alert

A major upgrade should not be left un-finalized for an extended period of time, beyond a small number of days necessary to gain confidence in the new release. This warning is a safety catch to guard against an operational error when a major upgrade is left un-finalized.

------

### Monitoring Metric

```
No metric is available. Monitor via a query against system tables.
```



### Monitoring Result of SQL Query

*Run the check query provided* in the [CockroachDB version upgrade article](../routine-maintenance/release-upgrade.md#finalizing-a-major-release-upgrade). If the query returns `false`, the last major upgrade has not been finalized.



### Alert Rule

| Tier    | Definition                                              |
| ------- | ------------------------------------------------------- |
| WARNING | Major release upgrade has not been finalized for 3 days |



### Alert Response

Finalize the upgrade.

