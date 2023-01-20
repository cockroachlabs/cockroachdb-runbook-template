# Procedure:  Remove (Decommission) Node(s)

### About this Procedure

Removal of nodes from a CockroachDB cluster is referred to as nodes' [decommissioning](https://www.cockroachlabs.com/docs/stable/node-shutdown?filters=decommission#decommission-the-node).

This procedure is used as a step of larger regular maintenance procedures or as an emergency procedure to resolve an operational issue.

Examples of regular on-line maintenance procedures relying on node decommission are CockroachDB *cluster contraction*/capacity reduction, [*cluster topology change*](./cluster-region-migrate.md), rolling repaving of cluster VMs for security reasons, replacement or *hardware change* in the CockroachDB node's underlying VM or bare metal server.

This procedure may be used for emergency repairs, for example to remove of a running node from the cluster when an underlying *hardware failure* or platform *configuration errors* are observed. 

> ✅ The node decommission command can be run *from any* **running** *cluster node*. The target node(s), i.e. node(s) being decommissioned, could be any cluster node(s) - self or other - and they could be *in any state* - running, stopped, or even no longer existing


> ✅ A CockroachDB cluster will *not decommission a node automatically*, under any circumstances. Decommissioning nodes is strictly an operator's prerogative. In particular, a failed node that was declared "dead" and whose hardware was removed, is technically still a member of the cluster until it is removed from the cluster by an operator's action. "Dead" nodes that have not been decommissioned will be continually gossiped and probed by other cluster nodes, in anticipation that a "dead" node may recover and rejoin the cluster.

> ✅ DO NOT circumvent the proper decommission procedure by terminating a node process. For example, in an attempt to expedite a node removal. Terminating a node process, even with a prior drain, leaves the cluster with under-replicated ranges and may lead to a loss of quorum.



### Node Decommission vs. Node Drain (Shutdown)

Decommission and [Drain](./node-stop.md) are functionally related, yet distinctly different and meant to be used in different situations.

A *node drain* is a precursor to stopping the node which is then expected to rejoin the cluster in a near term and reconnect its replicas to their raft groups. Drain transfers the range *leaseholders* (and raft groups' leaderships) away from the drained node and closes all SQL connections to the node, but still keeps data replicas participating in their raft consensus groups.
A *node decommission* implies that the node is not going to return to the cluster. During a decommission, the decommissioning node transfers the replicas of all its ranges to other nodes. In the final phase, the decommission executes the [drain steps](./node-stop.md#how-node-drain-is-implemented). Since at that time there should be no replicas on that node, draining of the leaseholders is a no-op. The drain steps are performed solely as a final attempt to facilitate an orderly failover of client SQL connections.

A *node decommission* is a safe operation from the standpoint of maintaining the cluster's fault tolerance level. A decommissioning action does not allow under-replicated ranges at any point. On the other hand, a *node drain* (which is a precursor to a node shutdown), if followed by a node process stop, leaves some ranges under-replicated.

> ✅ Stopping a node without decommissioning first, even if stopped after an orderly drain, will leave ranges under-replicated.

A *node decommission* includes data transfers (raft snapshots) in the amount of data stored on the decommissioning node - commonly hundreds of GBs to TBs. A *node drain* consists predominantly of light-weight metadata operations. Therefore, a node decommission should be expected to take significantly longer than a node drain.



### Node Decommission Implementation Highlights

- To maintain the designed replication factor (RF) at all times during node decommission, a new replica for each range on a decommissioning node is added to some other node in the cluster via a raft snapshot. 
- Each node (store) can send only 1 snapshot at a time.
- Any node (store) can only be receiving 1 snapshot at a time.
- Range replica snapshots are sent by nodes that are range's leaseholders.
- The decommissioning node will have a longer queue of snapshot work than other nodes. This is because each non-decommissioning node in the cluster will have to send snapshots for the subset of its leaseholders that have a replica on the decommissioning node, but the decommissioning node will always have to send snapshots for *all* of its leaseholders.



### Node Decommission Planning 

Node(s) decommission, by definition, results in an increased demand for processing and storage capacity on all nodes remaining in the cluster. The operator needs to ensure that the remaining nodes have sufficient compute and storage resources to process more workload and accommodate more data replicas.

> ✅ When designing various maintenance procedures that leverage node add/node remove sequence for on-line node swap, be sure to [add new nodes](./node-add.md) first, allow the data to rebalance, then decommission as a final step. 



### Should Nodes be Manually Drained before Decommission?

TLDR; **No**. Normally there is no need to drain nodes before decommissioning, as long as the decommissioning [procedure is robustly implemented](./node-remove.md#procedure-steps) and coordinated with configuration parameters of other components, particularly load balancers and connection pools.

**Considerations**

Since the snapshots for relocating replicas from the decommissioning node(s) are sent from the leaseholders, the decommissioning nodes will be sending the snapshots for *all* of its replicas, thus forming a potentially long queue of snapshots to send.

If the nodes are pre-drained before decommission, the leaseholders will be distributed evenly across the *remaining* nodes, allowing more distributed snapshot senders, which should result in a reduction of the elapsed time of the decommissioning. However that comes at the expense of more load on the remaining cluster nodes, which in general is undesirable. And since the nodes are receiving one snapshot at a time, a reduction in the elapsed time is under 15% based on CRL's nightly regression tests data. The relatively modest elapsed time benefit is generally not a good justification for an increased load on the remaining server during the decommission and why pre-draining is not advised under normal operating conditions.

**When to Pre-Drain**

A manual pre-drain may be done before decommissioning a malfunctioning node as a part of some emergency procedure. Particularly when decommissioning an overloaded node that doesn't have CPU or Disk IO headroom.

> ✅ If a manual node drain step is integrated into a decommissioning procedure, ensure that under no circumstances the node process is stopped after a manual drain. Shutting down nodes before decommissioning may result in under-replicated ranges or even a loss of quorum.
>
> Also beware that if a pre-drain is done, the operator effectively loses the ability to orderly "cancel" the decommission in progress with [`cockroach node recommission`](https://www.cockroachlabs.com/docs/stable/create-external-connection.html) because there is no online method to "un-drain" (get the leaseholders back to drained node) without a node restart. 



### Procedure Steps

1. Prior to executing this procedure for the first time, an operator is reminded to [set the cluster's *rebalance and recovery rates*](./change-rebalance-rate.md) to the value deemed [optimal](./change-rebalance-rate.md#considerations-for-setting-the-max-rates). Higher rates allow the cluster to complete transient data transfers and reach the steady state faster.

   

2. Ensure that the Connection Pool's maximum connection life is set to approximately the elapsed time to decommission the node(s) or 30 minutes, whichever is larger. 

   

3. Remove the node(s) being decommissioned from the *load balancers*' configuration.

   

4. Decommission the node(s). A single command

   `cockroach node decommission <space separated list of node ids> --host=…`

   decommissioning all nodes at once is expected to work best. Alternatively decommission 1 node at a time, but ensure that only one `cockroach node decommission` command runs at a time.

   

5. Observe the status:

   ```
    id | is_live | replicas | is_decommissioning | is_draining 
   +---+---------+----------+--------------------+-------------+
     4 | false   |        0 |       true         |    true   
   (1 row)
   ```

   

6. `cockroach node status <id> --all` or observe the node(s)' status in DB Console.

   Confirm that decommissioned node(s)' status is  DECOMMISSIONED. If a node status is "DECOMMISSIONING", some maintenance operations, such as release upgrades, can not proceed.

   *Beware of the following caveats.* If a CLI command to decommission a node exited before the decommissioning was processed in the cluster (for example if the CLI command had `--wait=none`  or if it had `--wait=all` and was interrupted with ^C), the decommission can [likely] run to completion on the server side but the status of the node will stay in "decommissioning" and will never finalize to "decommissioned". This is because the final state is set by the CLI program at the end. If the CLI exist early, it will not have an opportunity to make an underlying API call to finalize the "decommissioned" state. To resolve this, repeat the decommission command, which should promptly succeed and flip the [state](https://www.cockroachlabs.com/docs/stable/ui-cluster-overview-page.html#decommissioned-nodes) first to "UNAVAILABLE", which will then turn into "DECOMMISSIONED" 5 minutes later ([designed behavior](https://www.cockroachlabs.com/docs/stable/ui-cluster-overview-page.html#decommissioned-nodes)).

   

7. Confirm that “replicas per store” and “leaseholders per store” of the decommissioned nodes are 0 in the metrics page.

   

8. The decommission command *leaves the CRDB process alive* on decommissioned nodes at the end. Shut down the CockroachDB process on decommissioned hosts by sending a shutdown signal (e.g. SIGTERM) to the CRDB process or use Linux service manager if the CRDB process is running as a service. This has to be done locally on all Old Region’s nodes. Disable the CockroachDB service, if used, to prevent it from inadvertently re-starting. If the underlying server, VM or a container will be re-used as a new cluster node, the decommissioned node's store directory needs to be manually cleared.

   

9. After a decommission completes, the compaction queue normally grows as a side effect. The remaining/active cluster nodes receive new range replicas which have to be compacted with existing ranges. By design, the compaction process is single-threaded and relatively slow. Users should expect no material impact on SQL workload while the queue is drained over multiple hours.

