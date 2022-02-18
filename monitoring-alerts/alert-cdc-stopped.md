# Alert: Changefeed is Stopped

## Purpose of this Alert
This alert is raised when changefeed jobs are cancelled or paused. Changefeed can be paused from SQL cli or when an error is encountered (if created with `on_error = 'pause'`). 

## Monitoring Metric

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