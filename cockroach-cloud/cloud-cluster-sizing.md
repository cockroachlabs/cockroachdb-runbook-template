# CockroachDB Cloud — Cluster Sizing & Instance Type Selection


## Overview

This runbook guides operators through selecting the appropriate CockroachDB Cloud tier and sizing compute, storage, and replication configuration for a new or re-architected cluster. Unlike self-hosted deployments, hardware selection in CockroachDB Cloud is abstracted into cloud-managed instance families and storage classes managed by Cockroach Labs. The decisions made here directly affect performance, cost, and resilience.

<div style="margin-bottom: 50px;"></div>


## Concepts & Definitions

| Term | Definition |
|---|---|
| **Basic** | Consumption-based, auto-scaling, shared compute tier. No node management. Billed by Request Units (RUs) and storage. |
| **Standard** | Dedicated compute, managed by Cockroach Labs. User controls compute (vCPU) and storage capacity. |
| **Advanced** | Full control over cluster topology, regions, and node count. Supports private networking (VPC peering, PrivateLink). Equivalent to the former "Dedicated" tier. |
| **vCPU** | Virtual CPU allocated per node. Primary compute sizing dimension in Standard and Advanced. |
| **Request Unit (RU)** | Billing unit in Basic clusters representing compute and I/O consumed per operation. |
| **Storage** | Persistent SSD allocated per node (cloud provider-backed). |
| **Replication Factor** | Number of replicas maintained per range. (Default is 3) |
| **Cloud Console** | The CockroachDB Cloud web UI at cockroachlabs.cloud |

<div style="margin-bottom: 50px;"></div>

## Tier Selection Guide

### Choose Basic when:
- Development, staging, or low-traffic internal workloads
- Traffic is highly variable or unpredictable (auto-scaling is advantageous)
- No requirement for private networking or VPC peering
- Cost must track usage rather than reserved capacity

### Choose Standard when:
- Production workloads requiring predictable, dedicated compute performance
- Teams migrating from on-prem who want managed infrastructure without full topology control
- Requirement for dedicated compute without the overhead of node-level management

### Choose Advanced when:
- Enterprise production workloads with strict tenant isolation requirements
- Multi-region deployments requiring fine-grained replication topology control
- Requirements for private networking (AWS PrivateLink, GCP Private Service Connect, Azure Private Link)
- Compliance mandates single-tenant infrastructure
- Migrating from CockroachDB self-hosted; topology parity is needed

<div style="margin-bottom: 50px;"></div>


## Sizing Guidance — Standard & Advanced

<div style="margin-bottom: 20px;"></div>

### Compute per Node

| Workload Profile | Recommended vCPU/Node | Memory Ratio |
|---|---|---|
| OLTP, low-to-medium concurrency | 4–8 vCPU | 4 GB/vCPU minimum |
| OLTP, high concurrency | 8–16 vCPU | 4–8 GB/vCPU |
| Mixed OLTP + analytics | 16–32 vCPU | 8 GB/vCPU |
| Read-heavy, large working set | 8–16 vCPU + extra storage | 4–8 GB/vCPU |

<div style="margin-bottom: 20px;"></div>


>**Concurrency rule of thumb:** CockroachDB performs best when active connection concurrency does not exceed **4× total vCPU count** across the cluster.

<div style="margin-bottom: 50px;"></div>

### Storage per Node

- Provision storage at **no more than 60% utilization** — CockroachDB requires ~40% free space for compaction, rebalancing, and snapshots.
- Storage **cannot be decreased** after provisioning; plan conservatively and enable auto-scaling.

<div style="margin-bottom: 50px;"></div>


### Node Count & Replication

| Topology | Minimum Nodes | Replication Factor |
|---|---|---|
| Single-region production | 3 | 3 |
| Single-region, high availability | 5 | 5 |
| Multi-region (per region) | 3 | 3 (or 5) |
| Multi-region, survive region failure | 3 regions × 3 nodes | 5 |

<div style="margin-bottom: 20px;"></div>

>For Advanced multi-region clusters, ensure an odd number of regions or use a primary region with explicit zone configuration to maintain quorum during region failures.

<div style="margin-bottom: 50px;"></div>


## Prerequisites

- Workload characterization completed (QPS, peak concurrency, read/write ratio, data volume)
- Latency SLOs documented (p50, p99 targets)
- Network topology requirements confirmed (public endpoint vs. private networking)
- Cloud provider and region(s) selected
- Compliance and data residency requirements reviewed
- CockroachDB Cloud organization account provisioned with appropriate billing configured

<div style="margin-bottom: 50px;"></div>


## Sizing Decision Checklist

- Tier selected: Basic / Standard / Advanced
- Cloud provider selected: AWS / GCP / Azure
- Region(s) selected and confirmed available in chosen tier
- Number of nodes per region determined (Advanced)
- vCPU count per node determined
- RAM per node confirmed via instance family selection
- Storage per node provisioned with ≥40% headroom at expected peak
- Automatic storage scaling enabled (if available for tier)
- Estimated monthly cost reviewed against budget and approved

<div style="margin-bottom: 50px;"></div>

## Post-Provisioning Verification

- All nodes show **Live** status in Cloud Console → Cluster Overview
- SQL connectivity verified from application subnet or private endpoint
- TLS certificate validated for client connections
- Latency baseline captured: Cloud Console → Metrics → SQL Response Time
- Storage utilization confirmed below 60%: Cloud Console → Metrics → Capacity
- Alerting configured: Cloud Console → Alerts (or external via Datadog / PagerDuty integration)
- Managed backup is enabled and schedule confirmed: Cloud Console → Backup & Restore

<div style="margin-bottom: 50px;"></div>


## Common Mistakes to Avoid

| Mistake | Impact | Prevention |
|---|---|---|
| Provisioning storage at >60% utilization | Compaction failures, performance degradation | Always plan for 40% headroom |
| Choosing fewer than 3 nodes in a region | Loss of quorum on node failure | Minimum 3 nodes per region for production |
| Using Basic for latency-sensitive production | Shared compute causes unpredictable latency | Use Standard or Advanced for SLO-bound workloads |
| Not enabling auto-storage scaling | Manual intervention during growth surges | Enable at provisioning time |
| Ignoring connection pool sizing | Concurrency exceeds 4× vCPU; instability | Size pools relative to total cluster vCPU |

<div style="margin-bottom: 50px;"></div>


## Related Resources

- [CockroachDB Cloud Pricing](https://www.cockroachlabs.com/pricing/)
- [Cloud Cluster Management](https://www.cockroachlabs.com/docs/cockroachcloud/cluster-management)
- [Production Checklist — Self-hosted Reference](https://www.cockroachlabs.com/docs/stable/recommended-production-settings)
- [CockroachDB Cloud API](https://www.cockroachlabs.com/docs/api/cloud/v1)
- [Terraform Provider for CockroachDB Cloud](https://registry.terraform.io/providers/cockroachdb/cockroach/latest)
