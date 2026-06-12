# CockroachDB Cloud — Add Node(s)


### Overview

This runbook covers the procedure for adding nodes to an existing CockroachDB Cloud cluster. In CockroachDB Cloud, node management is performed through the Cloud Console or Cloud API — there is no manual binary deployment, systemd configuration, or OS-level work required.

<div style="margin-bottom: 50px;"></div>

### Concepts & Definitions

| Term | Definition |
|---|---|
| **Node** | A single CockroachDB instance within a cluster. In Advanced, nodes are added per region. |
| **Replica Rebalancing** | Automatic process by which CockroachDB redistributes range replicas across newly added nodes. This generates inter-node network traffic and takes time proportional to data volume. |
| **Leaseholder** | The replica responsible for serving reads and coordinating writes for a given range. Rebalances after node addition. |
| **Cloud Console** | The CockroachDB Cloud web UI at cockroachlabs.cloud |
| **Cloud API** | REST API for programmatic cluster management. Requires an API key. |

<div style="margin-bottom: 50px;"></div>

### When to Add Nodes

- CPU utilization per node consistently exceeds **70–80%** over a sustained period (visible in Cloud Console → Metrics → Hardware)
- p99 SQL response time is increasing and correlated with CPU saturation
- Storage per node is approaching 60% and expanding per-node storage is not sufficient or not preferred
- Anticipated workload growth (pre-provisioning before a traffic event)
- Adding a new cloud region to an Advanced multi-region cluster

### Prerequisites

- Cluster health confirmed — all nodes live, no active incidents
- Operator has Cluster Admin role in the CockroachDB Cloud organization
- No active backup or restore jobs running on the cluster
- Reason for scaling documented (capacity review, incident response, planned growth)
- Target node count determined and approved
- Change management ticket created and approved (if required by org policy)
- Application team notified — no downtime expected, but latency may briefly fluctuate during replica rebalancing

<div style="margin-bottom: 50px;"></div>

## Pre-Procedure Health Check

Run the following SQL against the cluster before initiating changes:

```sql
-- Confirm all nodes are live
SELECT node_id, address, is_live, is_available, replicas_count
FROM crdb_internal.kv_node_status
ORDER BY node_id;

-- Check for underreplicated ranges (should be 0)
SELECT count(*) AS underreplicated
FROM crdb_internal.ranges
WHERE array_length(replicas, 1) < 3;

-- Check for any active schema changes, backups, or restores
SELECT job_type, status, description
FROM crdb_internal.jobs
WHERE job_type IN ('SCHEMA CHANGE', 'BACKUP', 'RESTORE', 'IMPORT')
  AND status = 'running';
```

Do not proceed if any nodes are down, underreplicated ranges exist, or a restore is in progress.

<div style="margin-bottom: 50px;"></div>

## Procedure

### Option A: Cloud Console (Recommended for manual operations)

1. Navigate to [cockroachlabs.cloud](https://cockroachlabs.cloud) and select your cluster.
2. Click **Actions → Edit cluster**.
3. **For Advanced clusters:**
   - Select the target region in the region list.
   - Increase the **Number of nodes** field to the desired count.
   - To add a new region: click **+ Add region**, select the cloud provider region, and configure node count and hardware class.
4. **For Standard clusters:**
   - Adjust the **Compute** slider or field to the desired vCPU count.
5. Review the updated monthly cost estimate in the side panel.
6. Click **Update cluster**.
7. Monitor progress in **Cluster Overview** — new nodes will appear as `ADDING` and then transition to `LIVE`.

<div style="margin-bottom: 50px;"></div>

### Option B: Cloud API

```bash
# View current cluster configuration
curl -X GET \
  "https://cockroachlabs.cloud/api/v1/clusters/${CLUSTER_ID}" \
  -H "Authorization: Bearer ${CRDB_API_KEY}"

# Scale node count in a region (Advanced)
curl -X PATCH \
  "https://cockroachlabs.cloud/api/v1/clusters/${CLUSTER_ID}" \
  -H "Authorization: Bearer ${CRDB_API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "regions": [
      {
        "name": "us-east-1",
        "node_count": 5
      }
    ]
  }'

# Poll operation status
curl -X GET \
  "https://cockroachlabs.cloud/api/v1/clusters/${CLUSTER_ID}" \
  -H "Authorization: Bearer ${CRDB_API_KEY}" \
  | jq '.state'
```

<div style="margin-bottom: 50px;"></div>


## Post-Procedure Verification

Allow **15–30 minutes** after the Cloud Console shows nodes as `LIVE` for replica rebalancing to complete. For large clusters (>1 TB), allow more time.


**Cloud Console checks:**
- All new nodes show `LIVE` status in Cluster Overview
- No under replicated ranges
- Replica distribution is visible and balanced across nodes (Cloud Console → Metrics → Replication)
- CPU utilization on existing nodes is trending down as rebalancing completes
- SQL p99 latency has returned to baseline
- No new alerts have fired

<div style="margin-bottom: 50px;"></div>

## Rollback

CockroachDB Cloud does not provide instant rollback of node additions. To reverse:

1. Follow the [Remove Node(s) Runbook](../cockroach-cloud/cloud-remove-nodes.md) runbook to scale back down.
2. Nodes that received replicas will trigger a rebalancing cycle during removal — allow time for completion.
3. Do not reduce below the original node count while any ranges are underreplicated.
4. Minimum 3 nodes per region must be maintained at all times.

<div style="margin-bottom: 50px;"></div>

## Troubleshooting

| Symptom | Likely Cause | Action |
|---|---|---|
| New node stuck in `ADDING` state >30 min | Cloud provider resource availability | Open support ticket with Cockroach Labs |
| Replica count on new node stays at 0 | Rebalancing not yet started | Wait 10–15 min; check `crdb_internal.kv_node_status` |
| Latency spike during rebalancing | Increased inter-node traffic | Expected; monitor and wait for stabilization |
| Underreplicated ranges not clearing | Node failure during rebalancing | Check node status; contact support if nodes are not recovering |

<div style="margin-bottom: 50px;"></div>

## Related Resources

- [Manage Your Advanced Cluster](https://www.cockroachlabs.com/docs/cockroachcloud/advanced-cluster-management)
- [CockroachDB Cloud API Reference](https://www.cockroachlabs.com/docs/api/cloud/v1)
- [Terraform Provider — CockroachDB Cloud](https://registry.terraform.io/providers/cockroachdb/cockroach/latest)
- [Remove Nodes Runbook](./cloud-remove-nodes.md)
