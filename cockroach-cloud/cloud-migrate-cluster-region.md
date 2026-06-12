# CockroachDB Cloud — Migrate Cluster Region

<div style="margin-bottom: 50px;"></div>

## Overview

This runbook covers the procedure for migrating a CockroachDB Cloud Advanced cluster from one cloud region to another, or adding and removing regions in a multi-region cluster. Because CockroachDB is distributed and replicates data across nodes, region changes require careful orchestration: a new region must be provisioned, data replicated, application connectivity shifted, and the old region decommissioned all without data loss and with minimal downtime.

> Region migrations in CockroachDB Cloud Advanced are partially self-service (adding/removing regions via Console or API) but may require coordination with Cockroach Labs Support for large-scale data volumes or complex topologies.
>
> **Basic and Standard clusters:** Region changes may not be self-service. Contact Cockroach Labs Support for region migration assistance.

---

## Concepts & Definitions

| Term | Definition |
|---|---|
| **Home Region** | The region where a cluster's primary replicas (leaseholders) are pinned, if using regional-by-row or regional-by-table locality. |
| **Leaseholder** | The replica that serves reads and coordinates writes for a given range. |
| **Regional Table** | A table whose data is pinned to a specific region using `REGIONAL BY TABLE IN <region>`. |
| **Regional by Row** | A table pattern where each row's home region is tracked in a `crdb_region` column. Enables low-latency reads from any region. |
| **Global Table** | A table replicated to all regions with stale reads allowed globally. |
| **Zone Configuration** | CockroachDB's mechanism for controlling replica placement, lease preferences, and voting replica distribution. |
| **Survival Goal** | Cluster-level setting defining whether the cluster survives zone or region failure. |

<div style="margin-bottom: 50px;"></div>


## Migration Strategies

### Strategy 1: Add New Region → Migrate Data → Remove Old Region (Recommended)

Zero-downtime approach. Uses CockroachDB's native replication to move data before cutting over.

**Steps:**
1. Add the new target region to the cluster
2. Update zone configurations or table localities to move leaseholders to the new region
3. Update application connection strings to route to the new region
4. Verify replication and latency
5. Remove the old region

### Strategy 2: Provision New Cluster → Migrate Data → Cutover

Used when a full fresh start is preferred or the original cluster cannot be modified. Involves application-level data migration via BACKUP/RESTORE or logical replication.

<div style="margin-bottom: 50px;"></div>


## Prerequisites

- Target region is available in CockroachDB Cloud for the cluster's cloud provider (verify in Cloud Console)
- Operator has **Cluster Admin** role in the CockroachDB Cloud organization
- Database size and network bandwidth assessed to estimate replication time
- Application connection string update plan documented and reviewed
- Private networking (PrivateLink / VPC peering) configured in the new region if required
- Change management ticket created and approved
- Maintenance window scheduled for the application connection cutover step
- Cockroach Labs Support engaged for clusters >5 TB or with complex multi-region table locality configurations

<div style="margin-bottom: 50px;"></div>


## Pre-Migration Health Check

```sql
-- Confirm all nodes are live
SELECT node_id, address, is_live                                                                                                           FROM crdb_internal.gossip_nodes                                                                                                      ORDER BY node_id;

-- Confirm no underreplicated ranges
SELECT count(*) AS underreplicated
FROM crdb_internal.ranges
WHERE array_length(replicas, 1) < <replication_factor>;

-- Review current region/zone configuration
SHOW ZONE CONFIGURATION FOR DATABASE your_database;

-- Review table localities (multi-region clusters)
SELECT table_name, locality
FROM crdb_internal.tables
WHERE database_name = 'your_database';

-- Check database survival goal
SHOW DATABASES;
-- Look for 'survival_goal' column
```

<div style="margin-bottom: 50px;"></div>


## Procedure — Strategy 1: Add → Migrate → Remove

### Phase 1: Add the New Region

**Via Cloud Console:**
1. Navigate to your cluster → **Actions → Edit cluster**
2. Click **+ Add region**
3. Select the target region, configure node count (minimum 3) and instance type matching the existing regions
4. Click **Update cluster**
5. Wait for all new nodes to reach `LIVE` status before proceeding

**Via Cloud API:**
```bash
curl -X PATCH \
  "https://cockroachlabs.cloud/api/v1/clusters/${CLUSTER_ID}" \
  -H "Authorization: Bearer ${CRDB_API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "regions": [
      { "name": "us-east-1", "node_count": 3 },
      { "name": "eu-west-1", "node_count": 3 }
    ]
  }'
```

### Phase 2: Update Table Localities to Target Region

For **Regional by Table** tables — update the home region:
```sql
ALTER TABLE your_table SET LOCALITY REGIONAL BY TABLE IN "eu-west-1";
```

For **Regional by Row** tables — update the default home region:
```sql
ALTER DATABASE your_database SET PRIMARY REGION "eu-west-1";
```

For **zone configurations** (if not using multi-region SQL):
```sql
ALTER DATABASE your_database CONFIGURE ZONE USING
  constraints = '{"+region=eu-west-1": 2, "+region=us-east-1": 1}',
  lease_preferences = '[[+region=eu-west-1]]';
```

Allow **15–30 minutes** for leaseholders to migrate to the new region.

### Phase 3: Verify Leaseholder Migration

```sql
-- Check that leaseholders have moved to the new region
SELECT lease_holder, count(*) AS lease_count
FROM crdb_internal.ranges
GROUP BY lease_holder
ORDER BY lease_count DESC;

-- Cross-reference node IDs with regions
SELECT node_id, locality
FROM crdb_internal.kv_node_status;
```

Leaseholders should now predominantly reside on nodes in the new target region.

### Phase 4: Update Application Connectivity

1. Update connection strings to point to the new region's endpoint (available in Cloud Console → Connect)
2. If using private networking: ensure VPC peering or PrivateLink is configured for the new region and the old peering is not yet removed
3. Validate connectivity from the application tier:
   ```bash
   cockroach sql --url "postgresql://user:pass@new-region-host:26257/db?sslmode=verify-full"
   ```
4. Run application smoke tests against the new region endpoint
5. Shift production traffic to the new endpoint

### Phase 5: Monitor New Region Under Load

- Monitor for **15–30 minutes** after traffic cutover:
  - SQL p99 latency: Cloud Console → Metrics → SQL Response Time
  - CPU utilization per node
  - Error rates in application logs

### Phase 6: Remove the Old Region

**Via Cloud Console:**
1. **Actions → Edit cluster**
2. Click the **remove (×)** control next to the old region
3. Click **Update cluster**
4. Monitor decommissioning — nodes will appear as `DECOMMISSIONING` then disappear

**Via Cloud API:**
```bash
curl -X PATCH \
  "https://cockroachlabs.cloud/api/v1/clusters/${CLUSTER_ID}" \
  -H "Authorization: Bearer ${CRDB_API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "regions": [
      { "name": "eu-west-1", "node_count": 3 }
    ]
  }'
```

<div style="margin-bottom: 50px;"></div>


## Post-Migration Verification

**Cloud Console checks:**
- Old region nodes no longer appear
- All new-region nodes are `LIVE`
- SQL latency is at or below pre-migration baseline
- Application smoke tests passing
- Managed backup has completed successfully on the new topology
- Old private networking (VPC peering / PrivateLink) decommissioned if no longer needed

<div style="margin-bottom: 50px;"></div>


## Rollback Plan

If issues arise during Phase 4 (application cutover) or Phase 5:

1. **Do not remove the old region** — keep it live until the new region is fully validated
2. Revert application connection strings to the old region endpoint
3. Use zone configurations to move leaseholders back to the old region:
<div style="margin-bottom: 10px;"></div>

   ```sql
   ALTER DATABASE your_database CONFIGURE ZONE USING
     lease_preferences = '[[+region=us-east-1]]';
   ```
4. Investigate root cause before retrying Phase 4

<div style="margin-bottom: 50px;"></div>


## Troubleshooting

| Symptom | Likely Cause | Action |
|---|---|---|
| Leaseholders not migrating to new region | Zone config not applied; nodes not yet up | Wait 15 min; verify zone config with `SHOW ZONE CONFIGURATION` |
| Latency higher in new region | Geographic distance from clients | Verify application is connecting to correct regional endpoint |
| Old region nodes not decommissioning | Replicas stuck; storage constraints | Check `crdb_internal.ranges`; may need to increase storage in new region |
| Underreplicated ranges after region removal | Too many nodes removed too quickly | Stop removal; contact Cockroach Labs Support |

<div style="margin-bottom: 50px;"></div>


## Related Resources

- [Multi-Region Overview](https://www.cockroachlabs.com/docs/stable/multiregion-overview)
- [Table Localities](https://www.cockroachlabs.com/docs/stable/table-localities)
- [Zone Configurations](https://www.cockroachlabs.com/docs/stable/configure-replication-zones)
- [Manage Your Advanced Cluster](https://www.cockroachlabs.com/docs/cockroachcloud/advanced-cluster-management)
- [CockroachDB Cloud API Reference](https://www.cockroachlabs.com/docs/api/cloud/v1)
