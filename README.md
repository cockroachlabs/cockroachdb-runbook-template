
# CockroachDB Runbook Template


### Overview

This document is a _Template_ for creating a custom CockroachDB _Runbook_, a.k.a. CockroachDB Operation Manual

A runbook is a reference document which describes a CockroachDB deployment in a specific application environment with related tasks, checklists, and operational procedures.

This template provides an overall structure and implementation outlines for common CockroachDB operating procedures, expediting the creation of a custom runbook - an important deliverable of the overall IT system to ensure a required state of preparedness.

Customers who already have a CockroachDB runbook can use this template to check their existing manual for completeness.

In practice, CockroachDB operators will strive to automate most of the checks and procedures. This template, however, is focused on documenting the detailed checklists and steps comprising individual operational procedures. The automation of these procedures is not in scope of this document.



---

## Terms Used in this Document

**CockroachDB Node**  is an instance of a cockroach server process. To underscore this point - a *node* is neither a [virtual] server nor an instance of an OS nor a container. Cockroach Labs strongly recommends running one CockroachDB *node* per one instance of an OS or per container.

**CockroachDB Cluster**  is a set of connected CockroachDB Nodes that form a single system that works together on all tasks.

**Platform**  is a set of compatible hardware, virtualized or containerized hardware, as well as related structures, on which CockroachDB can be run. Platform examples are bare metal x86\_64, AWS EC2, Google Cloud Platform, Microsoft Azure, VMware vSphere, Docker, Kubernetes.



---

## Contents

1. **[Service or System Overview](system-overview)**
    * [Business Overview](system-overview/_under-construction_.md)
        * [Functional System Overview](system-overview/_under-construction_.md)
        * [Environments and Designations](system-overview/_under-construction_.md)
        * [Service / System Owner](system-overview/_under-construction_.md)
        * [Operations and Management Roles and Responsibilities](system-overview/_under-construction_.md)
        * [Hours of Operation](system-overview/_under-construction_.md)
        * [Peak Periods](system-overview/_under-construction_.md)
        * [Cool (Quiet) Periods](system-overview/_under-construction_.md)
        * [Impact of a System Failure](system-overview/_under-construction_.md)
        * [Service Level Agreement (SLA)](system-overview/_under-construction_.md)
        * [Operational Priorities](system-overview/_under-construction_.md)
        * [Level 1 Support](system-overview/support-level-1.md)
        * [Level 2 Support](system-overview/support-level-2.md)
    * [Technical Overview](system-overview/_under-construction_.md)
        * [Hardware Platform](system-overview/_under-construction_.md)
        * [Virtualization or Containerization](system-overview/_under-construction_.md)
        * [Operating system](system-overview/_under-construction_.md)
        * [Clock Management](system-overview/_under-construction_.md)
        * [Network Design](system-overview/_under-construction_.md)
        * [Data Volumes](system-overview/_under-construction_.md)
        * [Planned Capacity](system-overview/_under-construction_.md)
        * [Cluster Right-Sizing, Expansion Strategy](system-overview/_under-construction_.md)
        * [Cluster Topology and Configuration](system-overview/_under-construction_.md)
        * [Auto-Scaling](system-overview/_under-construction_.md)
        * [Connection Management (Pooling, Balancing, Failover/Failback](system-overview/connection-management.md))
        * [Transactions: Implicit vs. Explicit](system-overview/transaction-implicit-explicit.md)
        * [Transaction Retries](system-overview/transaction-retires.md)
        * [Upstream Dependent Systems](system-overview/system-upstream.md)
        * [Downstream Dependent Systems](system-overview/system-downstream.md)
        * [Ecosystem Tools](system-overview/_under-construction_.md)
        * [Deployment and Configuration management tools](system-overview/config-management-tools.md)
1. **[Routine Maintenance Procedures](routine-maintenance/_under-construction_.md)**
    * [Open / Close database &quot;gates&quot;](routine-maintenance/_under-construction_.md)
    * [Node Start](routine-maintenance/node-start.md)
    * [Node Shutdown (Stop)](routine-maintenance/node-stop.md)
    * [Add a Node](routine-maintenance/node-add.md)
    * [Remove (Decommission) Node ](routine-maintenance/node-remove.md)
    * [Cluster Region Migration](routine-maintenance/cluster-region-migrate.md)
    * [Server / VM Replacement](routine-maintenance/_under-construction_.md)
    * [Changefeed Management](routine-maintenance/changefeed-management.md)
    * [Backup / Restore](routine-maintenance/backup-restore/README.md)
    * [Change --max-offset](routine-maintenance/change-max-offset.md)
    * [Change Rebalance Rate](routine-maintenance/change-rebalance-rate.md)
    * [Change Cluster Settings](routine-maintenance/change-cluster-setting.md)
    * [CockroachDB Version Upgrade](routine-maintenance/release-upgrade.md)
1. **[The Most Common Problems experienced by CockroachDB users](most-common-problems/README.md)**
1. **[Monitoring and Alerting](monitoring-alerts)**
    * Monitoring tools
        * [Prometheus](monitoring-alerts/monitoring-prometheus.md)
        * [Dashboard (DB Console, Grafana)](monitoring-alerts/monitoring-dashboard.md)
        * [System tables](monitoring-alerts/monitoring-sys-tables.md)
        * [Message and Error Logs](monitoring-alerts/monitoring-logs.md)
    * [Monitoring Alerts (README)](monitoring-alerts/README.md)
        * [Node CPU](monitoring-alerts/alert-node-cpu.md)
        * [Node CPU Anomaly](monitoring-alerts/alert-node-cpu-anomaly.md)
        * [Node Memory](monitoring-alerts/alert-node-memory.md)
        * [Node Storage Capacity](monitoring-alerts/alert-node-storage-capacity.md)
        * [Node Storage Performance](monitoring-alerts/alert-node-storage-perf.md)
        * [Node Liveness (inconsistent liveness check)](monitoring-alerts/alert-node-liveness.md)
        * [Node LSM Storage Health](monitoring-alerts/alert-lsm-health.md)
        * [Live Node Count Change](monitoring-alerts/alert-node-count.md)
        * [High Query Latency](monitoring-alerts/_under-construction_.md)
        * [Intent Buildup](monitoring-alerts/alert-intent-buildup.md)
        * [Changefeed Falling Behind](monitoring-alerts/alert-cdc-behind.md)
        * [Changefeed Frequent Restarts](monitoring-alerts/alert-cdc-restarts.md)
        * [Changefeed Stopped](monitoring-alerts/alert-cdc-stopped.md)
        * [Non-Incrementing Uptime Counter](monitoring-alerts/alert-non-incrementing-uptime.md)
        * [Version Mismatch](monitoring-alerts/alert-version-mismatch.md)
        * [CA Expiry](monitoring-alerts/_under-construction_.md)
1. **[Diagnostic and Support](diagnostic-support)**
    * [CRDB Error Codes and Descriptions](diagnostic-support/errors-codes.md)
    * [L1 Support (In-House)](diagnostic-support/support-l1.md)
    * [L2 Support (Escalation to Cockroach Labs)](support-l2.md)
    * [Troubleshooting procedures](diagnostic-support)
      * [Troubleshooting Workload Contention](diagnostic-support/troubleshooting-sql-contention.md)
      * [Troubleshooting Hardware Resource Contention](diagnostic-support/troubleshooting-hardware-contention.md)
1. **[Emergency Procedures / Operation Continuity](emergency-procedures/_under-construction_.md)**
    * [Node Replace](emergency-procedures/node-replace.md)
    * [Node Wipe](emergency-procedures/node-wipe.md)
    * [Node LSM compaction](emergency-procedures/lsm-compact.md)
    * [CockroachDB server/VM replacement](emergency-procedures/server-vm-replacement.md)
    * [Disaster Recovery: Database Restore](emergency-procedures/_under-construction_.md)
    * [Recovery from a Quorum Loss](emergency-procedures/_under-construction_.md)
    * [Recovery from logical data corruption](emergency-procedures/corruption-logical.md)



---

## Useful Resources and Examples

- Monitoring Alerts deployed [Cockroach Cloud managed Service](https://github.com/cockroachlabs/managed-service/tree/master/pkg/monitoring/prometheus/assets):  ([common](https://github.com/cockroachlabs/managed-service/tree/master/pkg/monitoring/prometheus/assets/common), [dedicated](https://github.com/cockroachlabs/managed-service/tree/master/pkg/monitoring/prometheus/assets/dedicated), [host](https://github.com/cockroachlabs/managed-service/tree/master/pkg/monitoring/prometheus/assets/host))
- Including the 6 [alerts delivered to users](https://github.com/cockroachlabs/managed-service/blob/master/pkg/monitoring/prometheus/assets/dedicated/alerts.cockroach-customer.yml) of Cockroach Cloud Dedicated
- Available [Monitoring Metrics](https://www.cockroachlabs.com/docs/v21.1/ui-custom-chart-debug-page.html#available-metrics) 

