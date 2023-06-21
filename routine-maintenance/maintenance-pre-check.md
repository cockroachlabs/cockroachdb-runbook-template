# Procedure: Cluster Health Pre-Check

### About this Procedure

Executing this cluster health check procedure should be a pre-requisite prior to commencing any [routine maintenance](./) procedure. 

This multi-point cluster health check covers the following:

- **Non-overlapping maintenance procedures**. Eliminate risks related to multiple maintenance procedures being executed concurrently.
- **Nodes versions consistency**. Ensure the last major cluster upgrade has been finalized.
- **Nodes membership and availability status**. Ensure build consistency and that all nodes are live and available.
- **Replication state**. Ensure that data on the cluster is neither under-replicated nor unavailable.
- **Underlying hardware health**. Ensure the health of underlying hardware resources.

The procedure is largely the same across CockroachDB version. Any version-specific steps are called out explicitly. Each point of the cluster health check includes step-by-step guidance.



### Procedure Steps



#### 1. Non-overlapping maintenance procedures

------

Confirm no other operating procedure is currently running in the cluster or is incomplete. If another [routine maintenance](./) procedure is in progress, do not proceed until the previous procedure completes successfully. 



#### **2. Avoid maintenance during active replica rebalancing**

----

Operators should not commence routine maintenance procedures while the cluster is actively rebalancing replicas. To determine if active rebalancing is in progress, observe the following graphs in DB Console - [ranges](https://www.cockroachlabs.com/docs/stable/ui-replication-dashboard#ranges), [snapshot data received](https://www.cockroachlabs.com/docs/stable/ui-replication-dashboard#snapshot-data-received), [receiver snapshots queued](https://www.cockroachlabs.com/docs/stable/ui-replication-dashboard#receiver-snapshots-queued). An operator can [control the rebalancing speed](./change-rebalance-rate.md). If the graphs show active replica rebalancing, wait until rebalancing settles into a steady state.



#### 3. Nodes versions consistency

------

Confirm the last major release upgrade was successfully finalized. *Run the check query provided* in the [CockroachDB version upgrade article](./release-upgrade.md#finalizing-a-major-release-upgrade). If the query returns `false`, the last major upgrade has not been finalized. No maintenance procedures should be performed until the upgrade is finalized. 



#### 4. Nodes membership and availability status

------

Execute the following command:

```
cockroach node status [other options]
```

The output (redacted for clarity) includes a row for each of the cluster nodes:

```shell
% cockroach node status --insecure
  id | ... |  build  | ... |         locality         | is_available | is_live
-----+-...-+---------+-...-+--------------------------+--------------+---------
   1 | ... | v23.1.3 | ... | region=us-east1,az=b     | true         | true
   2 | ... | v23.1.3 | ... | region=us-east1,az=c     | true         | true
   3 | ... | v23.1.3 | ... | region=us-east1,az=d     | true         | true
   4 | ... | v23.1.3 | ... | region=us-west1,az=a     | true         | true
   5 | ... | v23.1.3 | ... | region=us-west1,az=b     | true         | true
   6 | ... | v23.1.3 | ... | region=us-west1,az=c     | true         | true
   7 | ... | v23.1.3 | ... | region=europe-west1,az=b | true         | true
   8 | ... | v23.1.3 | ... | region=europe-west1,az=c | true         | true
   9 | ... | v23.1.3 | ... | region=europe-west1,az=d | true         | true
```

Check the output columns:

​	**is_available:** The node is part of quorum

​	**is_live:** The node is up and running

Before proceeding with maintenance, ensure that all cluster nodes are live  and available by confirming that ***all*** values in columns `is_available` and `is_live`  are `true`. If any value in these columns is `false` - the *cluster is not in a healthy state* and should be investigated prior to proceeding with maintenance.



------

Execute the following command:

```shell
cockroach node status --decommission [other options]
```

The  `--decomission` flag reports the following (some columns redacted for clarity):

```shell
% cockroach node status --decommission --insecure
  id | ... |  build  | ... |         locality         | is_available | is_live | ... | is_decommissioning | membership | is_draining
-----+-...-+---------+-...-+--------------------------+--------------+---------+-...-+--------------------+------------+-------------
   1 | ... | v23.1.3 | ... | region=us-east1,az=b     | true         | true    | ... | false              | active     | false
   2 | ... | v23.1.3 | ... | region=us-east1,az=c     | true         | true    | ... | false              | active     | false
   3 | ... | v23.1.3 | ... | region=us-east1,az=d     | true         | true    | ... | false              | active     | false
   4 | ... | v23.1.3 | ... | region=us-west1,az=a     | true         | true    | ... | false              | active     | false
   5 | ... | v23.1.3 | ... | region=us-west1,az=b     | true         | true    | ... | false              | active     | false
   6 | ... | v23.1.3 | ... | region=us-west1,az=c     | true         | true    | ... | false              | active     | false
   7 | ... | v23.1.3 | ... | region=europe-west1,az=b | true         | true    | ... | false              | active     | false
   8 | ... | v23.1.3 | ... | region=europe-west1,az=c | true         | true    | ... | false              | active     | false
   9 | ... | v23.1.3 | ... | region=europe-west1,az=d | true         | true    | ... | false              | active     | false
```

Check the output columns:

​	**is_decommissioning:** The node is undergoing a [decommissioning process](./node-remove.md).

​	**is_draining:** The node is either draining or [shutting down](./node-stop.md).

Before proceeding with maintenance, ensure that there are no cluster nodes in decommissioning state, or draining, or shutting down, by confirming that ***all*** values in columns  `is_decommissioning` and `is_draining` are `false`. If any value in these two columns is `true` - the *cluster is not in a healthy state* and should be investigated prior to proceeding with maintenance.



----

##### Critical nodes

Starting with v23.1, operators can use a [critical nodes HTTP endpoint](https://www.cockroachlabs.com/docs/stable/monitoring-and-alerting.html#critical-nodes-endpoint) to get more insights into the cluster state. A node is "critical" if some ranges can lose their quorum should that node become unavailable.

The critical nodes endpoint returns two types of objects in JSON format - `criticalNodes` that lists the currently "untouchable" nodes and the `report` objects with detailed insights.

```shell
% curl -X POST http://localhost:8080/_status/critical_nodes
{
  "criticalNodes": [
  ],
  "report": {
    "underReplicated": [
    ],
    "overReplicated": [
    ],
    "violatingConstraints": [
    ],
    "unavailable": [
    ],
    "unavailableNodeIds": [
    ]
  }
}
```

A healthy cluster is represented above, with no critical nodes and no report entries. 



#### 5. Replication state

Execute the following command:

```shell
cockroach node status --ranges [other options]
```

The  `--ranges` flag reports the following (some columns redacted for clarity):

```shell
% cockroach node status --ranges --insecure
  id | ... |  build  | ... |         locality         | is_available | is_live | ... | ranges_unavailable | ranges_underreplicated
-----+-...-+---------+-...-+--------------------------+--------------+---------+-...-+--------------------+------------------------
   1 | ... | v23.1.3 | ... | region=us-east1,az=b     | true         | true    | ... |                  0 |                      0
   2 | ... | v23.1.3 | ... | region=us-east1,az=c     | true         | true    | ... |                  0 |                      0
   3 | ... | v23.1.3 | ... | region=us-east1,az=d     | true         | true    | ... |                  0 |                      0
   4 | ... | v23.1.3 | ... | region=us-west1,az=a     | true         | true    | ... |                  0 |                      0
   5 | ... | v23.1.3 | ... | region=us-west1,az=b     | true         | true    | ... |                  0 |                      0
   6 | ... | v23.1.3 | ... | region=us-west1,az=c     | true         | true    | ... |                  0 |                      0
   7 | ... | v23.1.3 | ... | region=europe-west1,az=b | true         | true    | ... |                  0 |                      0
   8 | ... | v23.1.3 | ... | region=europe-west1,az=c | true         | true    | ... |                  0 |                      0
   9 | ... | v23.1.3 | ... | region=europe-west1,az=d | true         | true    | ... |                  0 |                      0
```

Check the output columns:

​	**ranges_unavailable:** The number of currently unavailable ranges

​	**ranges_underreplicated:** The number of currently under replicated ranges

Before proceeding with maintenance, ensure that there are no unavailable or under replicated ranges, by confirming that ***all*** values in columns  `ranges_unavailable` and `ranges_underreplicated` are `0`. If any value in these two columns is not `0` - the *cluster is not in a healthy state* and should be investigated prior to proceeding with maintenance.



#### 6. Underlying hardware health

----

Confirm no underlying hardware resource is utilized above operator's established IT practices to maintain SLA for latency-sensitive data services. Prior to commencing a maintenance procedure, ensure that:

​	**CPU:** VM's (server) CPU does not exceed 70% utilization

​	**Memory:** VM's (server) memory utilization does not exceed 70%

​	**Storage capacity:** There is ample free space on the data volume for CockroachDB store, i.e. the volume is consistently less than 70% full.

Thresholds are workload and use-case dependent. Operators can use these recommendations as a starting point. DB Console's hardware page provides the necessary insight into the CockroachDB VMs' hardware utilization.



### Prevent Unnecessary Data Transfers

The cluster setting [time_until_store_dead](https://www.cockroachlabs.com/docs/stable/cluster-settings.html#setting-server-time-until-store-dead) controls the "grace period" before a "suspect" Cockroach DB node (that stops heart beating its liveness) transitions into a "dead" state. The default is 5 minutes.

When a CRDB node is brought down for maintenance and the downtime exceeds `server.time_until_store_dead`, the node will be marked "dead" and the cluster will start self-healing by rebuilding (up-replicating) the under-replicated replicas. This utilizes all system resources - CPU, network, disk, memory. When the node maintenance completes and the node comes back online and rejoins the cluster, it still has its replicas that now have to be reconciled via Raft, with excess replicas discarded - also resource intensive process.

These resource-consuming data transfers between nodes can be avoided by *temporarily* increasing the `server.time_until_store_dead`, for the duration of a cluster maintenance. The ranges would stay in place, just the range leases are moved to online nodes while a node is down for maintenance.

Operators can extend the `server.time_until_store_dead` so it exceeds a node downtime by a few minutes. For example, if a node downtime during maintenance can be up to 10 minutes, an operator can set the cluster setting to 15 minutes:

```
SET CLUSTER SETTING server.time_until_store_dead = '15m';
```

After the cluster maintenance completes, and if the `server.time_until_store_dead` has been temporarily extended, change the setting back to its default:

```
RESET CLUSTER SETTING server.time_until_store_dead;
```

> 
>
> ✅ Operators should weigh in the benefits of this optimization against an increased risk of an exposure to a double node fault. This is a risk mitigation decision.
>
> If a node maintenance takes under 10 minutes, the increased probability of another node fault during a node maintenance downtime may be negligible. However keeping a cluster in an under-replicated state for 30 - 60 minutes node downtime, particularly if a cluster is configured with the default replication factor RF=3, maybe objectionable. If another node fails during this time, it will result in a temporary quorum loss, until the quorum is restored. While even with RF=3, a double fault will not result in "data loss", it would mean a data service disruption.
>
> The increased risk of a double fault is generally not a concern with RF=5 and higher.
>
> 

