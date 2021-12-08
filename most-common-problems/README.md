## The Most Common Problems Encountered by CockroachDB Users

This document summarizes the recent observations by Cockroach Labs field engineers who provide technical assistance (provisioning, sizing, configuration, tuning, best practices) to CockroachDB customers.

This list outlines the most commonly encountered problems, _by root cause_.



### 1. Insufficient CPU to support the scale of the workload

**Background**

At present, CRDB has a limited workload management capability. A user workload concurrency persistently exceeding the underlying computing resources will eventually render a CRDB cluster unresponsive. An excessive concurrency is characterized by the number of always-active connections (or the number of actively executing statements) that is significantly exceeding the CRDB sizing guidance of 4 times the number of vCPUs (cores) in the cluster.

If the workload concurrency continues to increase much past CRDB sizing guidance, users will first observe a degradation in database response time simply due to the lack of CPU resources. With time, the performance degradation will be exacerbated by the LSM storage tree becoming inverted, leading to the read amplification related performance overhead. LSM compaction will start falling behind – it&#39;s a fairly CPU intensive process primarily due to decompression and recompression of the SST files. Compaction needs CPU to run concurrent worker threads (currently 3 or the number of vCPUs available, whichever is smaller). If compaction falls progressively behind because it is starved of CPU, the cluster stability will be compromised. The affected nodes will start losing liveness heartbeats (liveness state requires raft consensus which requires storage durability) and will be marked `DEAD` or `SUSPECT`. Losing cluster nodes will eventually lead to a quorum loss, although the cluster will probably become &quot;unresponsive&quot; before that.

**Possible Symptoms**

Depending how prolonged the CPU starvation has been, users may observe any of the following

- SQL response time and throughput degradation, not threatening the overall cluster stability. This is the most common manifestation. This condition is easily observable in the DB Console. Note that the CPU graph in DB Console only shows user CPU (system CPU is not reflected in the graph), so if the graph gets over 80%, the actual CPU utilization is closing on 100%. Observe the open transaction/active statement metric to determine the actual workload concurrency and compare to the available vCPU count.
- LSM inversion. This condition can be confirmed by monitoring a custom metric &quot;rocksdb.read-amplification&quot;, observing frequent compaction status messages in the CRDB log, and high read amplification warning in the log.
- Cluster instability, DEAD/SUSPECT nodes.

Note: The above symptoms may, of course, have other root causes. Yet they could be caused by CPU insufficient for the workload, which is a common albeit preventable problem.

**Prevention/Resolution**

- The main measure to avoid CPU starvation is capacity planning, reconciling the workload concurrency requirements with the required number of vCPU in the cluster.
- Implement an application-level workload governance; specifically – use connection pooling to control the workload concurrency. The total maximum number of connections across all connection pools should not exceed 4 x vCPUs.
- Use larger CRDB VMs, the cluster will be more resilient to temporary workload spikes and processing hotspots.
- If LSM compaction falls behind, throttle the workload concurrency back to free up CPU resources to allow the compaction to catch up and restore a healthy LSM shape.



### 2. Insufficient RAM

**Background**

[This blog entry](https://www.cockroachlabs.com/blog/memory-usage-cockroachdb/) written a few years ago still shares the current insights into how CRDB node processes use memory. CRDB&#39;s internal memory use accounting is hardening with every major release, yet given the reality of memory management in golang, precise memory accounting may not be feasible, and we will always have to allow for &quot;other&quot; memory. With insufficient memory, OOM kills become one of the major causes of cluster instability and inconsistent performance.

In a properly designed CRDB cluster, a node process restart does not compromise transactional guarantees. However frequent node restarts, particularly after an abrupt process exit, will add resource use overhead during node recovery, potentially impacting the database performance consistency and SLA.

**Possible Symptoms**

- Node process re-starts. In Kubernetes environments pod restarts may be regarded as &quot;routine&quot; in a containerized environment and left unattended. While the cluster will recover from an OOM node process kill, the condition causing OOM should not be allowed to persist. CRL advises to monitor dmesg or K8s logs for OOM.
- Query &quot;blast radius&quot;. An unbound/untuned query may acquire a significant amount of resources, primarily memory, thus impacting other users&#39; query workload. CRL advises to monitor resource utilization by queries. The &quot;blast radius&quot; of a [inadvertently] bad query can only be minimized by killing the offending query.

**Prevention/Resolution**

- OOM is a rare occurrence as long as the VMs are provisioned according to CRL best practices.
  - The recommended memory-to-vCPU ratio for a production environment is 4GB per vCPU.
  - The lowest acceptable memory-to-vCPU ratio is 2GB per vCPU, which is only suitable for testing.
- If all CockroachDB VMs are provisioned with [sufficient RAM](https://docs.google.com/document/d/15mIY6z3ifo_z-kPaT45T0mnHR2Zy2vvScVScb-adzPs/edit#heading=h.35nkun2), more memory can be allocated for [data caching at the database level](https://www.cockroachlabs.com/docs/v20.1/recommended-production-settings.html#cache-and-sql-memory-size) by setting _--cache_  and _--max-sql-memory_ each to 35%, i.e.:
  - _cockroach start --cache=.35 --max-sql-memory=.35 <other start flags>_
- CRL recommends disabling swap on the CRDB servers.



### 3. Insufficient Disk IO performance

**Background** 

Under-configured or under-provisioned disk storage is a common root cause of inconsistent CockroachDB cluster performance.

**Possible Symptoms**

- Poor SQL performance across the board
- Poor bulk load performance
- Mis-shaped LSM

**Prevention/Resolution**

- Provision the storage for CRDB data as follows:
  - The recommended storage throughput MBPS-to-vCPU ratio is 30 MBPS per vCPU.
  - The recommended storage IOPS-to-vCPU ratio is 500 IOPS per vCPU.
- Users can detect underperforming storage in CockroachDB Console-\&gt; Metrics -\&gt; Dashboard: Hardware. Observe &#39;&#39;Disk IOPS In Progress&#39;&#39; graph. The metrics on this graph is actually the device request queue, corresponding to `avgqu-sz` in the `iostat` device status output. Ideally, this value should be 0. A low single digit value for a short period of time does not indicate a problem. Yet a double-digit value observed persistently on any node indicates a storage device saturation - a likely symptom of under-provisioned or mis-configured storage. To confirm, continue the suspected IO bottleneck investigation with guest Linux tools like iostat.
- For best performance and operating stability, CockroachDB data should be placed on a dedicated volume that is not shared with any other IO on a server.
- Place [CRDB message logs](https://www.cockroachlabs.com/docs/stable/configure-logs.html) on the OS drive.
- Cockroach Labs recommends not using LVM in the IO path. The main concern is not necessarily in a performance overhead on an initially provisioned volume, but LVM may encourage users to dynamically grow CockroachDB store volumes and that is prone to significant storage performance degradation. Also using LVM snapshots in lieu of CockroachDB backup/restore is not supported.



### 4. Contention

**Background**

CRDB always uses [serializable isolation](https://www.cockroachlabs.com/docs/stable/demo-serializable.html), eliminating a possibility of non-repeatable and phantom reads. The strongest isolation adds a read-write contention phenomenon, in addition to write-write contention.

Reads can also be forced to restart due to the clock skew uncertainty. A read generally returns the latest tuple with a timestamp earlier than the reader&#39;s timestamp but if an open write transaction is within an uncertainty interval of the read transaction, the latter will be forced to re-start at a later timestamp. The restarts will continue until the read transaction either succeeds or gets canceled by the client.

An application can mitigate both write-write and read-write contention with its design choices.

**Possible Symptoms**

- Transaction contention most commonly comes across as a performance issue. Contention can affect all SQL statements, including SELECTs. Read and write transactions encountering a write lock will be queued up for execution pending that lock release. A long [wait queue](https://www.cockroachlabs.com/docs/stable/architecture/transaction-layer.html#txnwaitqueue) of contending writers will result in a delayed response.
- A high statement / transaction retires count observed in [DB Console](https://www.cockroachlabs.com/docs/stable/ui-statements-page.html).
 If an application implements single statement transactions with a small result set, CRDB makes the best effort to auto-retry on the server and the client will experience a relatively limited increase in the response time. However, in some circumstances the server has to push the retry to the client, which would increase the cost of each retry.
- Transaction failure, if a client application did not implement the [retry logic](https://www.cockroachlabs.com/docs/stable/transactions#client-side-intervention)

**Prevention/Resolution**

- If the application design allows historical reads, use the [&quot;follower read&quot;](https://www.cockroachlabs.com/docs/stable/follower-reads.html) queries to eliminate read-write contention.
- Use multiple [column families](https://www.cockroachlabs.com/docs/stable/column-families) to avoid contention between writes to different columns within the same row.
- Use [SELECT FOR UPDATE](https://www.cockroachlabs.com/docs/stable/select-for-update) to reduce retries when a transaction reads and then updates the same row(s).
- If a &quot;fail fast&quot; approach fits the application design, [SELECT FOR UPDATE … NOWAIT](https://www.cockroachlabs.com/docs/stable/select-for-update#wait-policies) can reduce or prevent the contention by returning an error if a row cannot be locked immediately.
- For greater response time predictability under severe per-key contention, consider limiting the maximum wait queue size with [_kv.lock\_table.maximum\_lock\_wait\_queue\_length_](https://github.com/cockroachdb/cockroach/pull/66146) cluster setting. If an existing lock wait-queue is already longer than that, a new transaction will be quickly rejected instead of entering the queue and waiting.
- If the [system clocks](https://www.cockroachlabs.com/docs/stable/recommended-production-settings.html#clock-synchronization) are tightly synchronized, consider lowering the `--max-offset` startup flag to 250 ms (500 ms by default) to reduce the probability of read transactions&#39; restarts due to clock skew uncertainty.
- Design the transactions for the best opportunity for [automatic retries](https://www.cockroachlabs.com/docs/stable/transactions.html#automatic-retries) (conditions for automatic retries explained in that doc chapter). Server side automatic retries have lesser performance overhead than client side retries.
- Follow the CRL&#39;s [best practices for transaction design](https://www.cockroachlabs.com/docs/stable/performance-best-practices-overview#understanding-and-avoiding-transaction-contention).



### 5. Hotspots

**Background**

Hotspots, or simply put - overloaded nodes, can develop in a distributed data system for 2 reasons:

- Due to a data distribution pattern incompatible with its usage. For example, when a time series data is range-partitioned by time while an application is mainly operating on recent data.
- Or due to a highly localized processing driven by the application logic. For example, when an application is spot-processing a lot of activity for the same identifier.

**Possible Symptoms**

- SQL response time increases proportionally to concurrency increase. This means that the maximum throughput of that SQL reached a plateau, so increasing the number of concurrent connections results in latency increase as no more productive work can be done.
- No scaling after a cluster expansion. With a single node being a bottleneck, a cluster would not be able to process more workload with more nodes added, as should be expected.
- A cluster node showing signs if instability – marked as &quot;SUSPECT&quot; or &quot;DEAD&quot;, node process restarts.
- Overall cluster performance degradation. This can happen with mixed workloads. Distributed queries [that run at the speed of the slowest node] may be widely affected by one node overloaded with spot processing on a &quot;hot&quot; key.

**Prevention/Resolution**

- Follow CockroachDB best practices for [primary keys and unique indexes](https://www.cockroachlabs.com/docs/stable/performance-best-practices-overview#unique-id-best-practices).
- Avoid inadvertently creating hot spots on secondary indexes. For example, adding a secondary index on a timestamp to support fast deletes may result in a hotspot during inserts.
- Monitor hotspots developing, react pro-actively before an overloaded node threatens wider cluster stability.
- Limit/govern the concurrency of application components that can spot-process a large amount of activity on a single key
- Use [hash sharded indexes](https://www.cockroachlabs.com/docs/stable/hash-sharded-indexes.html) on sequential keys to distribute the workload across nodes and avoid hotspots. Beware that the hash sharded indexes don&#39;t work well when the workload includes scan queries. As a rule of thumb, a hash sharded index could be considered when the required write throughput on sequential keys is beyond what a single range can sustain (around 2k ops/s).
- [Pre-split](https://www.cockroachlabs.com/docs/stable/split-at.html#why-manually-split-a-range) data ranges manually to avoid hotspots when the data set is small and fits into a small number of ranges, which would be prone to hotspots. For example, when a new database / table was just created.
- If the application design allows historical reads, use the [&quot;follower read&quot;](https://www.cockroachlabs.com/docs/stable/follower-reads.html) queries to avoid hotspots. Follower reads are serviced by the closest replica, regardless of the replica&#39;s leaseholder status, distributing concurrent read access to the same data across cluster nodes.



### 6. Connection Disruptions

**Background**

An application environment using a CRDB cluster would commonly include two 3rd party components for distinctly different functions:

- load balancers for connection balancing, failover, and failback
- connection pooling for connection governance and to eliminate connect/disconnect overhead

A robust, validated, redundant configuration of load balancers and/or connection pools is required to eliminate the 3rd party components as the weakest spot in the entire environment, undermining the SLA.

**Possible Symptoms**

- EOF errors received by an application client. Often related to maintenance events, but more often sporadic.

**Prevention/Resolution**

- EOF on a connection error at a client means that either the server or a &quot;man-in-the-middle&quot;, such as a load balancer, closed the tcp connection. The most likely reason is a low idle connection timeout in a load balancer or in a connection pool. Often, it&#39;s a default that hasn&#39;t been changed. For example, the default idle timeout in AWS ELB is 60secs - a good setting for web servers but pretty bad for databases. If that isn&#39;t the case, check statement/transaction/session timeouts in CRDB. They are off by default but perhaps were set. CRL recommends configuring the idle timeouts (there could be one for each side - client and server) to be MUCH larger than the slowest possible SQL operation, for example 10 or even 30 minutes.
- Load balancer configuration should be dynamic and be included into regular maintenance procedures. Such as online upgrades, node decommissioning, adding new nodes, etc. Application connectivity disruptions during an online CRDB cluster upgrade is a preventable yet a common example of an oversight in an operating procedure.
