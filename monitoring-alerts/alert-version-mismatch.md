# Alert: Version Mismatch

### Purpose of this Alert

<describe>

| # Alert on version mismatch. |                                                              |
| ---------------------------- | ------------------------------------------------------------ |
|                              | # This alert is intentionally loose (4 hours) to allow for rolling upgrades. |
|                              | # This may need to be adjusted for large clusters.           |
|                              | - alert: VersionMismatch                                     |
|                              | expr: count by(cluster) (count_values by(tag, cluster) ("version", build_timestamp{job="cockroachdb"})) |
|                              | > 1                                                          |
|                              | for: 4h                                                      |
|                              | annotations:                                                 |
|                              | description: Cluster {{ $labels.cluster }} running {{ $value }} different versions |



------

### Alert Rule

| Tier     | Definition |
| -------- | ---------- |
| CRITICAL |            |
| WARNING  |            |



### Alert Response

The actual response varies depending on the alert tier, i.e. the severity of potential consequences.

- Check ....

- 

