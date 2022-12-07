# Procedure:  Set Snapshot Rebalancing Rate

### About this Procedure

 **< UNDER CONSTRUCTION >**

Default is 8MB, deliberately thrilled down to ensure user workload is not impacted. However 256MB works better when a frequent topology change (elasticity) is a part of routine operations.
 Also monitor for loooong rebalancing and leaseholders excessive movements during expansion - the remedy is increasing the allowable snapshot rates.

### About this Procedure

This procedure is invoked when all or some nodes in one of the CockroachDB cluster's regions are relocated to another region. For example, moving all nodes from `US East` to `Europe (Ireland)`. The procedure is online, no service interruption, and it's designed to minimize the performance impact on the workload.

The steps in this procedure can be edited to implement similar/related procedures, such as adding a new region to a CockroachDB cluster.


### Procedure Steps

1. Increase the rebalancing rate to 256MB or better by setting two configuration options to the *same value*:

   ```
    set cluster setting kv.snapshot_rebalance.max_rate = '256 MB';
    set cluster setting kv.snapshot_recovery.max_rate = '256 MB';
   ```

   

2. 
