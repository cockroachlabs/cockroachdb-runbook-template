# Alert: Node's CPU Utilization

### Purpose of these Alerts

<Why this rule, what it prevents>

Health rule violation event.

1) High CPU

2) Monitoring for hot spots -  a single node has significantly higher CPU usage compared to the cluster median. Check for anomalies on that node. Hot ranges or processing hotspot.



------

#### Alert 1 Rule

| Tier     | Definition                                      |
| -------- | ----------------------------------------------- |
| CRITICAL | Node CPU Utilization exceeds 85% for 30 minutes |
| WARNING  | Node CPU Utilization exceeds 80% for 1 hour     |




#### Alert 1 Response

<Response varies depending on the Tier (severity of potential consequences)>



------

#### Alert 2 Rule

| Tier     | Definition                          |
| -------- | ----------------------------------- |
| CRITICAL | cpu_combined_percent_normalized ... |
| WARNING  |                                     |




#### Alert 2 Response

<Response varies depending on the Tier (severity of potential consequences)>

