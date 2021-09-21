# Procedure: Node Replace

#### Pre-Requisites

Check cluster health?

Make sure cluster is not in a degrade state?

Check if range replication status allows for it (i.e can survive a full failure) 



------

#### Procedure

On any node:

1. [Add a new node](../routine-maintenance/node-add.md) 

2. [Decommission the problem node](../routine-maintenance/node-decommission.md).


Adding a node first provides an opportunity for maintaining SLAs by keeping a sufficient number of vCPUs in the cluster available for workload processing.

