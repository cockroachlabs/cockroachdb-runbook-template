# Procedure:  Remove (Decommission) Node(s)

### About this Procedure

Removal of nodes from a CockroachDB cluster is referred to as nodes' [decommissioning](https://www.cockroachlabs.com/docs/stable/node-shutdown?filters=decommission#decommission-the-node).

This procedure is used as a step of larger regular maintenance procedures or as an emergency procedure to resolve an operational issue.

Examples of regular on-line maintenance procedures relying on node decommission are CockroachDB *cluster contraction*/capacity reduction, [*cluster topology change*](./cluster-region-migrate.md), rolling repaving of cluster VMs for security reasons, replacement or *hardware change* in the CockroachDB node's underlying VM or bare metal server.

This procedure may be used for emergency repairs, for example to remove of a running node from the cluster when an underlying *hardware failure* or platform *configuration errors* are observed. 

> ✅ The node decommission command can be run *from any* **running** *cluster node*. The target node(s), i.e. node(s) being decommissioned, could be any cluster node(s) - self or other - and they could be *in any state* - running, stopped, or even no longer existing


> ✅ A CockroachDB cluster will *not decommission a node automatically*, under any circumstances. Decommissioning nodes is strictly an operator's prerogative. In particular, a failed node that was declared "dead" and whose hardware was removed, is technically still a member of the cluster until it is removed from the cluster by an operator's action. "Dead" nodes that have not been decommissioned will be continually gossiped and probed by other cluster nodes, in anticipation that a "dead" node may recover and rejoin the cluster.



### Node Decommission vs. Node Drain (Shutdown)

Superficially, Decommission and [Drain](./node-stop.md) may appear to be functionally similar actions, yet they are materially different and implementation-wise have separate code paths, *not* sharing configuration/tuning "knobs". 

During the decommission, a node transfers the replicas of all its ranges to other nodes, drains all leases (it's a no-op since at that time there should be no replicas on that node), and closes all SQL client connections to the node. Node decommission implies that the node is not going to return to the cluster.
Drain transfers the range *leaseholders* (and raft groups' leaderships) away from the drained node and closes all SQL connections to the node, but still keeps data replicas participating in their raft consensus groups. A node draining is a precursor to stopping the node which is then expected to rejoin the cluster in a near term and reconnect its replicas to their raft groups.

Decommission is a safe operation from the standpoint of maintaining the cluster's fault tolerance level. A decommissioning action does not allow under-replicated ranges at any point. On the other hand, a node drain (which is a precursor to a node shutdown), if followed by a node process stop, leaves some ranges under-replicated.

> ✅ Stopping a node without decommissioning first, even if stopped after an orderly drain, will leave ranges under-replicated.

A node decommission includes data transfers (raft snapshots) in the amount of data stored on the decommissioning node - commonly hundreds of GBs to TBs. A node drain consists predominantly of light weights metadata operations. Therefore, a node decommission should be expected to take significantly longer than a node drain.



### Node Decommission Implementation Highlights

- To maintain the designed replication factor (RF) at all times during node decommission, a new replica for each range on a decommissioning node is added to some other node in the cluster via a raft snapshot. 
- Each node (store) sends only 1 snapshot at a time. And any node (store) can be receiving 1 snapshot at a time too.
- Range replica snapshots are sent by nodes that are range's leaseholders.
- The decommissioning node will have a longer queue of snapshot work than other nodes. This is because each non-decommissioning node in the cluster will have to send snapshots for the subset of its leaseholders that have a replica on the decommissioning node, but the decommissioning node will always have to send snapshots for *all* of its leaseholders.



### Node Decommission Planning 

Node(s) decommission, by definition, results in an increased demand for processing and storage capacity on all nodes remaining in the cluster. The operator needs to ensure that the remaining nodes have sufficient compute and storage resources to process more workload and accommodate more data replicas.

> ✅ When designing various maintenance procedures that leverage node add/node remove sequence for on-line node swap, be sure to [add new nodes](./node-add.md) first, allow the data to rebalance, then decommission as a final step. 



### Should Nodes be Manually Drained before Decommission?

TLDR; There is no need to complicate the procedure with a manual node drain before decommissioning it, as long as the decommissioning [procedure is robustly implemented](./node-remove.md#procedure-steps) and coordinated with configuration parameters of other components, particularly load balancers and connection pools.

However, a manually node drain could be a helpful remediation tool if a decommissioning node is malfunctioning or taking an excessively long time to shed its ranges. A manual drain should be fast and the leaseholders distributed evenly across the remaining nodes of the cluster after the drain should be able to send snapshots faster than a long queue of snapshots from one decommissioning node.

> ✅ If a manual node drain step is integrated into a decommissioning procedure, ensure that under no circumstances the node process is stopped after a manual drain. Shutting down nodes before decommissioning may result in under-replicated ranges or even a loss of quorum.



### Procedure Steps

1. [Set the *rebalancing rate*](./change-rebalancing-rate.md) to 256MB or better. This will allow the cluster to complete transient data transfers and reach the steady state faster.

   

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

   

6. `cockroach node status <id> --all`

   

7. Confirm that “replicas per store” and “leaseholders per store” of the decommissioned nodes are 0 in the metrics page.

   

8. The decommission command *leaves the CRDB process alive* on decommissioned nodes at the end. Shut down the CockroachDB process on decommissioned hosts by sending a shutdown signal (e.g. SIGTERM) to the CRDB process or use Linux service manager if the CRDB process is running as a service. This has to be done locally on all Old Region’s nodes. Disable the CockroachDB service, if used, to prevent it from inadvertently re-starting.

   

9. After a decommission completes, the compaction queue normally grows as a side effect. The remaining/active cluster nodes receive new range replicas which have to be compacted with existing ranges. By design, the compaction process is single-threaded and relatively slow. Users should expect no material impact on SQL workload while the queue is drained over multiple hours.

