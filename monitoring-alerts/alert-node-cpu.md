# Alert:  Hot CPU

### Purpose of this Alert

I node with a high CPU utilization (a.k.a. an "overloaded" node) has a limited ability to process the user workload and increases the risks of cluster instability. Potential causes of node CPU overload are outlined in the "Insufficient CPU" Section of [the common problems experienced by CockroachDB users](../most-common-problems/README.md).

------

### Monitoring Metric

```
sys.cpu.combined.percent-normalized, sys.cpu.host.combined.percent-normalized
```



### Alert Rule

| Tier     | Definition                                   |
| -------- | -------------------------------------------- |
| WARNING  | Node CPU Utilization exceeds 80% for 4 hours |
| CRITICAL | Node CPU Utilization exceeds 90% for 1 hour  |




### Alert Response

Confirm the high CPU utilization of cluster nodes.  Observe the following metrics in CockroachDB Console:

- Metrics --> Hardware Dashboard --> "CPU Percent" showing a high CPU utilization on cluster nodes
- Metrics --> SQL Dashboard --> "Active SQL Statements" showing the true concurrency of the workload, possibly exceeding the cluster capacity planning guidance of no more than 4 active statements per vCPU (core)

A persistently high CPU utilization of all nodes in a CockroachDB cluster suggests the current compute resources may be insufficient to support the user workload's concurrency requirements. If confirmed, the number of processors (vCPUs/cores) in the CockroachDB cluster needs to be adjusted to sustain the required level of workload concurrency.

For a prompt resolution, either [add cluster nodes](../routine-maintenance/node-add.md) or throttle down the workload concurrency, for example by reducing the number of concurrent connections, to not exceed 4 active statements per vCPU/core.

For more deliberate planning, review [cluster right-sizing, expansion strategy](../system-overview/_under-construction_.md).

For additional insights, review the "Insufficient CPU to support the scale of the workload" section of [the common problems experienced by CockroachDB users](../most-common-problems/README.md).





# Alert: Hot Node (Hotspot)

### Purpose of this Alert

Unbalanced utilization of CockroachDB nodes in a cluster may negatively affect the cluster's performance and stability, with some nodes getting overloaded while others remain relatively underutilized. Potential causes of node hotspots are outlined in the "Hotspots" section of [the common problems experienced by CockroachDB users](../most-common-problems/README.md).



------

### Monitoring Metric

```
sys.cpu.combined.percent-normalized, sys.cpu.host.combined.percent-normalized
```



### Alert Rule

| Tier    | Definition                                                   |
| ------- | ------------------------------------------------------------ |
| WARNING | The MAX CPU utilization across nodes exceeds the cluster's MEDIAN CPU utilization by 30 for 2 hours |


### Alert Response

Confirm the Hotspot by observing the graphs in the CockroachDB Console and identify the likely cause. It could be application processing dynamic (spot processing of a high volume of activity for some key, e.g. user id, account number), or uneven distribution of data across ranges, or a skew in connection load balancing, etc.  Observe the following metrics in CockroachDB Console:

- Metrics --> Hardware Dashboard --> "CPU Percent" showing a significantly higher CPU utilization on an small number of nodes relatively to the cluster median. This would confirm a hotspot/overloaded node.
- Metrics --> SQL Dashboard --> "Open SQL Sessions" showing an uneven distribution of connections (sessions). If a Hot node has significantly more connections vs other nodes, this may explain the cause of the Hotspot. This would be a probably cause yet not a proof since connections could be idling.
- Metrics --> SQL Dashboard --> "Active SQL Statements" showing the true concurrency of the workload exceeding the cluster sizing guidance (no more than 4 active statements per vCPU). In combination with a possible connection distribution skew, this may be a lead to investigate.
- Advanced Debug --> Hot Ranges --> "All Nodes" may show the most active ranges in the cluster concentrated on a Hot node. This would suggest either an application processing hotspot or a schema design choice leading to poor data distribution across ranges.

To eliminate a Hotspot caused by a data distribution / design issue, review the Hotspot Prevention/Resolution section of [the common problems experienced by CockroachDB users](../most-common-problems/README.md).

