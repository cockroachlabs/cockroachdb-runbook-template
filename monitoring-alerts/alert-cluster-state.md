

**[THIS IS A SCRATCHPAD. WORK IN PROGRESS]**

# Alert: Heartbeat Latency

### Purpose of this Alert

Monitor the cluster health for early signs of instability.



WORK IN PROGRESS

------

### Monitoring Metric

```
liveness.heartbeatlatency
```

If this metric exceeds 1 sec,  it's a sign of instability. The recommended alert rule: warning if 0.5 sec,  critical if 3secs



### Alert Rule

| Tier     | Definition |
| -------- | ---------- |
| WARNING  |            |
| CRITICAL |            |

### Alert Response

< TODO >



--------------------



# Alert: Live Node Count Change

### Purpose of this Alert

Live node count change

<Formula doesn't make sense?>



------

### Alert Rule

| Tier     | Definition                                                   |
| -------- | ------------------------------------------------------------ |
| CRITICAL | min cluster (liveness.livenodes) - min cluster  (liveness.livenodes) as of 5 minutes ago  < 0 for 90 minutes |
| WARNING  | min cluster (liveness.livenodes) - min cluster  (liveness.livenodes) as of 2 minutes ago  < 0 for 90 minutes |



### Alert Response

The actual response varies depending on the alert tier, i.e. the severity of potential consequences.

- Check ....

- 

