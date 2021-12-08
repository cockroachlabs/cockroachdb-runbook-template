# Alert: Node's CPU Anomaly (Hotspot)

### Purpose of this Alert

Unbalanced utilization of CockroachDB nodes in a cluster may negatively affect the cluster's performance and stability, with some nodes getting overloaded while others remain relatively underutilized. Potential causes of node hotspots are outlined in the "Hotspots" Section of [the common problems experienced by CockroachDB users](../most-common-problems/README.md).



------

### Monitoring Metric

```
sys.cpu.user.percent, sys.cpu.sys.percent
```



### Alert Rule

| Tier     | Definition                                                   |
| -------- | ------------------------------------------------------------ |
| CRITICAL | A node has a significantly higher CPU usage - 70% or higher above the cluster's median - for 1 hour<br/>and<br/>the node's CPU utilization is over 85% |
| WARNING  | A node has a significantly higher CPU usage  - 70% or higher above the cluster's median -  for 2 hours<br/>and<br/>the node's CPU utilization is over 70% |




### Alert Response

Confirm the Hotspot by observing the graphs in the CockroachDB Console and identify the likely cause. It could be application processing dynamic (spot processing of a high volume of activity for some key, e.g. user id, account number), or uneven distribution of data across ranges, or a skew in connection load balancing, etc.  Observe the following metrics in CockroachDB Console:

- Metrics --> Hardware Dashboard --> "CPU Percent" showing a significantly higher CPU utilization on an small number of nodes relatively to the cluster median. This would confirm a hotspot/overloaded node.
- Metrics --> SQL Dashboard --> "Open SQL Sessions" showing an uneven distribution of connections (sessions). If a Hot node has significantly more connections vs other nodes, this may explain the cause of the Hotspot. This would be a probably cause yet not a proof since connections could be idling.
- Metrics --> SQL Dashboard --> "Active SQL Statements" showing the true concurrency of the workload exceeding the cluster sizing guidance (no more than 4 active statements per vCPU). In combination with a possible connection distribution skew, this may be a lead to investigate.
- Advanced Debug --> Hot Ranges --> "All Nodes" may show the most active ranges in the cluster concentrated on a Hot node. This would suggest either an application processing hotspot or a schema design choice leading to poor data distribution across ranges.

To eliminate a Hotspot caused by a data distribution / design issue, review the Hotspot Prevention/Resolution section of [the common problems experienced by CockroachDB users](../most-common-problems/README.md).

