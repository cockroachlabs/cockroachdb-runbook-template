# Alert: Node Memory Utilization

### Purpose of this Alert

I node with high memory utilization is a cluster stability risk. Potential causes of high memory utilization are outlined in "Insufficient RAM" section of [the common problems experienced by CockroachDB users](../most-common-problems/README.md).



------

### Monitoring Metric

```
sys.rss
```



### Alert Rule

| Tier     | Definition                                      |
| -------- | ----------------------------------------------- |
| WARNING  | Node Memory Utilization exceeds 80% for 4 hours |
| CRITICAL | Node Memory Utilization exceeds 90% for 1 hour  |




### Alert Response

High memory utilization is a prelude to a node's OOM (process termination by the OS when the system is critically low on memory). OOM condition is not expected to occur if a CockroachDB cluster is provisioned and sized per Cockroach Labs guidance:

- All CockroachDB VMs are provisioned with [sufficient RAM](https://www.cockroachlabs.com/docs/stable/recommended-production-settings#memory).

