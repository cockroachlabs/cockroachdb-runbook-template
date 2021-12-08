
# Procedure: Node Shutdown (Stop)

### Purpose of this Procedure

This procedure is an integral part of all regular maintenance procedures that requires a CockroachDB node process restart. For example, a rolling software upgrade.

The first and the most essential phase of the shutdown is a *node drain*, during which a node orderly ends processing of transactions, closes client connections, transfers its range leases and stops ranges from rebalancing onto the node.

During a node *shutdown*, a *node drain* is followed by a node *process termination*.

A user can initiate just the node drain (without shutting down the node process) with the `cockroach node drain` [command](https://www.cockroachlabs.com/docs/v21.2/cockroach-node.html). Make a note of an available `--drain-wait` flag and its default value (`10m` as of v21.2).



### How to Trigger a Node Shutdown

A node shutdown is triggered by sending `SIGTERM` or `SIGINT` signal to the node process. Pick your method:

- If a CockroachDB node process is running as a service, use Linux service manager to send the signal to stop.
- If a CockroachDB node is running as a foreground process, press ctrl-C in the terminal.
- If a CockroachDB node is running as a background process, use `kill <pid>`.
- If using Kubernetes orchestration, use `kubectl` to signal the CockroachDB pod termination.

After a signal initiates a graceful shutdown, further `SIGTERM` signals are ignored. However, a `SIGINT` received during a node drain will result in an abrupt immediate termination of the node process, i.e. in a hard shutdown. A hard (disorderly) shutdown will require a node to undergo a recovery process when it is re-started. An impatient hard shutdown is not likely to save the total time of a node re-start.

A `SIGKILL` signal, by definition, always results in a hard shutdown (process kill).  

```
The best operating practice is to always ensure a graceful database cluster node shutdown.
```



### How Node Shutdown (Drain) is Implemented

A CockroachDB node drain is a sequence of steps executed *consecutively*:

1. An initial period during which the node is processing everything normally, however the health check `/health?ready=1` starts returning false. This period is designed to allow load balancers and connection management tools/libraries to mark the node as down and start routing connection traffic to available cluster nodes.  A user can control the duration of this step with a `server.shutdown.drain_wait` cluster setting, `0s` by default. If configured, the node will remain in this state for at least the specified amount of time. There is no early exit from this wait.
2. A period to complete the processing of open transactions initiated on this node, i.e. transactions for which this node is a gateway. During this period the node does not accept new connections nor new transactions, but continues to execute its open transactions and distributed statements from transactions initiated on other gateway nodes. The elapsed time of this step is the time to complete the longest running transaction initiated on this node before the drain was initiated, or the maximum duration of this step configured with the `server.shutdown.query_wait` cluster setting, `10s` by default, whichever is smaller. At the end of this step, all client connections to the node are closed by the server. Transactions that were not admitted by the server or were interrupted because of the forced connection closure will fail with error [57P01](https://www.postgresql.org/docs/13/errcodes-appendix.html) `server is shutting down` or error [08006](https://www.postgresql.org/docs/13/errcodes-appendix.html) `An I/O error occurred while sending to the backend`. Ideally, before this step completes, all node connections are closed and failed over *by the client* side connection management so the server does not need to forcibly close them. 
3. A period to transfer all range leaseholders and raft leaderships away to other nodes. To give a node an ultimate opportunity to shutdown gracefully, this step, by design, does not have a user configurable maximum elapsed time. It is implemented as a "forever" loop where the maximum duration of individual iterations can be configured with the `server.shutdown.lease_transfer_wait` cluster setting, `5s` by default. This step will exit as soon as all transfers are completed. A graceful node shutdown is predicated on an orderly completion of this step. Although not recommended, this step can be interrupted externally to the node process by a signal.



##### Notable points:

- CockroachDB Node Shutdown behavior does not match any of the [Postgres Shutdown Modes](https://www.postgresql.org/docs/current/server-shutdown.html). 
- All three cluster settings controlling the individual drain periods have *different semantics*:
  - `server.shutdown.drain_wait`  is the minimum amount of time the drain step (1) can elapse
  - `server.shutdown.query_wait`  is the maximum amount of time the drain step (2) can elapse
  - `server.shutdown.lease_transfer_wait` only controls how frequently a progress message in printed in the message log; there is no cluster setting that controls the amount of time the drain step (3) can elapse.
- Configuring `server.shutdown.drain_wait` and `server.shutdown.query_wait` settings to `0s` would inevitably be disruptive to the user workload.
- The  `server.shutdown.lease_transfer_wait` should *not* be set to `0` under any circumstances. If set to `0` , the drain process will iterate indefinitely (until killed) while doing noting in each iteration. `0` setting would defeat all opportunities for an orderly shutdown.



### Avoiding Application Service Interruptions due to Node Shutdown

To ensure uninterrupted application service during a rolling cluster maintenance, implement *<u>all</u>* of the following best practices *across the entire stack* - in the application retry logic, system service or container termination grace period, connection management layers, CockroachDB cluster settings.



##### System Service or Container Termination Grace Period Settings

An orderly node drain can be adversely affected by a low settings in a OS service control or container orchestration configuration, disrupting a graceful node shutdown.

If a CockroachDB node process is configured to run as a systemd service, the setting `TimeoutStopSec` in a service configuration file defines the termination grace period. When a *service stop* command is issued, a `SIGTERM` is sent to the process, initiating the CockroachDB Node shutdown sequence. If the node process doesn't complete its shutdown during the grace period, the node process is hard killed with a `SIGKILL`.

If CockroachDB cluster deployment is orchestrated with Kubernetes, the functionally equivalent configuration setting controlling CockroachDB pod termination is `terminationGracePeriodSeconds`.

The best operating practice is to set the termination grace period in the process control reasonably high, providing for an orderly node draining and shutdown, which would eliminate the need for recovery/cleanup on the next node start up.

Set the grace period, `TimeoutStopSec` or `terminationGracePeriodSeconds`, as follows:

- a minimum setting is 300 seconds (5 minutes)
- correlate with the `cockroach node drain --drain-wait` setting used in the same operating environment; by default, the `cockroach node drain` command waits for up to 10 minutes for the drain process to complete orderly
- avoid both - excessively low and excessively high extremes; a reasonable setting is generally in a 5-10 minutes range

Setting a termination grace period higher does not increase the elapsed time of a node shutdown, beyond the actual time to execute the last open transaction and to transfer all range leaseholders. However, a termination grace period should not be excessively large, to provide a safety hedge in case a drain process is "stuck" due to some underlying hardware of software issues.



##### Application Retries

In addition to the client side [transaction error](https://www.cockroachlabs.com/docs/v21.2/transaction-retry-error-reference.html) retires, an application is required to implement a retry logic for errors it will receive during a node shutdown, as designed. These errors are [57P01](https://www.postgresql.org/docs/13/errcodes-appendix.html) `server is shutting down` and [08006](https://www.postgresql.org/docs/13/errcodes-appendix.html) `An I/O error occurred while sending to the backend`.

The above errors indicate that the current connection is broken and is no longer usable. If a connection is broken while a transaction was open, the application has to be prepared to handle the state of that transaction as *unknown*. A transaction on the server may complete successfully, however an application could receive a broken connection error. For example, once a transaction execution enters a commit phase on the server, it will run to completion, successfully committing or not, even if the client connection goes away.

To handle this scenario, an application needs to implement the error handling logic, treating the result of the transaction as ambiguous. In a loop, as follows:

- Close the the current connection
- Open a new connection
- Reissue the transaction on a new connection. If the transaction included a database write, the retry logic can't assume whether or not the previous transaction execution resulted in a committed change. E.g. rely on a primary key uniqueness constraint and re-issue an insert transaction, treating a duplicate primary key error as a success. 
- If a shutdown related error persists, and the number of retires did not exceed some configured maximum - repeat



##### Connection Management

A meaningfully complete implementation of the best practices for [connection management](../system-overview/connection-management.md) is a pre-requisite for uninterrupted application service during CockroachDB node shutdown for maintenance.

Here a digest of essential connection management points related to shutdown handling: 

- A connection *load balancer*, if deployed in the environment, needs to be configured with active health checks to monitor the health of CockroachDB nodes, promptly maintain the operational state of nodes in the pool, and direct the connection traffic to online nodes. And the cluster setting `server.shutdown.drain_wait` should be coordinated with the load balancer's active health monitor configuration options. Details included in a chapter below.
- The application stack needs to implement a robust connection failover and failback. That implies a re-connect logic, which can be configured in a connection pool, if deployed in the environment, or otherwise implemented in the application error handling logic wrapping database driver calls.
- The application stack needs to implement an active connection rebalancing mechanism, to maintain an even distribution of connections across cluster nodes. Otherwise when a node is restarted, it will not be utilized to the full potential until it gets its share of application connections. Active connection balancing can be achieved, for example, by setting a maximum connection lifetime is a connection pool, if deployed in the environment. 



##### Cluster Setting:  `server.shutdown.drain_wait` 

This setting should provide a connection management facility, in particular a load balancer, with a sufficient amount the time to mark the draining node as *down* and fail the affected connections over to the online nodes.

The default setting of `0s` does not give a load balancer the time to make appropriate adjustments, so the `server.shutdown.drain_wait` should be custom set in coordination with the load balancer's settings.

For example,  a HAProxy load balancer configured for active health checks has reasonable default settings :  `... check  inter 2000  fall 3 ...`
Which means HAProxy will consider a node *down* and will temporarily remove the server from the pool after 3 consecutive unsuccessful health checks with 2 seconds interval between the checks.
Thus in general, a fair calculation for the `server.shutdown.drain_wait` when using HAProxy would be:

```
server.shutdown.drain_wait > ( (fall + 1) * inter )
```

 E.g. `server.shutdown.drain_wait='8s'` or greater for the default HAProxy health check setting would be sensible. However users should be careful not to set this setting high because the drain process with wait unconditionally for that amount of time.



##### Cluster Setting: `server.shutdown.query_wait` 

This setting should provide a sufficient time for all transactions in progress to complete. Therefore the `server.shutdown.query_wait`  should be greater than any of the following:

- The longest possible transaction in the workload expected to complete successfully under normal operating conditions
- `sql.defaults.idle_in_transaction_session_timeout` cluster setting
- `sql.defaults.statement_timeout` cluster setting

Setting `server.shutdown.query_wait` to a high value does not necessarily increase the elapsed time of a node drain because the `query_wait` will end as soon as the last open transaction completes.



**Cluster Setting:`server.shutdown.lease_transfer_wait`**

This setting does not have a material impact on the node drain process, it only controls how frequently a progress message in printed in the message log.

In practice, the transfer of ranges leaseholders in a cluster provisioned per Cockroach Labs sizing and configuration guidance, takes anywhere from tens of seconds to a small number of minutes. The default setting  for `server.shutdown.lease_transfer_wait` works well in most cases. For large cluster nodes with high storage density (many ranges), this setting may be increased towards `20s` to reduce the message log chatter.

As a reminder, do not set `server.shutdown.lease_transfer_wait` to `0` under any circumstances.





