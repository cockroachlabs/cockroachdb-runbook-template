# Procedure:  Set Snapshot Rebalance & Recovery Rates

### About this Procedure

 **< UNDER CONSTRUCTION >**



Default is 8MB, deliberately thrilled down to ensure user workload is not impacted. However 256MB works better when a frequent topology change (elasticity) is a part of routine operations.

Also monitor for loooong rebalancing and leaseholders excessive movements during expansion - the remedy is increasing the allowable snapshot rates.

The rebalance rate is used for moving replicas around for range count, load-based rebalancing, splits/merges, and user actions.

The recovery rate is used for up-replication of replicas on decommissioning or dead nodes (or catching up to other Raft group members). 

The value of this rate really depends on a few things - increasing it may cause things like increased read amp if the stores in the cluster can't handle the added write traffic.


### Procedure Steps

Increase the rebalance and recovery rates to 256MB by setting two configuration options to the *same value*:

```
 set cluster setting kv.snapshot_rebalance.max_rate = '256 MB';
 set cluster setting kv.snapshot_recovery.max_rate = '256 MB';
```

