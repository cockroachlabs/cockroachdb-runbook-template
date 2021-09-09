
 **< UNDER CONSTRUCTION >**

### Server / VM Replacement

[From](https://cockroachlabs.atlassian.net/wiki/spaces/CS/pages/2156822576/Runbook+to+patch+Hypervisor+hosts)

Company C runs their nodes on Hypervisor VMs…. and the VMs need to be patched for security or other reasons. A Change Request window is scheduled for 6 hours and it usually takes ~3 hours to patch the VMs. Taking a node down for a few hours has caused the Production cluster to become unavailable, and can push the envelope on CockroachDB&#39;s resiliency. This runbook outlines the steps to take the node down gracefully and to bring it back successfully.

Steps

Verify current rebalance/recovery rate. Increase the rate to 64MB to speed up upreplication in step #4.

set server.time\_until\_store\_dead=2m to shorten the wait time from 5 mins

(One hour before CR begins) use KILL -15 ( SIGTERM) to gracefully shutdown a node.

Alternatively for maximum cluster availability, use cockroach node drain before shutting down the node. It waits for active SQL queries to finish and leaseholders to move away.

Drain nodes of SQL clients, distributed SQL queries, and range leases, and prevent ranges from rebalancing onto the node. This is normally done by sending SIGTERM during node shutdown, but the drain subcommand provides operators an option to interactively monitor, and if necessary intervene in, the draining process.

Also see the Note from Cockroach Document:

The amount of time you should wait before sending SIGKILL can vary depending on your cluster configuration and workload, which affects how long it takes your nodes to complete a graceful shutdown. In certain edge cases, forcefully terminating the process before the node has completed shutdown can result in temporary data unavailability, latency spikes, uncertainty errors, ambiguous commit errors, or query timeouts. If you need maximum cluster availability, you can run cockroach node drain prior to node shutdown and actively monitor the draining process instead of automating it.

4. After 2m, the node is considered dead. And cluster starts to upreplicate.

- Under RF3, the cluster is vulnerable to any additional node failure in this stage
- Watch underreplicated ranges to increase to 5K and wait till it reduce to 0
- when increasing to 64MB it took 6-9 hr to fully replicate
- when not fully upreplicated it seems to lead to cluster instabilities when bringing the node back

5. Patch HyperVisor VM (estimated 3hrs)

6. Wait until cluster has fully replicated, then reduce the rebalance/recovery rate and server.time\_until\_store\_dead to previous values to avoid overwhelming the incoming node

7. Set cluster setting kv.allocator.min\_lease\_transfer\_interval=30min;

8. Bring the node back online

9. Watch range snapshots being generated in replication dashboard to come down to zero and then set cluster setting kv.allocator.min\_lease\_transfer\_interval to original value &#39;1s&#39;
