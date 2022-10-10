# Alert: Node CPU High Utilization

### Purpose of this Alert

I node with a high CPU utilization (a.k.a. an "overloaded" node) has a limited ability to process the user workload and increases the risks of cluster instability.



------

### Monitoring Metric

```
sys.cpu.user.percent, sys.cpu.sys.percent
```



### Alert Rule

| Tier     | Definition                                      |
| -------- | ----------------------------------------------- |
| WARNING  | Node CPU Utilization exceeds 85% for 2 hours    |
| CRITICAL | Node CPU Utilization exceeds 90% for 60 minutes |




### Alert Response

Confirm the high CPU utilization of cluster nodes.  Observe the following metrics in CockroachDB Console:

- Metrics --> Hardware Dashboard --> "CPU Percent" showing a high CPU utilization on cluster nodes
- Metrics --> SQL Dashboard --> "Active SQL Statements" showing the true concurrency of the workload, possibly exceeding the cluster capacity planning guidance of no more than 4 active statements per vCPU (core)

A persistently high CPU utilization of all nodes in a CockroachDB cluster suggests the current compute resources may be insufficient to support the user workload's concurrency requirements. If confirmed, the number of processors (vCPUs/cores) in the CockroachDB cluster needs to be adjusted to sustain the required level of workload concurrency.

For a prompt resolution, either [add cluster nodes](../routine-maintenance/node-add.md) or throttle down the workload concurrency, for example by reducing the number of concurrent connections, to not exceed 4 active statements per vCPU/core.

For more deliberate planning, review [cluster right-sizing, expansion strategy](../system-overview/_under-construction_.md).

For additional insights, review the "Insufficient CPU to support the scale of the workload" section of [the common problems experienced by CockroachDB users](../most-common-problems/README.md).

