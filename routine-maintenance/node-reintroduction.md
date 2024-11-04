# Procedure:  Reintroducing Node(s)

### About this Procedure

This procedure is used as a step of larger regular maintenance procedures or as an emergency procedure to resolve an operational issue.  This document assumes that there are no unavailable ranges when node(s) that are down.
Examples of regular on-line maintenance procedures relying on the reintroduction of nodes after a region or AZ outage.
This procedure may be used for emergency repairs, for example to swap a cluster node that suffered an underlying *hardware failure*. 


### Considerations For Reintroducing Multiple Nodes

Definition:

Light write workload - less than 10% of queries are writes

Average write workload - 15-20% of queries are writes

Heavy write workload - >25% of queries are writes


1. In the event of a region failure, up to 1/3 of the nodes in a cluster may be unavailable for a period of time.  Once enough data changes have occurred on the diminished cluster, the reintroduction of the segmented nodes can become more work to reconcile the differences than to add new nodes and start replicating data from scratch.

2. If the nodes of the lost region have been disconnected from the cluster for less than 12 hours and you have an average amount of writes in your workload, have those nodes rejoin the cluster.

If the nodes of the lost region have been dicsonnected from the cluster for more than 24 hours and you have an average amount of writes in your workload, build new nodes to replace them and decommission the old nodes that were segmented.

If the nodes of the lost region have been disconnected from the cluster for 12-24 hours and you have a heavy write workload, follow #2 above.  If the write workload is light, follow #1 above.

If your normal write workload is light, even if it has been 24hrs follow #1 above.  If your normal write workload is heavy and it has been 12hrs, follow #2 above.

If there is a region failure and the remaining portion of the cluster is continuing on with the workload and is mostly consumed keeping up with the regular workload, reintrouce the nodes one at a time.  If the cluster is not being stressed by the workload, nodes can be reintroduced a few at a time.


### Procedure Steps

You can use the other files in this section (node-add, node-remove, node-start, node-stop) to accomplish these tasks.
