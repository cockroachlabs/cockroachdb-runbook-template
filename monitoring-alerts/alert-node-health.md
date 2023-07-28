# Alert: Heartbeat Latency

### Purpose of this Alert

Monitor the cluster health for early signs of instability.





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



The liveness checks reported by a node is inconsistent with the rest of the cluster.

Inconsistent Liveness check



------

### Alert Rule

| Tier     | Definition                                                   |
| -------- | ------------------------------------------------------------ |
| WARNING  | max cluster (liveness.livenodes) - min (liveness.livenodes) > 0 for 2 minutes |
| CRITICAL | max cluster (liveness.livenodes) - min (liveness.livenodes) > 0 for 5 minutes |



### Alert Response

The actual response varies depending on the alert tier, i.e. the severity of potential consequences.

- Check ....

- 









doing regular maintenance (upgrade, rehydrate, ...) you will also get these messages, so you need to control your monitoring alarms during maintenance.
