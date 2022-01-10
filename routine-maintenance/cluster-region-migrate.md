
# Procedure: Cluster Region Migration

### Purpose of this Procedure

< work in progress >



### Procedure Steps

1. Increase the rebalancing rate to 256MB or better by setting two configuration options to the *same value*:

   ```
    set cluster setting kv.snapshot_rebalance.max_rate = '256 MB';
    set cluster setting kv.snapshot_recovery.max_rate = '256 MB';
   ```

   

2. Add all New Region’s nodes to the connection load balancing configuration.

   

3. Add all planned nodes in the New Region. Ensure the `--locality` reflects the new topology.

   

4. Wait for the rebalancing to complete and the cluster to settle into a steady state. Confirm nodes are available to take over the range replicas from the noes in the Old Region. 

   

5. Remove all Old Region’s nodes from the load balancing configuration.

   

6. Start decommissioning Old Region’s nodes. A single command

   `cockroach node decommission <space separated list of node ids> --host=…`

   to decommissioning all Old Region nodes is expected to work best. Alternatively decommission 1 node at a time, however, ensure that only one `cockroach node decommission` command runs at a time. A decommission command can be run from any cluster node.

   

7. Observe the status:

   ```
    id | is_live | replicas | is_decommissioning | is_draining 
   +---+---------+----------+--------------------+-------------+
     4 | false   |        0 |       true         |    true   
   (1 row)
   ```

   

8. `cockroach node status <id> --all`

   

9. Confirm that “replicas per store” and “leaseholders per store” of the decommissioned nodes are 0 in the metrics page.

   

10. The decommission command *leaves the CRDB process alive* on decommissioned nodes at the end. Shut down the Cockroach DB process on decommissioned hosts by sending a shutdown signal (e.g. SIGTERM) to the CRDB process or use Linux service manager if the CRDB process is running as a service. This has to be done locally on all Old Region’s nodes. Disable the CockroachDB service, if used, to prevent it from inadvertently re-starting.

    

11. After a decommission completes, the compaction queue normally grows as a side effect. The remaining/active cluster nodes receive new range replicas which have to be compacted with existing ranges. By design, the compaction process is single-threaded and relatively slow. Users should expect no material impact on SQL workload while the queue is drained over multiple hours.


### Migration of a Region in a Multi-Region Cluster

Addional considerations apply when migrating a Region in a cluster that is using [multi-region capabilities](https://www.cockroachlabs.com/docs/v21.2/multiregion-overview.html). The data in a multi-region cluster is partitioned on a [specially maintained column](https://www.cockroachlabs.com/docs/v21.2/set-locality#crdb_region) that stores the row's home region name.

A region can be migrated with or without region name change. If the region name can be preserved, a migration procedure would be simpler and measurably faster. However it could be confusing to database users if the region name refers to region's location and that location is changing. 

If the region name can be preserved, an outline of the region migration procedure would be:
- Add nodes in *new location* using the *old region name*
- Wait for the rebalancing to complete and the cluster to settle into a steady state
- Decommission nodes in the *old location*

If the region name has to change, in addition to the above outline:
- Add a new region name to the database
- Update all of the rows in the table that should be sent to the *new region / location*
- Remove the region from the database.
