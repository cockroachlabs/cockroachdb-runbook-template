# Alert: Node's Memory Utilization

### Purpose of this Alert

Unbalanced utilization of CockroachDB nodes in a cluster may negatively affect the cluster's performance and stability, with some nodes getting overloaded while others remain relatively underutilized. Potential causes of node hotspots are outlined in the "Hotspots" Section of [the common problems experienced by CockroachDB users](../most-common-problems/README.md).



Health rule violation event. Node is low on memory.



------

### Alert Rule

```
Node Memory Utilization exceeds 90% for 1 hour
```


### Alert Tier

```
WARNING
```


### Alert Response

```
<Response varies depending on the Tier (severity of potential consequences)>
```

