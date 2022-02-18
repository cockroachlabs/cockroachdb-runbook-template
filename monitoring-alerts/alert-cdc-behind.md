# Alert: Changefeed is Behind

## Purpose of this Alert
Changefeed has fallen behind. This can be due to cluster capacity or changefeed sink availability. 

## Monitoring Metric

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