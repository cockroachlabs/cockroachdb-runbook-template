
# Procedure:  Cluster Region Migration

### About this Procedure

This procedure is invoked when all or some nodes in one of the CockroachDB cluster's regions are relocated to another region. For example, moving all nodes from `US East` to `Europe (Ireland)`. The procedure is online, with no service interruptions, and designed to minimize a performance impact on user workload.

The steps in this procedure can be edited to implement similar/related procedures, such as adding a new region to a CockroachDB cluster.




### Procedure Steps

1. Prior to executing this procedure for the first time, an operator is reminded to [set the cluster's *rebalance and recovery rates*](./change-rebalance-rate.md) to the value deemed [optimal](./change-rebalance-rate.md#considerations-for-setting-the-max-rates). Higher rates allow the cluster to complete transient data transfers and reach the steady state faster.

   

2. [Add all New Region’s nodes](./node-add.md) to the connection load balancing configuration.

   

3. Add all planned nodes in the New Region at once. Ensure the `--locality` reflects the new topology.

   

4. Wait for the rebalancing to complete and the cluster to settle into a steady state. Confirm nodes are available to take over the range replicas from the noes in the Old Region. 

   

5. Remove all Old Region’s nodes from the load balancing configuration.

   

6. [Decommission all Old Region’s nodes](./node-remove.md) at once, in one command.

   


### Migration of a Region in a Multi-Region Cluster

Additional considerations apply when migrating a Region in a cluster that is using [multi-region capabilities](https://www.cockroachlabs.com/docs/v21.2/multiregion-overview.html). The data in a multi-region cluster is partitioned on a [specially maintained column](https://www.cockroachlabs.com/docs/v21.2/set-locality#crdb_region) that stores the row's home region name.

A region can be migrated with or without region name change. If the region name can be preserved, a migration procedure would be simpler and measurably faster. However it could be confusing to database users if the region name refers to region's location and that location is changing. 

If the region name can be preserved, an outline of the region migration procedure would be:
- Add nodes in *new location* using the *old region name*
- Wait for the rebalancing to complete and the cluster to settle into a steady state
- Decommission nodes in the *old location*

If the region name has to change, in addition to the above outline:
- Add a new region name to the database
- Update all of the rows in the table that should be sent to the *new region / location*
- Remove the region from the database.



See also the procedure for transitioning (expanding) a [single-region cluster to a multi-region cluster](./cluster-sr2mr-transition.md).
