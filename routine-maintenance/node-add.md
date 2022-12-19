# Procedure:  Add Node(s)

### About this Procedure

This procedure is used as a step of larger regular maintenance procedures or as an emergency procedure to resolve an operational issue.

Examples of regular on-line maintenance procedures relying on adding nodes are CockroachDB *cluster expansion/capacity increase, [*cluster topology change*](./cluster-region-migrate.md), rolling repaving of cluster VMs for security reasons, replacement or *hardware change* in the CockroachDB node's underlying VM or bare metal server.

This procedure may be used for emergency repairs, for example to swap a cluster node that suffered an underlying *hardware failure*. 



### Considerations For Adding Multiple Nodes

**Q:** Is it better to add all new nodes at once or add a node at a time, waiting for the range and leaseholder rebalancing to settle into a steady state before adding the next node?

**A:** Adding all nodes at once is commonly preferred by operators for the following reasons:

It is more efficient (or potentially vastly more efficient if the number of added nodes is large) because it minimizes the total of all data transfers before the cluster settles into a steady state.

The new nodes start actively participating in processing the user workload faster, taking the CPU pressure off the "old" nodes more eagerly.

The cluster reaches a steady state faster, making the cluster more elastic in response to a capacity adjustment. This is what operators commonly prefer. 

An operator is usually supervising/monitoring the cluster for the duration of a cluster expansion. Adding a node at a time increases the complexity of the operating procedure and extends the elapsed time to completion. If adding many nodes one by one, the procedure can take longer than a business day. For larger cluster expansions, the elapsed time may just be unacceptably long. 



**Q:** Are there situations when an operator has to add a node at a time?

**A:** There is a school of thought that if a cluster is *severely overloaded*, adding one node at a time and waiting for rebalancing to reach a steady state before adding the next node may be safer, saving existing nodes from "tipping over".

> Before commencing any maintenance procedure like a cluster expansion, an operator is advised to control cluster overload effects, if any, by governing the incoming workload concurrency.

An overloaded cluster is a reality that an operator has to be prepared to act on. There is a concern, however, that the approach of adding one node at a time may be creating a false sense of a "safer approach". In practice the "slow & measured" method could be the opposite of what the situation may be calling for - add more processing resources as fast as possible to bring relief to a struggling cluster.

Adding all nodes at once to an overloaded cluster is still believed to be a better strategy for a more vigorous offloading of the thrashing nodes. Because throwing more processors in and making them "work" sooner seems more effective in helping an overloaded cluster (particularly in combination with setting a faster rebalancing rate) than a "careful" but slow expansion dragging over an extended period of time.

An assumption that adding a node at a time reduces the extra load on existing nodes may also not hold water. The existing nodes are expected to only send rebalancing snapshots and not receive any. So the rebalancing overhead would be from sending the snapshots, which is much lighter than the resource overhead from receiving and applying the snapshots. As of v23.1 (current) or earlier, a node can only be sending one snapshot at a time. So the added CPU overhead to send rebalancing snapshots may not be materially larger vs. adding all nodes at once.

Fwiw, the current operating practice of the Cockroach Cloud service is to add all nodes at once (all nodes are added minutes apart) regardless of the number of nodes in the cluster and the number of nodes being added.



### Procedure Steps

1. Prior to executing this procedure for the first time, an operator is reminded to [set the cluster's *rebalance and recovery rates*](./change-rebalance-rate.md) to the value deemed [optimal](./change-rebalance-rate.md#considerations-for-setting-the-max-rates). Higher rates allow the cluster to complete transient data transfers and reach the steady state faster.  

   

2. Ensure that the Connection Pool's maximum connection life is set to 30 - 60 minutes. This is an essential step to ensure application (client) connection rebalancing. If omitted, the new nodes will not be receiving the new SQL connections and will not be fully participating in workload processing.

   

3. Add the new node(s) to the cluster. Adding [all new nodes at once](#considerations-for-adding-multiple-nodes) generally works best.

   

4. Add the added node(s) to the *load balancers*' configuration.

   

5. Increase the size of the connection pools to take advantage of the cluster's added capacity to process more concurrent workload.

   

6. Observe the rebalancing of ranges and leaseholders converging to a steady state with `CRDB Console -> Metrics -> Dashboard: Replication -> "Replicas per Node" and "Leaseholders per Node"` graphs.

