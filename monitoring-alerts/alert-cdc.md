# Alert: Changefeed Falling Behind

### Purpose of this Alert

Changefeed has fallen behind. This can be due to cluster capacity or changefeed sink availability. 

------

### Monitoring Metric

```
(max(changefeed_max_behind_nanos{job="crdb"})/1000000000) > 60
```

### Alert Rule

| Tier     | Definition                                                   |
| -------- | ------------------------------------------------------------ |
| CRITICAL | Max changefeed latency for any changefeed is greater than `15m` |
| WARNING  | Max changefeed latency for any changefeed is greater than `10m` |


## Alert Response
1. Open changefeeds metrics dashboard for the cluster (e.g. https://url/#/metrics/changefeeds/cluster) and check max latency

Alternatively, individual changefeed latency can be verified by using the SQL cli

```sql
select job_id, status,((high_water_timestamp/1000000000)::int::timestamp)-now() as "changefeed latency",created, left(description,60),high_water_timestamp from crdb_internal.jobs where job_type = 'CHANGEFEED' and status in ('running', 'paused','pause-requested') order by created desc;
```

2. Copy the job number for the changefeed job with highest latency and pause it

```sql
PAUSE JOB 681491311976841286;
```

3. Check the status of the pause request by running the same query from step 1. If the job status is `pause-requested`, check again in few minutes.

4. After the job is `paused`, resume the job.

```sql
RESUME JOB 681491311976841286;
```

5. The changefeed latency may not progress after above steps due to lack of cluster resources, availability of changefeed sink, and etc. Contact Cockroach Labs support.



------------



# Alert: Frequent Changefeed Restarts

### Purpose of this Alert

Changefeed can restart frequently during cluster upgrades when job coordinator duties are passed around from node to node or when the changefeed sinks are not available. Use the steps below to identify the cause.

------

### Monitoring Metric

```
sum(changefeed_error_retries{job="crdb"}) - sum(changefeed_error_retries{job="crdb"} offset 2h)
```

### Alert Rule

| Tier    | Definition                                                   |
| ------- | ------------------------------------------------------------ |
| WARNING | If number of restart is greater than `30` for more than `1800s` |



### Alert Response

1. If the changefeeds must be paused during cluster upgrades and this alert is raised, mute this alert and closely monitor the changefeed latency using the query below.

```sql
select job_id, status,((high_water_timestamp/1000000000)::int::timestamp)-now() as "changefeed latency",created, left(description,60),high_water_timestamp from crdb_internal.jobs where job_type = 'CHANGEFEED' and status in ('running', 'paused','pause-requested') order by created desc;
```

2. If the cluster is not undergoing maintenance, check the health of sink endpoints (e.g. AWS S3, Kafka). You can check Scalyr for sink connection errors - e.g. `ERROR: connecting to kafka: path.to.cluster:port: kafka: client has run out of available brokers to talk to (Is your cluster reachable?)`

3. If there are no issues with the sink endpoints and there are no cluster maintenance, contact Cockroach Labs support.



-----------------



# Alert: Changefeed is Stopped

### Purpose of this Alert

This alert is raised when changefeed jobs are cancelled or paused. Changefeed can be paused from SQL cli or when an error is encountered (if created with `on_error = 'pause'`). 

------

### Monitoring Metric

```
(sum(jobs_changefeed_currently_running{job="crdb"})) - (sum(jobs_changefeed_currently_running{job="crdb"} offset 5m))
```

### Alert Rule

| Tier     | Definition                                                   |
| -------- | ------------------------------------------------------------ |
| CRITICAL | The number of stopped changefeed is greater than `0` for more than `600s` |
| WARNING  | The number of stopped changefeed is greater than `0` for more than `300s` |



## Alert Response

1. Open SQL cli and check status of each changefeed. 

```sql
select job_id, status,((high_water_timestamp/1000000000)::int::timestamp)-now() as "changefeed latency",created, left(description,60),high_water_timestamp from crdb_internal.jobs where job_type = 'CHANGEFEED' and status in ('running', 'paused','pause-requested') order by created desc;
```
2. If all the changefeeds are in `running` state, one or more feed may have ran into an error and recovered. Check the UI (e.g. `https://<cluster_url>/#/metrics/changefeeds/cluster`) for number of changefeed restarts. 

3. If any of the changefeeds are paused, try resume with the job id (e.g. `RESUME JOB 681491311976841286;`).

4. Contact Cockroach Labs support if the changefeed cannot be restarted.

