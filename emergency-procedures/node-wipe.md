

# Procedure: Node Wipe

#### Pre-Requisites

Check cluster health?

Make sure cluster is not in a degrade state?

Check if range replication status allows for it (i.e can survive a full failure) 



------

#### Procedure

On the problem node:

1. [Stop the node](../routine-maintenance/node-start-stop.md)
2. Delete the node's store directory

3. [Start the node](../routine-maintenance/node-start-stop.md)
4. Observe the node being "rehydrated" (up-replicated)

