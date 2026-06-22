# CockroachDB Cloud — CockroachDB Version Upgrade

<div style="margin-bottom: 50px;"></div>


## Overview

This runbook covers CockroachDB version upgrades in CockroachDB Cloud. Unlike self-hosted deployments — where operators manage binary downloads, rolling restarts, and finalization — **CockroachDB Cloud handles all node-level upgrade mechanics**. The operator's responsibilities are:

1. Reviewing release notes for breaking changes and application impact
2. Scheduling and initiating the upgrade via Cloud Console or API
3. Monitoring the upgrade and verifying application health
4. Finalizing the upgrade (major versions only)

All upgrades in CockroachDB Cloud are performed as rolling upgrades — nodes are upgraded one at a time, maintaining cluster availability throughout.

<div style="margin-bottom: 50px;"></div>


## Concepts & Definitions

| Term | Definition |
|---|---|
| **Major Release** | A release with a new Y.R version number (e.g., 25.1 → 25.2). Introduces new features and may include backward-incompatible changes. Requires finalization. |
| **Patch Release** | A release with a new P number (e.g., 25.1.3 → 25.1.4). Bug fixes only. No finalization required. Auto-applied in Cloud. |
| **Innovation Release** | An optional release type introduced after v24.2 on a 6-month cadence. Can be skipped when upgrading. |
| **Regular Release** | The twice-yearly major release. Cannot be skipped; all Regular releases must be installed in order. |
| **Finalization** | The step that permanently upgrades the on-disk data format to the new major version. Prevents downgrade after this point. |
| **Auto-Upgrade** | Cloud feature (configurable) where Cockroach Labs automatically applies patch releases. |
| **Mixed-Version State** | The temporary state during a rolling upgrade when some nodes run the new version and others run the old version. |
| **Cluster Version** | The logical version at which the cluster is operating, distinct from individual node binary versions. |

<div style="margin-bottom: 50px;"></div>


## Release Cadence

```
Regular Releases:     Y.1 (Q1/Q2)  →  Y.2 (Q3/Q4)     — 2× per year, cannot skip
Innovation Releases:  Y.1.1 / Y.2.1                  — 2x per year, Optional, can skip
Patch Releases:       Y.R.1, Y.R.2, ...              — Bug fixes, auto-applied in Cloud
```

**Upgrade path rules:**
- Regular → next Regular: allowed
- Regular → next Innovation: allowed
- Regular → skip a Regular: **NOT allowed**
- Innovation → next Regular or Innovation: allowed
- Downgrade after finalization: **NOT possible**

<div style="margin-bottom: 50px;"></div>


## Pre-Upgrade Checklist

### Review Release Notes

- Read the [CockroachDB Release Notes](https://www.cockroachlabs.com/docs/releases/) for the target version
- Identify any **backward-incompatible changes** affecting your SQL, drivers, or ORMs
- Review **deprecated features** that may be removed in the target version
- Check **new reserved keywords** — verify none conflict with existing table/column names
- Review any **cluster setting changes** — defaults that may change behavior
- Validate **driver/ORM compatibility** with the target version (especially pgwire protocol changes)

### Application & Schema Readiness

- SQL-level tests run against the target version in a non-production environment
- Application smoke tests designed and ready to execute post-upgrade
- Any required schema changes (for deprecated syntax or removed features) applied and tested
- Connection pooling and retry logic reviewed — rolling upgrades cause brief per-node disruptions

### Cluster Health

- All nodes **Live** in Cloud Console
- Zero underreplicated ranges
- No active restore jobs
- No active schema change jobs (run the SQL below to verify)
- Managed backup completed successfully within the last 24 hours

<div style="margin-bottom: 20px;"></div>

> Do not proceed if any nodes are down, underreplicated ranges exist, or a restore is in progress.

<div style="margin-bottom: 50px;"></div>

## Upgrade Procedure

### Patch Release Upgrades

Patch releases are **automatically applied** by Cockroach Labs on Cloud clusters. No operator action is required. You will receive email notification before and after the upgrade.

To **opt out of auto-patch upgrades** (Advanced only):
1. Cloud Console → Cluster → **Settings**
2. Disable **Automatic patch upgrades**

To **manually apply a patch release:**
1. Cloud Console → **Actions → Upgrade cluster**
2. Select the target patch version
3. Click **Start upgrade**

### Major Release Upgrades

Major releases require operator initiation and finalization.

#### Step 1: Initiate the Upgrade

**Via Cloud Console:**
1. Navigate to your cluster → **Actions → Upgrade cluster**
2. Review the target version and the upgrade summary
3. Click **Start upgrade**
4. Cockroach Labs begins a rolling upgrade — one node at a time, in each region

**Via Cloud API:**
```bash
# Check available upgrade versions
curl -X GET \
  "https://cockroachlabs.cloud/api/v1/clusters/${CLUSTER_ID}/available-upgrades" \
  -H "Authorization: Bearer ${CRDB_API_KEY}"

# Initiate upgrade
curl -X POST \
  "https://cockroachlabs.cloud/api/v1/clusters/${CLUSTER_ID}/upgrade" \
  -H "Authorization: Bearer ${CRDB_API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{"cockroach_version": "v25.2.0"}'
```

#### Step 2: Monitor the Rolling Upgrade

The rolling upgrade typically takes **10–30 minutes** for small clusters. Allow more time for larger clusters or multi-region deployments.


**Cloud Console:** 

Monitor cluster status — nodes will briefly go `UPGRADING` then return to `LIVE` as each node is upgraded.

**Application monitoring during upgrade:**
- Brief per-node connection disruption is expected as each node restarts
- Connection pools with retry logic will handle this transparently
- Monitor application error rates and latency for anomalies

#### Step 3: Validate Post-Upgrade (Before Finalization)

After all nodes are on the new version, the cluster enters a **validation window** before finalization. During this window, downgrade is still possible.

```sql
-- Confirm all nodes are on the new version
SELECT node_id, server_version
FROM crdb_internal.kv_node_status
ORDER BY node_id;

-- Confirm cluster version has advanced (but not yet finalized)
SHOW CLUSTER SETTING version;

-- Check for any job failures during upgrade
SELECT job_type, status, error
FROM crdb_internal.jobs
WHERE finished > now() - interval '2 hours'
  AND status = 'failed';
```

**Application validation:**
- Run application smoke tests
- Verify all critical queries execute correctly
- Verify changefeed jobs resumed and are running (if applicable)
- Verify scheduled backup ran successfully on new version

#### Step 4: Finalize the Upgrade

> **Finalization is irreversible.** After finalizing, the cluster cannot be downgraded to the previous major version. Ensure all validation steps above have passed before proceeding.

**Via Cloud Console:**
1. A **Finalize upgrade** button appears in the cluster overview once all nodes are upgraded
2. Click **Finalize upgrade**
3. CockroachDB runs internal migration jobs to update on-disk data formats
4. Finalization typically completes in 5–15 minutes but may take longer for large clusters



<div style="margin-bottom: 50px;"></div>


## Post-Upgrade Verification


**Cloud Console checks:**
- All nodes show `LIVE` status
- Version shown in Cluster Overview matches target version
- No alerts fired during or after upgrade
- Managed backup completed on new version
- Application smoke tests all passing

<div style="margin-bottom: 50px;"></div>


## Downgrade (Before Finalization Only)

If issues are found after the rolling upgrade but **before finalization**:

1. Cloud Console → **Actions → Downgrade cluster** (available only in pre-finalization window)
2. The cluster will perform a rolling downgrade back to the previous version
3. Document the issue and contact Cockroach Labs Support before re-attempting the upgrade

> Downgrade is not available after finalization. If a critical issue is found post-finalization, contact Cockroach Labs Support.

<div style="margin-bottom: 50px;"></div>


## Upgrade Timing Guidance

| Upgrade Type | Suggested Timing |
|---|---|
| Patch release | Any time; auto-applied; brief maintenance window if on-demand |
| Major release (pre-validation) | During a low-traffic window; schedule 1–2 hours |
| Major release finalization | After 24–72 hours of validation post-upgrade; off-peak recommended |

<div style="margin-bottom: 50px;"></div>

## Troubleshooting

| Symptom | Likely Cause | Action |
|---|---|---|
| Upgrade stuck >60 min | Node failing to restart; cloud provider issue | Check Cloud Console status; contact Cockroach Labs Support |
| Application errors during rolling upgrade | Missing connection retry logic | Add exponential backoff and retry to connection pool configuration |
| Changefeed paused post-upgrade | Coordinator node restarted | Resume with `RESUME JOB <job_id>` |
| Schema change jobs failed during upgrade | Incompatibility with new version | Review error in `crdb_internal.jobs`; contact support |
| Finalization jobs stuck | Large data volume or internal migration issue | Monitor; contact Cockroach Labs Support if stuck >1 hour |
| Queries returning errors after upgrade | Breaking change in SQL behavior | Review release notes for behavioral changes; update application queries |

<div style="margin-bottom: 50px;"></div>


## Related Resources

- [CockroachDB Release Notes](https://www.cockroachlabs.com/docs/releases/)
- [Upgrade Policy](https://www.cockroachlabs.com/docs/cockroachcloud/upgrade-policy)
- [Upgrade a CockroachDB Cloud Cluster](https://www.cockroachlabs.com/docs/cockroachcloud/upgrade-cockroachdb)
- [Major Release Finalization](https://www.cockroachlabs.com/docs/stable/upgrade-cockroach-version#finalize-the-upgrade)
- [CockroachDB Cloud API Reference](https://www.cockroachlabs.com/docs/api/cloud/v1)
- [Changefeed Management Runbook](./cloud-changefeed-management.md)
