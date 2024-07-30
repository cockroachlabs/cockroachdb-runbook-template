# Operational Preparedness: Day One / Two

### Overview

This article offers Operational Preparedness (a.k.a. Readiness) ***Day One and Day Two task list*** templates for CockroachDB cluster deployments. Operators may leverage these task lists as a starting point for building custom deployment plans driven by the organization's data services SLAs.

Operational preparedness is the *operator's ability to deploy, maintain, and operate CockroachDB environments*. A measure of customer's operational readiness may be the number of P1/P2 CockroachDB Technical Support issues open monthly. 

> ✔️  This article is primarily designed to assist Operators of *self hosted* CockroachDB environments.



### "Day N" Nomenclature

Operational preparedness can be prioritized into two lists of required tasks, commonly referred to as "Day 1 and "Day 2" preparedness (readiness).

Completing the tasks to meet requirements of "Day 1" and "Day 2" phases is a prerogative of IT organizations.

**"Day 1"** is the scope of minimally required technical enablement and procedural preparedness before CockroachDB service is ready for a production deployment. "Day 1" is effectively *MVP of Operational Preparedness*, consisting of tasks that *must* be completed before the initial "go live" date.

**"Day 2"** is the scope of activity to achieve a meaningfully *complete Operational Preparedness*, with automation of all daily operations routines. This phase is a continuation from the Day 1 preparedness baseline.



>  👉  A term "Day 0" is often used in reference to the Application development phase by teams responsible for the Application Architecture and Engineering . That phase is outside the scope of this article - herein the Application tier development and testing is assumed "completed".



The practical purpose of defining Day 1 and Day 2 preparedness is to *safely* bring the initial deployment date forward. By focusing on Day 1 requirements up front, operations teams can prevent end-user / applications problems from occurring after the CockroachDB service goes live.

The boundary between Day 1 and Day 2 tasks can be generalized, as in the proposed task list templates, however each deployment plan is unique, driven by specific customer requirements and SLA definitions. While developing the custom deployment plan from this template, operators may promote or de-prioritize a task by moving it between Day 1 and Day 2 lists, and add new or remove impertinent tasks to either list.

The Day 1 and Day 2 definitions are best presented as a categorized, but not necessarily a prioritized task list, since all task activities must be completed to claim a given level of preparedness.



### Day 1 Task List (Template)

Below is a categorized list of Day 1 preparedness tasks. The order of items does not imply priority.



- [ ] Technical / implementation completeness. To proactively mitigate risks to continuous service availability. While CockroachDB delivers supreme data protection and liveness, all adjacent infrastructure components must also be hardened to the same level of resiliency in order for the entire CockroachDB -based environment to provide uninterrupted data service. These infrastructure components include (1) the underlying platform (Linux, Kubernetes), (2) CockroachDB cluster (software), (3) connection management (encompasses connection pooling and load balancing), and (4) application level error handling.

  Tasks in this section:

  - [ ] SLA definition for normal operating conditions and for each infrastructure failure scenario that must be tolerated. This pre-requisite document frames all technical tasks in this section. Sizing, provisioning and tuning of a data processing environment is only possible when an operator knows the goals to configure and tune for. SLA is commonly expressed as SQL response time at required level of workload concurrency. To emphasize - the SLA definition needs to go beyond just the [scenarios to survive](https://bramgruneir.github.io/); it also needs to quantify the performance expectations when a cluster is in a degraded configuration. For illustration, in case of a regional failure in a common 3 region configuration - a cluster will lose 1/3 of processing resources and may be unable to handle the nominal workload.
  - [ ] Cluster Sizing. CockroachDB is intrinsically elastic. Adding/removing nodes and modifying the cluster topology is a quick on-line procedure. However right-sizing the cluster for an initial deployment is a critical task to ensure CockroachDB cluster performance and stability. To eliminate disruptive cluster overloads, the concurrency of SQL workloads across all applications should be reconciled with the cluster's processing and storage capacity.
  - [ ] Platform provisioning, per CockroachDB best practices and validated deployment references. This task applies to self hosted CockroachDB environments only. Cockroach Labs provides provisioning guidance for all validated deployment platforms, and is prepared to conduct a complete platform health check. A Linux level platform health check report covers relevant Linux OS configuration parameters, compute (CPU and RAM) provisioning, storage provisioning, cluster startup configuration. A separate Kubernetes platform health check if offered in addition (self-hosted) or instead of (Google Kubernetes Engine, Amazon Elastic Kubernetes Service) Linux platform health check.
  - [ ] Data Availability. [This article](./data-availability.md) provides planning guidance for configuring a CockroachDB cluster to handle possible fail-stop and gray failures in the underlying infrastructure, including blast radius controls, maximum disruption controls, partial and asymmetric network failures handling, disk stalls handling.
  - [ ] Connection management is a critical infrastructure component, contributing to the continuous service availability. [This article](./connection-management.md) provides implementation blueprints for connection pooling and load balancing.
  - [ ] Application level error handling is another critical component that must be implemented per CockroachDB guidance and best practices. It should include proper handling of routine rolling maintenance procedures, specifically orderly [node shutdowns](../routine-maintenance/node-stop.md#avoiding-application-service-interruptions-due-to-node-shutdown), as well as handling or runtime errors with retries and reconnects, [as recommended](https://www.cockroachlabs.com/docs/stable/transaction-retry-error-reference).

- [ ] Monitoring
  
  Tasks in this section:
  
  - [ ] Essential metrics. [This article](../monitoring-alerts/monitoring-dashboard-custom.md) provides a curated list of CockroachDB metrics to monitor in operator's dashboard. Each metric in the list has a narrative, explaining how to make a practical/actionable use of each metric in a production deployment.
  - [ ] Essential alerts.
  - [ ] System Activity Report Configuration (SAR) should be configured and continuously run on all CockroachDB node servers or VMs by operators of CockroachDB on self hosted platforms, as recommended in [this article](../monitoring-alerts/sar.md).
  
- [ ] Security and Compliance. Regulatory and Other Compliance
  
  Tasks in this section:
  
  - [ ] Mandatory Compliance items vs troubleshooting preparedness
  - [ ] Security (encryption at rest, audit, ...)
  - [ ] Backup, if compliance item
  
- [ ] Enablement. Operators of a CockroachDB cluster (Cockroach Labs customer) shall receive training to be able to perform routine maintenance and to provide Level 1 (immediate) technical support within the operator's organization. Competency to provide Level 1 support includes ability to execute alert response procedures and triage technical issues in the environment.
  
  Tasks in this section:
  
  - [ ] A CockroachDB operations manual, a.k.a. a runbook, is a pre-requisite collateral for cluster deployment. Operators  may leverage this [runbook template](https://github.com/cockroachlabs/cockroachdb-runbook-template) for creating a custom CockroachDB runbook.
  - [ ] Automation of routine maintenance procedures affecting all CockroachDB cluster nodes (a.k.a. rolling maintenance) is required to avoid inadvertent operator's errors. Realistically, it's a hard requirement for larger clusters, over 20 nodes. Procedures affecting one or few cluster nodes may be handled manually if step-by-step actions are documented.
  - [ ] Cluster health pre-check, as suggested in [this article](../routine-maintenance/maintenance-pre-check.md), shall be integrated into all routine maintenance procedures as a pre-requisite initial step.
  - [ ] Alert response procedures
  - [ ] Node start, stop
  - [ ] Troubleshooting / triage
  - [ ] Level 2 technical support - escalation to Cockroach Labs Tech Support - Ideal ticket - https://cockroachlabs.atlassian.net/wiki/spaces/TS/pages/3063382143/The+Anatomy+of+a+Good+Ticket






### Day 2 Task List (Template)

Day 2 builds on Day 1 task list, with emphasis on completeness and automation of all daily operations routines.



Automation - focus/main theme of day 2... as clusters grow, over 50 must automate all   themes are operations runbook completeness and automation of all procedures end to end

- [ ] Remaining runbook procedures. As production CockroachDB clusters grow past 30 nodes, end-to-end automation of all cluster operations become necessary.

  Tasks in this section:

  - [ ] Fine tune cluster sizing, topology
  - [ ] Backup, if not a compliance item
  - [ ] Restore
  - [ ] Upgrades
  - [ ] Cluster expansion, contraction - Node add, remove
  - [ ] Topology modifications procedures
  - [ ] Expired security Certificates renewal
  - [ ] Custom monitoring integration (e.g. custom Grafana, sql execution statistics via Datadog)

