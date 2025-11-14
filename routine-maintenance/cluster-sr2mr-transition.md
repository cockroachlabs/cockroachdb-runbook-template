
# Single-Region to Multi-Region Cluster Transition

### About this Procedure

This procedure is invoked when a CockroachDB cluster deployed in a single region is transitioned to multi-region after new nodes in 2 or more additional regions are added to the cluster. For example, extending a cluster deployed only in `aws-us-east-1` to span over  `aws-us-east-1`,  `aws-us-east-2` and `aws-eu-central-1`. The procedure is online, with no service interruptions, and designed to minimize the performance impact on the workload.

This procedure is commonly executed in a final phase of a legacy database migration into CockroachDB using MOLT tools. MOLT currently does not support a multi-region CockroachDB database as a direct target.



### Procedure Steps

This procedure assumes a database `mydb` exists in a cluster deployed in one region `aws-us-east-1`.

1. Promote the single-region database into a multi-region database (that for now would only have 1 region) by setting the PRIMARY REGION:
   	`ALTER DATABASE mydb SET PRIMARY REGION "aws-us-east-1";`

   That will have 2 effects:
   
   - CockroachDB multi-region SQL abstractions are unlocked and can be used in `mydb`.
   - All existing tables in `mydb` become Regional tables homed in the original region, which is now also the primary region.
   
   Leave survivability goal at the default **ZONE**. That will keep all ranges in the original region, preventing a spontaneous data rebalancing to newly added regions, which is the desired state at this time.

```
> SELECT NAME as "database_name", PRIMARY_REGION, REGIONS, SURVIVAL_GOAL from crdb_internal.databases WHERE NAME = 'mydb';
  database_name | primary_region | regions | survival_goal
----------------+----------------+---------+----------------
  mydb          | NULL           | {}      | NULL

> show regions;
     region     |                     zones                      | database_names | primary_region_of | secondary_region_of
----------------+------------------------------------------------+----------------+-------------------+----------------------
  aws-us-east-1 | {aws-us-east-1a,aws-us-east-1b,aws-us-east-1c} | {}             | {}                | {}
```

2. Add nodes in 2 or more new regions to the cluster. Perhaps `aws-us-east-2` and `aws-eu-central-1`.

   The order of steps 1 and 2 is important. If you add nodes before converting a single-region database into a trivial multi-region with just one region, CRDB will immediately start rebalancing data to new nodes. That's a welcome behavior in general, but in this case we want to control the data distribution per our design and avoid unnecessary data movements. Setting the PRIMARY region achieves that - existing tables become Regional tables homed in PRIMARY, so the data will not rebalance to new nodes.

3. Add 2 new regions to the database:

   `ALTER DATABASE mydb ADD REGION "aws-us-east-2"; ALTER DATABASE mydb ADD REGION "aws-eu-central-1";`

   We now have a multi-region database spanning 3 regions. The data can be rebalanced to new regions, but since all tables are Regional tables with PRIMARY in `aws-us-east-1`, the data will still stay in place.

4. Change the table types to Global and Regional-by-Row, per your design.

   Global tables will promptly get replicas in all regions, with the leaseholders staying in the PRIMARY REGION. [This blog](https://www.cockroachlabs.com/blog/optimize-write-latency/) explains how to optimize the response time of transactions with Global table writes.

   Regional-by-Row tables will get partitioned and all existing data will remain in the PRIMARY region because it will be the default value for home.
   To re-home records in a Regional-by-Row table, update the `crdb_region` column for rows that need to be relocated to other regions. All data will be online and continuously available through all steps.

5. Decide whether to stay with ZONE survivability goal or change it to REGION.
   This is a trade off decision. ZONE keeps the quorum of all ranges in-region, for best read and write response time. But if the entire home region is down or if the majority of zones is down - that region's data will not be available. Also note that if the survivability goal is ZONE, the number of replicas in Regional-by-Row tables is 3. If the goal is changed to REGION survivability, the number of replicas will be 5.

6. The data will automatically rebalance to a new steady state. Observe it in DBConsole:  Metrics -> Replication -> Replicas per Node (for physical range rebalancing) and Leaseholder per Node (leaseholder metadata rebalancing).

