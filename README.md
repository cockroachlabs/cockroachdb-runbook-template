
# CockroachDB Runbook Template


### Overview

This document is a _Template_ for creating a custom CockroachDB _Runbook_, a.k.a. CockroachDB Operation Manual

A runbook is a reference document which describes a CockroachDB deployment in a specific application environment with related tasks, checklists, and operational procedures.

This template provides an overall structure and implementation outlines for common CockroachDB operating procedures, expediting the creation of a custom runbook - an important deliverable of the overall IT system to ensure a required state of preparedness.

Customers who already have a CockroachDB runbook can use this template to check their existing manual for completeness.

In practice, CockroachDB operators will strive to automate most of the checks and procedures. This template, however, is focused on documenting the detailed checklists and steps comprising individual operational procedures. The automation of these procedures is not in scope of this document.

If a Cockroach Labs Enterprise Architect (CEA) participates in a deployment rollout, a customer will receive consultative assistance with:

- Review of the custom runbook
- CockroachDB best practices



---

## Terms Used in this Runbook

**CockroachDB Node**  is an instance of a cockroach server process. To underscore this point - a *node* is neither a [virtual] server nor an instance of an OS nor a container. Cockroach Labs strongly recommends running one CockroachDB *node* per one instance of an OS or per container.

**CockroachDB Cluster**  is a set of connected CockroachDB Nodes that form a single system that works together on all tasks.

**Platform**  is a set of compatible hardware or virtualized hardware, as well as related structures, on which CockroachDB can be run. Platform examples are bare metal x86\_64, AWS EC2, Google Cloud Platform, Microsoft Azure, VMware vSphere.




---

## Contents

1. **[Service or System Overview](system-overview/under-construction.md)**
    * [Business Overview](system-overview/under-construction.md)
        - [Functional System Overview](system-overview/under-construction.md)
        - [Environments and Designations](system-overview/under-construction.md)
        - [Service / System Owner](system-overview/under-construction.md)
        - [Operations and Management Roles and Responsibilities](system-overview/under-construction.md)
        - [Contact List](system-overview/under-construction.md)
        - [Hours of Operation](system-overview/under-construction.md)
        - [Peak Periods](system-overview/under-construction.md)
        - [Cool (Quiet) Periods](system-overview/under-construction.md)
        - [Service Level Agreement (SLA)](system-overview/under-construction.md)
        - [Impact of a System Failure](system-overview/under-construction.md)
        - [Operational Priorities](system-overview/under-construction.md)
    * [Technical Overview](system-overview/under-construction.md)
        
        - [Data Volumes](system-overview/under-construction.md)
        * [Growth Rate](system-overview/under-construction.md)
        - [Cluster Topology and Configuration](system-overview/under-construction.md)
        - [Hardware Platform](system-overview/under-construction.md)
        - [VM shape](system-overview/under-construction.md)
        - [Planned Capacity](system-overview/under-construction.md)
        - [Auto-Scaling](system-overview/under-construction.md)
        - [Virtualization or Containerization](system-overview/under-construction.md)
        - [Operating system](system-overview/under-construction.md)
        - [Network Design](system-overview/under-construction.md)
        - [Upstream Dependent Systems](system-overview/under-construction.md)
        - [Downstream Dependent Systems](system-overview/under-construction.md)
        - [Ecosystem Tools](system-overview/under-construction.md)
        - [Deployment and Configuration management tools](system-overview/under-construction.md)
        - [Connections Load Balancing](system-overview/under-construction.md)
        - [Concurrent Connections Governance](system-overview/under-construction.md)
        * [Throttling of connections](system-overview/under-construction.md)
1. **[Routine Maintenance Procedures](routine-maintenance/under-construction.md)**
    * [Open / Close database &quot;gates&quot;](routine-maintenance/under-construction.md)
    * [Force closing application connections](routine-maintenance/under-construction.md)
    * [Firewall off the incoming traffic](routine-maintenance/under-construction.md)
    * [Node Start / Stop / Restart](routine-maintenance/under-construction.md)
    * [Cluster Startup](routine-maintenance/under-construction.md)
    * [Cluster Shutdown](routine-maintenance/under-construction.md)
    * [Add Node(s)](routine-maintenance/under-construction.md)
    - [Decommission Node(s)](routine-maintenance/under-construction.md)
    * [Server / VM Replacement](routine-maintenance/under-construction.md)
    * [Backup / Restore](routine-maintenance/under-construction.md)
    * [Change --max-offset](routine-maintenance/under-construction.md)
    * [Configuration Management](routine-maintenance/under-construction.md)
    * [CockroachDB License Management](routine-maintenance/under-construction.md)
    * [Secrets Management](routine-maintenance/under-construction.md)
    * [Job Schedules](routine-maintenance/under-construction.md)
    * [Scheduled Maintenance and Upgrades](routine-maintenance/under-construction.md)
        - [OS Updates](routine-maintenance/under-construction.md)
        - [Containerization Platform Upgrade](routine-maintenance/under-construction.md)
        - [CockroachDB Minor Version Upgrade](routine-maintenance/under-construction.md)
        - [CockroachDB Major Version Upgrade](routine-maintenance/under-construction.md)
        - [Application Upgrade](routine-maintenance/under-construction.md)
    * [Database Administration Procedures](routine-maintenance/under-construction.md)
    - [New database user setup](routine-maintenance/under-construction.md)
1. **[The Most Common Problems experienced by CockroachDB users](most-common-problems/README.md)**
1. **[Monitoring](monitoring-alerts/under-construction.md)**
    * [Monitoring tools](monitoring-alerts/under-construction.md)
        - [DB Console](monitoring-alerts/under-construction.md)
        - [System tables](monitoring-alerts/under-construction.md)
        - [Prometheus / Grafana](monitoring-alerts/under-construction.md)
        - [Message and Error Logs](monitoring-alerts/under-construction.md)
    * [Critical metrics](monitoring-alerts/under-construction.md)
        - [Performance monitoring](monitoring-alerts/under-construction.md)
            * [Query Latency](monitoring-alerts/under-construction.md)
        - [Capacity monitoring](monitoring-alerts/under-construction.md)
        - [LSM Storage Shape Monitoring](monitoring-alerts/under-construction.md)
        - [CA Expiry](monitoring-alerts/under-construction.md)
        - [Snapshot Rebalancing Rate](monitoring-alerts/under-construction.md)
    * [Alert Response Procedures](monitoring-alerts/under-construction.md)
    * [Node Health Check](monitoring-alerts/under-construction.md)
    * [Diagnostic and Support](monitoring-alerts/under-construction.md)
        - [CRDB Error Codes and Descriptions](monitoring-alerts/under-construction.md)
        - [Infrastructure Errors](monitoring-alerts/under-construction.md)
        - [Troubleshooting procedures](monitoring-alerts/under-construction.md)
        - [Level 1 (in-house) support procedures](monitoring-alerts/under-construction.md)
        - [Escalation of support to Level 2 (Cockroach Labs)](monitoring-alerts/under-construction.md)
1. **[Emergency Procedures / Operation Continuity](emergency-procedures/under-construction.md)**
    * [Node Recovery](emergency-procedures/under-construction.md)
    * [Manual LSM compaction](emergency-procedures/under-construction.md)
    * [CockroachDB server/VM replacement](emergency-procedures/under-construction.md)
    * [Application Connections Failover and Failback](emergency-procedures/under-construction.md)
    * [Application connection retries](emergency-procedures/under-construction.md)
    * [Application SQL execution retries](emergency-procedures/under-construction.md)
    * [Disaster Recovery: Node Restore](emergency-procedures/under-construction.md)
    * [Disaster Recovery: Database Restore](emergency-procedures/under-construction.md)
    * [Recovery from a Quorum Loss](emergency-procedures/under-construction.md)
    * [Recovery from &quot;Fat Finger&quot; or malicious data alteration](emergency-procedures/under-construction.md)
