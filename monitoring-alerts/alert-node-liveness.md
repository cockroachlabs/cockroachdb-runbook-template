# Alert: Node Liveness

### Purpose of this Alert

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

