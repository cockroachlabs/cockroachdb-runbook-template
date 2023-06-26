# Data Availability: Disruptions Control

### Overview of Hardware Failures Leading to Data Service Disruptions

This article is focused on mitigation of CockroachDB data access disruptions due to faults in underlying platforms - either definitive, unambiguously detectable fail-stop failures, or so called "gray" failures that mask problems and confuse fault detectors.

Gray failures are subtle, non-binary faults in the underlying hardware that result in some components of CockroachDB observing that the system is unhealthy, while other components observe the system is healthy. Handling of differential observability due to gray failures requires a high degree of sophistication in the implementation of a distributed system. This article provides insights into hardened implementation of edge-case underlying failures by CockroachDB as of v23.1. Unmitigated gray failures may result in extended data service disruptions. CockroachDB provides tunable configuration settings enabling operators to meet the required availability SLAs across fail-stop and gray underlying failures.

Gray failures discussed in this article include:

- [Asymmetric network partitions](#asymmetric-network-partitions)
- [Partial network partitions](#partial-network-partitions)
- [Disk Stalls](#disk-stalls)

On the other side, the most common - fail-stop software and hardware failures - are handled by CockroachDB robustly and predictably out of the box. In the vast majority of use cases, no cluster configuration adjustments are necessary to meet the required availability SLAs. An automatic restoration of access to data affected by a node's fail-stop failure typically takes a few seconds, without operator's intervention.

Examples of fail-stop, i.e. complete node failure, are:

- Underlying VM ceases to exist (crashes)
- VM is running but CockroachDB node process ceases to exist (terminated)
- VM and CockroachDB node process are running but the network is completely down (physical wire or equipment damage) 



### Range Replication Insights

The operator should be familiar with CockroachDB [replication layer](https://www.cockroachlabs.com/docs/stable/architecture/replication-layer.html) insights and understand the CockroachDB architecture terms [range](https://www.cockroachlabs.com/docs/stable/architecture/glossary.html#range), [replica](https://www.cockroachlabs.com/docs/stable/architecture/glossary.html#replica), [gateway](https://www.cockroachlabs.com/docs/stable/architecture/life-of-a-distributed-transaction.html#overview), [leaseholder](https://www.cockroachlabs.com/docs/stable/architecture/glossary.html#leaseholder), [raft](https://www.cockroachlabs.com/docs/stable/architecture/glossary.html#raft-protocol), [raft leader](https://www.cockroachlabs.com/docs/stable/architecture/glossary.html#raft-leader).

##### Expiration-based vs. Epoch-based Leases

CockroachDB implements two types of leases - [expiration-based](https://www.cockroachlabs.com/docs/stable/architecture/replication-layer.html#expiration-based-leases-meta-and-system-ranges) and [epoch-based](https://www.cockroachlabs.com/docs/stable/architecture/replication-layer.html#epoch-based-leases-table-data). By default, expiration-based leases are used for meta and liveness ranges while epoch-based leases are used for other system ranges and table data ranges.

Epoch-based lease implementation relies on system ranges that, in turn, rely on expiration-based leases.

Expiration-based leases have better availability characteristics than epoch-based leases. They do not rely on a single liveness range for availability and will regularly verify that the range lease is functional by performing a Raft roundtrip to extend the lease. Unlike epoch-based leases, a *liveness range failure does not affect expiration-based leases on other nodes*, removing the liveness range as a temporary single point of failure. With epoch-based leases, the liveness leaseholder loss will have a cluster-wide implications (albeit temporary), while with expiration-based leases only leases owned by the failed node are affected.

The expiration-based leases have shorter data availability restoration time in all leaseholder failure scenarios such as crashes, network partitions and disk stalls.

Epoch-based leases are implemented as a performance optimization over expiration-based leases. Epoch-based leaseholders do not renew their own leases, which reduces the amount of network traffic and CPU overhead of extra Raft processing, while still providing lease tracking for individual ranges. Expiration-based leases expire at a [lease interval](https://www.cockroachlabs.com/docs/stable/architecture/replication-layer.html#important-values-and-timeouts) (6 seconds in v23.1 and later, 9 seconds in v22.2 and earlier) and need to interact with the Raft quorum to be extended.

For a CockroachDB writes and consistent reads to work, the following is required for ranges with either expiration-based or epoch-based leases:

- A gateway node `N[gw]` for the query needs access to data range leaseholder node `N[dl]`
- The data range leaseholder `N[dl]` must be connected to the Raft leader `N[dl-rl]` of the data range group and the Raft leader `N[dl-rl]` must be connected to a quorum of its replicas.

For ranges with epoch-based leases there is, however, an additional requirement:

1) The data range leaseholder `N[dl]`, to be "alive" and keep its leases, needs to *write* to its system liveness range. This means the system liveness range leaseholder `N[ll]` needs to be accessible, and also connected to `N[ll-rl]` and a quorum of its replicas.

##### Range Leaseholder vs. Raft Leader

Most of the time, the range leaseholder and its Raft leader are on the same node, but not necessarily. Raft is agnostic of CockroachDB leases. Raft leader and leaseholder elections may proceed independently.

Raft leadership follows the leaseholder when the two are on different nodes. For all active (not quiescent) ranges, the Raft leader checks whether it is the leaseholder on every raft tick (~500ms as of v23.1, ~200ms in prior versions). If it finds that it is not, the Raft leadership is transferred to the leaseholder node. For quiescent ranges, the location of the Raft leader does not matter.



### Controls of Data Availability Disruptions

*Data availability disruption* refers to a momentary loss of access to CockroachDB managed data due to a fault in an underlying computing platform. 

As of CockroachDB v23.1, an operator can mitigate the business impact of a data service disruption by controlling its characteristics:

- [Blast radius](#blast-radius-control), refers to the scale of data service disruption, i.e. whether a disruption affects access to all or only some data;
- [Maximum duration](#maximum-duration-controls), refers to the longest possible time interval, in seconds, during which some or all data remains  inaccessible.



#### Blast Radius Control

The following blast radius controls apply across all fail-stop and gray failure scenarios!

##### Use of Expiration-based Leases for Data Ranges

Version 23.1 introduces an [experimental cluster setting](https://github.com/cockroachdb/cockroach/pull/94457) `kv.expiration_leases_only.enabled`, which uses expiration-based leases for all ranges, including the data ranges (that otherwise use epoch-based leases). This setting is marked as experimental because it may have performance implications that need to be measured and acknowledged for each customer workload.

***The principle benefit*** of enabling expiration-based leases for data ranges is to limit the blast radius of a single node failure to only a partial temporary loss of access to data leaseholders on the failed node, eliminate a potential temporary cluster-wide service outage. Conversely, if epoch-based leases were used, a failure of a leaseholder of the liveness system range would lead to a temporary loss of access to all cluster data.

***To enable expiration leases***, set the cluster setting `kv.expiration_leases_only.enabled = true`. This configuration change is non-disruptive and is processed online. The leases are asynchronously switched to the appropriate type via the replicate queue. This can take about 10 minutes. Leases can be monitored via the metrics `leases.expiration` and `leases.epoch`. When all leases switch to expiration-based, the `leases.epoch` metric drops to 0.

***A performance overhead*** associated with using expiration-based leases is principally due to a required Raft roundtrip to extend a lease once every 1/2 of the lease interval (6/2=3 seconds), which adds some network IO, CPU processing, and disk IO. This overhead would be more pronounced with more ranges per node. Performance characterization tests conducted by Cockroach Labs engineering show that this overhead does not materially affect workload performance when nodes have under 5,000 data ranges but may become objectionable in clusters with high storage density over 10,000 ranges per node.



#### Maximum Duration Controls

Precisely calculating the maximum disruption interval is not possible because it depends on non-trivial interactions between many factors, including randomized time intervals (used, for example, in Raft leader elections). These factors include network round trip latencies, network timeouts, connection timeouts, Raft elections timeouts, re-proposal timeouts, cache invalidation, retry policies, etc. The estimates below are empirical, based on actual measurements during performance characterization testing in CockroachDB engineering lab.

With v23.1 default [lease management intervals and timeouts](https://www.cockroachlabs.com/docs/stable/architecture/replication-layer.html#important-values-and-timeouts) (lease interval 6 seconds), and default [COCKROACH_ENGINE_MAX_SYNC_DURATION]( https://www.cockroachlabs.com/docs/stable/cluster-setup-troubleshooting.html#disk-stalls) (20 seconds), the worst-case data access disruption times are estimated as:

| Failure Scenario                     | Estimated Worst-case Disruption with expiration-based leases | Estimated Worst-case Disruption with epoch-based leases | Explanation (with expiration-based leases)                   |
| ------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------- | ------------------------------------------------------------ |
| Node failure                         | 7 seconds                                                    | 9 seconds                                               | The lease expiration time (6 seconds), plus 2 network roundtrips (a Raft leader and range leaseholder elections). In practice the network roundtrips are orders of magnitude smaller than the lease expiration, so the estimate may be closer to 6 seconds. |
| Network partition                    | 11 seconds                                                   | 15 seconds                                              | The worst-case randomized Raft election timeout fires after about 4 seconds, rather than the lease expiration time at 6 seconds, it's reasonable to use 5 seconds. Then there is waiting for various network timeouts -- RPC timeouts, network timeouts, connection timeouts, re-proposal timeouts, etc. In particular, the RPC heartbeat timeout is currently set to 6 seconds to avoid clusters falling apart under heavy load, and reducing it carries significant stability risks. |
| Disk Stall (database storage volume) | 23 seconds                                                   | 27 seconds                                              | By default, a database storage volume stall is detected within 20 seconds. By then the lease would have expired and moved before the node is killed, so there is no wait for the lease expiration afterwards. However, a KV request has to fail, get retried, find the new leaseholder, possibly acquire a new lease. This can add up to 3 additional seconds. |
| Disk Stall (message log volume)      | 33 seconds                                                   |                                                         | By default, a message log volume stall is detected within 30 seconds. By then the lease would have expired and moved before the node is killed, so there is no wait for the lease expiration afterwards. However, a KV request has to fail, get retried, find the new leaseholder, possibly acquire a new lease. This can add up to 3 additional seconds. |

> `!`  If the default epoch-based leases are configured for data ranges, the estimated worst-case availability disruption times increase by a couple of seconds.



##### Faster Leaseholder Recovery in all Failure Scenarios

The default liveness lease interval for both expiration-based and epoch-based leases [has been reduced](https://www.cockroachlabs.com/docs/releases/v23.1#kv-layer) from 9 seconds to 6 seconds in v23.1 and later. In comparison with v22.2 and earlier versions, this results in 1/3 reduction of range unavailability following its leaseholder loss.

##### Shorter Network Timeouts

Several network timeouts (not user visible or settable) have been reduced in v23.1 generally from 3 - 6 second range by 1 or 2 seconds. This speeds up recoveries from all types of network partitions.



### Fail-stop Node Failures

A *complete* CockroachDB cluster node liveness failure results in that node marked as `suspect` within a few seconds. If liveness is not restored within a few minutes, the affected node is subsequently marked as `dead`. Availability of the data momentarily affected by a failed node is automatically continued by [transferring the leaseholders of the affected ranges](https://www.cockroachlabs.com/docs/stable/architecture/replication-layer.html#how-leases-are-transferred-from-a-dead-node) to healthy / surviving nodes of the cluster.

The maximum data unavailability interval is estimated in the table [above](#maximum-duration-controls) (see "Node failure"). No custom tuning is advised. 



### Gray Failures

#### Partial network partitions

A partial network partitioning is a network fault that disrupts communication among some but not all nodes in a cluster. 

Partial network partitions are exceedingly rare in redundantly provisioned networks. The most common cause of a partial network partition is a misconfiguration error in firewall rules and/or routing tables.

The vast majority of the situations arising from partial network partitions are orderly/automatically handled by CockroachDB. However, there are esoteric situation that may require operator's manual intervention.

Epoch-based leases can become non-functional under certain forms of partial network partitions if the leaseholder can't reach the Raft leader but is able to heartbeat its liveness record. Expiration-based leases are less vulnerable to this, but there are known cases where partial network partitions may result in cluster being "stuck" until a partial network partition is either resolved or turned into a full network partition.

If a CockroachDB cluster gets "stuck" due to a partial network partitioning, it will continue retrying faulty network connection indefinitely. As soon as the network fault is cleared, the cluster will automatically resume normal operations.

The plan of record is to address all known issues and ensure that partial network partitions can't cause a prolonged service outage by v23.2.



#### Asymmetric network partitions


A partial network partitioning is a network fault whereby a node can send requests to a peer node, but can not receive from that node.

As of CockroachDB v23.1, an [enhancement](https://github.com/cockroachdb/cockroach/pull/94778) eliminates the need for special handling of asymmetric partitions. By implementing a mandatory bidirectional RPC connections, all edge cases pertinent to asymmetric network partitions are promoted to partial of complete network partitions.



#### Disk Stalls

As of CockroachDB v22.2 all issues interfering with accurate disk stall detection have been resolved ([search](https://www.cockroachlabs.com/docs/releases/v22.2.html#v22-2-4-bug-fixes) for "stall")

The maximum data unavailability intervals due to [stalls](https://www.cockroachlabs.com/docs/stable/cluster-setup-troubleshooting#disk-stalls) of data volume or message log volume are estimated in the table [above](#maximum-duration-controls) (see "Disk Stall").

***The disk stall detection can be tuned*** to meet maximum data availability disruption using the following controls:

- `COCKROACH_ENGINE_MAX_SYNC_DURATION_DEFAULT` environment variable or its equivalent cluster setting `storage.max_sync_duration` - sets the max allowable time an IO operation can be waiting in the storage layer. Default is 20 seconds. If both the cluster setting and 
- `COCKROACH_LOG_MAX_SYNC_DURATION` environment variable (there is no equivalent cluster setting) - sets the max allowable time an IO operation can be waiting in the message logging framework. Default is 20 seconds.

These two settings should always be set to the same value.

Setting these settings to a low value (less than 5 seconds) should be avoided because it can result in a "false positive" disk stall detection, triggering an unnecessary node panic.

***For example***, if the maximum data unavailability must be limited to 20 seconds, set both `COCKROACH_ENGINE_MAX_SYNC_DURATION_DEFAULT` and `COCKROACH_LOG_MAX_SYNC_DURATION` to 10 seconds.
