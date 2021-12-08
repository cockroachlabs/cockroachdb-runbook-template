
# Procedure: Cluster Region Migration

### Purpose of this Procedure

This procedure is an integral part of all regular maintenance that require a CockroachDB node process restart. For example, a rolling software upgrade.

The first and the most essential phase of the shutdown is a *node drain*, during which a node orderly ends processing of transactions, closes client connections, transfers its range leases and stops ranges from rebalancing onto the node.

During a node shutdown, a node drain is followed by a node process termination.



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

TODO!!!

**[Alex Entin](https://app.slack.com/team/UN56QMZU4)**[15 hours ago](https://cockroachlabs.slack.com/archives/C4ERHM60Z/p1637014728053700)

Do we have a region migration procedure for a multi-region cluster?
Does the region partitioning column has to be updated to match the new region name after new nodes added in New region and before nodes decommissioned in the Old region?
That can be massive. Is it a better idea to preserve the region name and just add nodes in New location using the old region name, rebalance, then decommission nodes in the Old location? (edited) 

**[Adam Storm](https://app.slack.com/team/U019J63NXUM)**[2 minutes ago](https://cockroachlabs.slack.com/archives/C4ERHM60Z/p1637068805054000?thread_ts=1637014728.053700&cid=C4ERHM60Z)

> Does the region partitioning column has to be updated to match the new region name after new nodes added in New region and before nodes decommissioned in the Old region?

I’m assuming that you’re talking about using the new 21.2 primitives here. If so, yes. You’d first have to add the new region to the database, then update all of the rows in the table that you want to send to the new region, then remove the region from the database.

> Is it a better idea to preserve the region name and just add nodes in New location using the old region name, rebalance, then decommission nodes in the Old location?

You could do this, and it’s likely to be much faster, but I’d imagine it would be very confusing for the customer long-term.

