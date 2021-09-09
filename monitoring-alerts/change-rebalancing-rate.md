
 **< UNDER CONSTRUCTION >**

### Snapshot Rebalancing Rate

Default is 8MB, deliberately thrilled down to ensure user workload is not impacted. However 256MB works better when a frequent topology change (elasticity) is a part of routine operations.
 Also monitor for loooong rebalancing and leaseholders excessive movements during expansion - the remedy is increasing the allowable snapshot rates.
