# CockroachDB Cloud — Remove (Decommission) Node(s) / Scale In


### Overview

This runbook covers the procedure for removing nodes from an existing CockroachDB Cloud Advanced cluster. In CockroachDB Cloud, node decommissioning is managed through the Cloud Console or Cloud API — Cockroach Labs handles the underlying drain and decommission sequence automatically. The operator's responsibility is to confirm cluster health before and after, and to ensure the resulting topology maintains quorum and replication targets.

<div style="margin-bottom: 50px;"></div>


## Concepts & Definitions

| Term | Definition |
|---|---|
| **Decommission** | The process of safely migrating all range replicas off a node before permanently removing it from the cluster. |
| **Drain** | The pre-decommission step in which a node stops accepting new connections, transfers range leases, and finishes in-flight transactions. |
| **Replica Rebalancing** | The redistribution of range replicas from the decommissioning node to remaining live nodes. This is network-intensive. |
| **Quorum** | The majority of replicas (⌊N/2⌋ + 1) required to make progress. Removing nodes must never threaten quorum. |
| **Replication Factor** | Number of replicas maintained per range. Default 3 (requires minimum 3 nodes). |

<div style="margin-bottom: 50px;"></div>

## When to Remove Nodes

- Sustained CPU utilization is consistently below **20–30%** across all nodes (cluster is over-provisioned)
- Workload has predictably decreased and rightsizing is warranted
- Region consolidation or architecture change
- Cost optimization initiative

<div style="margin-bottom: 50px;"></div>


## Prerequisites

- Cluster health confirmed — all remaining nodes are **Live** after removal
- Final node count will be **at least 3 per region** (replication factor 3 minimum)
- Final node count will not reduce any region below quorum for the configured replication factor
- Operator has **Cluster Admin** role in the CockroachDB Cloud organization
- No active backup, restore, or schema change jobs running
- Capacity analysis confirms remaining nodes can sustain current data volume (≤60% storage utilization post-removal)
- Capacity analysis confirms remaining nodes can sustain peak CPU load
- Change management ticket created and approved
- Application team notified — brief latency increase expected during replica migration

<div style="margin-bottom: 50px;"></div>


## Pre-Procedure Health Check

**Cloud Console checks:**
- All new nodes show `LIVE` status in Cluster Overview
- No under replicated ranges
- Replica distribution is visible and balanced across nodes (Cloud Console → Metrics → Replication)
- CPU utilization on existing nodes is trending down as rebalancing completes
- SQL p99 latency has returned to baseline
- No new alerts have fired

Do not proceed if:
- Any node is not live
- Any underreplicated ranges exist
- A restore job is in progress
- Storage utilization on remaining nodes would exceed 60% post-removal

<div style="margin-bottom: 50px;"></div>


## Capacity Check Before Removal

Estimate post-removal storage utilization:

```
Post-removal storage utilization per node =
  (Total cluster data volume) / (Nodes remaining × Storage per node)

Must be ≤ 60%
```

Example: 10 TB cluster, 5 nodes → 3 nodes, 2 TB each:
- Current: 10 TB / (5 × 2 TB) = **100%** ← **UNSAFE, do not proceed**
- Safe scenario: 10 TB / (5 × 4 TB) = 50% → removing to 4 nodes → 10 TB / (4 × 4 TB) = 62.5% ← **borderline, add storage first**

<div style="margin-bottom: 50px;"></div>


## Procedure

### Option A: Cloud Console (Recommended)

1. Navigate to the cloud console and select your cluster.
2. Click **Actions → Edit cluster**.
3. Select the target region.
4. Decrease the **Number of nodes** field to the desired count.
5. Review the updated cost estimate and the impact summary.
6. Click **Update cluster**.
7. Cockroach Labs will automatically:
   - Drain the selected node(s) (transfer leases, close connections)
   - Initiate decommission (migrate all replicas off the node)
   - Remove the node once decommission is complete
8. Monitor in **Cluster Overview** — nodes will appear as `DECOMMISSIONING` then disappear when complete.

### Option B: Cloud API

```bash
# View current cluster configuration
curl -X GET \
  "https://cockroachlabs.cloud/api/v1/clusters/${CLUSTER_ID}" \
  -H "Authorization: Bearer ${CRDB_API_KEY}"

# Reduce node count in a region (Advanced)
curl -X PATCH \
  "https://cockroachlabs.cloud/api/v1/clusters/${CLUSTER_ID}" \
  -H "Authorization: Bearer ${CRDB_API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "regions": [
      {
        "name": "us-east-1",
        "node_count": 3
      }
    ]
  }'
```



## Expected Duration

| Cluster Data Volume | Estimated Decommission Time |
|---|---|
| < 100 GB | 5–15 minutes |
| 100 GB – 1 TB | 15–60 minutes |
| 1 TB – 5 TB | 1–4 hours |
| > 5 TB | 4+ hours; coordinate with Cockroach Labs Support |


 ⚠️ Do not interrupt decommissioning once started. Interruption can leave ranges underreplicated and require manual recovery.

<div style="margin-bottom: 50px;"></div>


## Post-Procedure Verification



**Cloud Console checks:**
- Removed nodes no longer appear in Cluster Overview
- All remaining nodes show `LIVE`
- Storage utilization per node is within acceptable range
- SQL latency has returned to baseline after rebalancing
- Managed backup ran successfully on the new topology

<div style="margin-bottom: 50px;"></div>


## Rollback

If decommissioning must be reversed before it completes:

1. Contact Cockroach Labs Support immediately — Cloud does not expose a self-service recommission flow.
2. If decommissioning has completed, follow the **Add Node(s)** runbook to restore the original count.

<div style="margin-bottom: 50px;"></div>


## Troubleshooting

| Symptom | Likely Cause | Action |
|---|---|---|
| Decommission stalled >2× expected duration | Hotspot range preventing replica movement | Check `crdb_internal.ranges` for stuck ranges; contact support |
| Underreplicated ranges after decommission | Node removed before replica migration complete | Do not remove additional nodes; contact support |
| Storage utilization spikes on remaining nodes | Replicas consolidating faster than expected | Monitor; should stabilize post-rebalancing |
| SQL latency spike during decommission | Lease holder transfers | Expected; monitor and wait for stabilization |

<div style="margin-bottom: 50px;"></div>

## Related Resources

- [Manage Your Advanced Cluster](https://www.cockroachlabs.com/docs/cockroachcloud/advanced-cluster-management)
- [Node Shutdown — Self-hosted Reference](https://www.cockroachlabs.com/docs/stable/node-shutdown)
- [CockroachDB Cloud API Reference](https://www.cockroachlabs.com/docs/api/cloud/v1)
- [Add Nodes Runbook](./cloud-add-nodes.md)
