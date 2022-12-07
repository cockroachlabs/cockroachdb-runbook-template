# Procedure: Manual LSM compaction

### Pre-Requisites

<pre-requisites?>



------

### Procedure

On the problem node:

1. [Stop the node](../routine-maintenance/node-stop.md)

2. Run a manual offline compaction for a problem node suffering from read amplification by issuing the following command:

   `cockroach debug compact`

3. [Start the node](../routine-maintenance/node-start.md).  

