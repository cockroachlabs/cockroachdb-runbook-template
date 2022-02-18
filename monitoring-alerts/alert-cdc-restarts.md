# Alert: Frequent Changefeed Restarts

## Purpose of this Alert
Changefeed can restart frequently during cluster upgrades when job coordinator duties are passed around from node to node or when the changefeed sinks are not available. Use the steps below to identify the cause.

## Monitoring Metric

```
sum(changefeed_error_retries{job="crdb"}) - sum(changefeed_error_retries{job="crdb"} offset 2h)
```

### Alert Rule

| Tier     | Definition                                                   |
| -------- | ------------------------------------------------------------ |
| CRITICAL | This alert is not critical |
| WARNING  | If number of restart is greater than `30` for more than `1800s` |


## Alert Response
1. If the changefeeds must be paused during cluster upgrades and this alert is raised, mute this alert and closely monitor the changefeed lantecy using the query below.

```sql
select job_id, status,((high_water_timestamp/1000000000)::int::timestamp)-now() as "changefeed latency",created, left(description,60),high_water_timestamp from crdb_internal.jobs where job_type = 'CHANGEFEED' and status in ('running', 'paused','pause-requested') order by created desc;
```

2. If the cluster is not undergoing maintenance, check the health of sink endpoints (e.g. AWS S3, Kafka). You can check Scalyr for sink connection errors - e.g. `ERROR: connecting to kafka: path.to.cluster:port: kafka: client has run out of available brokers to talk to (Is your cluster reachable?)`

3. If there are no issues with the sink endpoints and there are no cluster maintenance, contact Cockroach Labs support.