> 
>
> ✅  During rolling  maintenance, the changefeed jobs restart following node restarts.
>
> Operators can **mute alerts described below during routine maintenance procedures** to avoid unnecessary distractions.
>
> 



------

# Alert: Changefeed Failure

### Purpose of this Alert

Changefeeds can suffer permanent failures (that the jobs system will not try to restart). Any increase in this counter should prompt an operator's  investigative action.  

------

### Monitoring Metric

```
changefeed.failures
```

### Alert Rule

| Tier     | Definition                                    |
| -------- | --------------------------------------------- |
| CRITICAL | If the number of failures is greater than `0` |


### Alert Response

1. If the alert goes off during cluster maintenance, mute it. Otherwise start investigation with the query:

```sql
select job_id, status,((high_water_timestamp/1000000000)::int::timestamp)-now() as "changefeed latency",created, left(description,60),high_water_timestamp from crdb_internal.jobs where job_type = 'CHANGEFEED' and status in ('running', 'paused','pause-requested') order by created desc;
```

2. If the cluster is not undergoing maintenance, check the health of sink endpoints. In case of Kafka, check for sink connection errors such as  `ERROR: connecting to kafka: path.to.cluster:port: kafka: client has run out of available brokers to talk to (Is your cluster reachable?)`





------

# Alert: Frequent Changefeed Restarts

### Purpose of this Alert

Changefeed automatically restart in case of transient errors. However "too many" restarts (outside of a routine maintenance procedure) may be due to a systemic condition and should be investigated.  

------

### Monitoring Metric

```
changefeed.error_retries
```

### Alert Rule

| Tier    | Definition                                                   |
| ------- | ------------------------------------------------------------ |
| WARNING | If number of restarts is greater than `50` for more than `15 minutes` |


### Alert Response

Same as when responding to Changefeed Failures.





------

# Alert: Changefeed Falling Behind

### Purpose of this Alert

Changefeed has fallen behind. Determined by the end to end lag between a committed change and that change applied at  destination. This can be due to cluster capacity or changefeed sink availability.

-----


### Monitoring Metric

```
changefeed.commit_latency
```


### Alert Rule

| Tier     | Definition                                                   |
| -------- | ------------------------------------------------------------ |
| WARNING  | Max end to end lag for any changefeed is greater than `10 minutes` |
| CRITICAL | Max end to end lag for any changefeed is greater than `15 minutes` |


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

5. The changefeed latency may not progress after above steps due to lack of cluster resources, availability of changefeed sink, etc.  Escalate to L2 Support.





-----------------

# Alert: Changefeed has been Paused for long time 

### Purpose of this Alert

A hedge against an operational error. Changefeed jobs should not be  paused for long time b/c the protected timestamp prevents garbage collections.  This is a safety catch to guard against an inadvertently "forgotten" pause. 

------

### Monitoring Metric

```
jobs.changefeed.currently_paused
```


### Alert Rule

| Tier     | Definition                                                   |
| -------- | ------------------------------------------------------------ |
| WARNING  | The number of paused changefeeds is greater than `0` for more than `15 minutes` |
| CRITICAL | The number of paused changefeeds is greater than `0` for more than `60 minutes` |


## Alert Response

1. Open SQL cli and check status of each changefeed. 

```sql
select job_id, status,((high_water_timestamp/1000000000)::int::timestamp)-now() as "changefeed latency",created, left(description,60),high_water_timestamp from crdb_internal.jobs where job_type = 'CHANGEFEED' and status in ('running', 'paused','pause-requested') order by created desc;
```
2. If all the changefeeds are in `running` state, one or more feed may have ran into an error and recovered. Check the UI (e.g. `https://<cluster_url>/#/metrics/changefeeds/cluster`) for number of changefeed restarts. 
3.  Resume paused changefeed(s) with the job id (e.g. `RESUME JOB 681491311976841286;`).



