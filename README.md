
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

1. **[The Most Common Problems experienced by CockroachDB users](most-common-problems/README.md)**
1. **[System Overview](system-overview)**
    * [Operational Readiness, Day 1](system-overview/day-1-2.md)
    * [Clock Management](system-overview/clock-management.md)
    * [Connection Management (Pooling, Balancing, Failover/Failback](system-overview/connection-management.md))
    * [Data Availability](system-overview/data-availability.md)
    * [Server Hardware Selection](system-overview/hw-selection.md)
    * [Custom DBA Role](system-overview/role-dba.md)
    * [Transactions: Implicit vs. Explicit](system-overview/transaction-implicit-explicit.md)
    * [Transaction Retries](system-overview/transaction-retires.md)
1. **[Routine Maintenance Procedures](routine-maintenance)**
    * [Maintenance Procedure Pre-Check](routine-maintenance/maintenance-pre-check.md)
    * [Start Node](routine-maintenance/node-start.md)
    * [Stop (Shutdown) Node](routine-maintenance/node-stop.md)
    * [Add Node(s)](routine-maintenance/node-add.md)
    * [Remove (Decommission) Node(s)](routine-maintenance/node-remove.md)
    * [Change Cluster Settings](routine-maintenance/change-cluster-setting.md)
    * [Changefeed Management](routine-maintenance/changefeed-management.md)
    * [Change --max-offset](routine-maintenance/change-max-offset.md)
    * [Change Rebalance Rate](routine-maintenance/change-rebalance-rate.md)
    * [Migrate Cluster Region](routine-maintenance/cluster-region-migrate.md)
    * [Backup / Restore](routine-maintenance/backup-restore/README.md)
    * [CockroachDB Version Upgrade](routine-maintenance/release-upgrade.md)
1. **[Monitoring and Alerting](monitoring-alerts)**
    * Monitoring tools
        * [Custom Dashboard (100 Essential Metrics to Monitor in Grafana, Datadog)](monitoring-alerts/monitoring-dashboard-custom.md)
        * [System tables](monitoring-alerts/monitoring-sys-tables.md)
        * [SAR](monitoring-alerts/sar.md)
    * [Monitoring Alerts (README)](monitoring-alerts/README.md)
        * [Node CPU](monitoring-alerts/alert-node-cpu.md)
        * [Node Memory](monitoring-alerts/alert-node-memory.md)
        * [Node Storage](monitoring-alerts/alert-node-storage.md)
        * [Node Storage Health (LSM)](monitoring-alerts/alert-lsm-health.md)
        * [Node Health (Liveness)](monitoring-alerts/alert-node-health.md)
        * [Intent Buildup](monitoring-alerts/alert-intent-buildup.md)
        * [Changefeeds](monitoring-alerts/alert-cdc.md)
        * [Version Mismatch](monitoring-alerts/alert-version-mismatch.md)
        * [Expirations](monitoring-alerts/alert-expirations.md)
    * [System Activity Report Configuration](monitoring-alerts/sar.md)
1. **[Diagnostic and Support](diagnostic-support)**
    * [Troubleshooting SQL Workload Contention](diagnostic-support/troubleshooting-sql-contention.md)
    * [Troubleshooting Hardware Resource Contention](diagnostic-support/troubleshooting-hardware-contention.md)
1. **[Emergency Procedures / Operation Continuity](emergency-procedures/_under-construction_.md)**
    * [Node Replace](emergency-procedures/node-replace.md)
    * [Node Wipe](emergency-procedures/node-wipe.md)
    * [Node LSM Compaction](emergency-procedures/lsm-compact.md)
    * [CockroachDB server/VM replacement](emergency-procedures/server-vm-replacement.md)

