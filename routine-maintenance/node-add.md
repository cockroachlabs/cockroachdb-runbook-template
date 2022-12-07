# Procedure:  Add Node(s)

### About this Procedure

This procedure is used as a step of larger regular maintenance procedures or as an emergency procedure to resolve an operational issue.

Examples of regular on-line maintenance procedures relying on adding nodes are CockroachDB *cluster expansion/capacity increase, [*cluster topology change*](./cluster-region-migrate.md), rolling repaving of cluster VMs for security reasons, replacement or *hardware change* in the CockroachDB node's underlying VM or bare metal server.

This procedure may be used for emergency repairs, for example to swap a cluster node that suffered an underlying *hardware failure*. 



### Procedure Steps

1. [Set the *rebalancing rate*](./change-rebalancing-rate.md) to 256MB or better. This will allow the cluster to complete transient data transfers and reach the steady state faster. This also minimizes the leaseholder transfers moves during range rebalancing, which reduces the occasional latency spikes if a key operation is retired due to leaseholder in transit to another node.  

   

2. Ensure that the Connection Pool's maximum connection life is set to 30 - 60 minutes. This is an essential step to ensure application (client) connection rebalancing. If omitted, the new nodes will not be receiving the new SQL connections and will not be fully participating in workload processing.

   

3. Add the new node(s). Adding all new nodes at once generally works best because it minimizes the total of all data transfers before the cluster settles into a steady state. However, if a cluster is severely overloaded, adding one node at a time and waiting for rebalancing to reach a steady state before adding the next node may be a prudent operational decision.

   

4. Add the added node(s) to the *load balancers*' configuration.

   

5. Increase the size of the connection pools to take advantage of the cluster's added capacity to process more concurrent workload.

   

6. Observe the rebalancing of ranges and leaseholders converge to a steady state.
   *< TODO: add a picture of graphs and leaseholders UI graphs during rebalancing >*

