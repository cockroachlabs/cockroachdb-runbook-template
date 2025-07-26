# Troubleshooting: Gray Network Failures in Cloud Platforms



## 1. Background

A "*gray failure*" is an ambiguous network connectivity disruption where a component doesn't completely fail, but its performance degrades subtly, making it difficult to detect or diagnose. Unlike a hard failure where connectivity is completely lost, gray failures manifest as performance issues, typically due to increased error rates, packet losses, and/or TCP retransmits.

Intermittent gray failures are relatively common in large scale global [networks operated by public cloud](https://docs.aws.amazon.com/whitepapers/latest/advanced-multi-az-resilience-patterns/gray-failures.html) service providers. On-prem network connectivity disruptions are predominantly hard failures, as a result of some physical equipment faults.

Service Level Agreements (SLA) made by public cloud service providers are typically based on definitions that equate "unavailability" to no external connectivity, i.e. to hard failures. Since gray failures are not counted towards SLA commitments, cloud providers typically don't offer observability into gray failures and their technical support does not acknowledge gray connectivity disruptions as "failures". This means that customers operating in the cloud have to rely on their own tooling for triage of gray connectivity issues. Cockroach Labs strongly advises operators to enable [SAR](https://github.com/cockroachlabs/cockroachdb-runbook-template/blob/main/monitoring-alerts/sar.md) with expanded metrics collection to capture [TCP/IP network errors](https://github.com/cockroachlabs/cockroachdb-runbook-template/blob/main/monitoring-alerts/sar.md#3-expand-default-sar-metrics-collection). CockroachDB operators that are also leveraging [Datadog NPM](https://www.datadoghq.com/blog/cloud-service-autodetection-datadog/), which provides insights into gray network failures similar (yet more aesthetically pleasing) to SAR reports.



## 2. Gray Failures Symptoms

Gray network failures are usually transient, lasting for less than 20 seconds, often less than 10 seconds. CockroachDB is designed to survive gray failures and minimize their impact on database workload.

> 👉  All symptoms listed below can be due to various causes. By themselves, these symptoms may not be a proof of gray network failure, but merely some of possible symptoms. To prove the symptoms are caused by gray network failures, they must be correlated to network reports, following the [triage](#3.-tirage) protocol.

#### Observable in DB Console

In no particular order:

- Momentary [node heartbeat latency](https://www.cockroachlabs.com/docs/stable/common-issues-to-monitor#node-heartbeat-latency)  [`liveness.heartbeatlatency-p99`](https://www.cockroachlabs.com/docs/stable/ui-distributed-dashboard#node-heartbeat-latency-99th-percentile)  spike over 1 second. An example below shows only one node of the cluster experiencing network connectivity issues in a real-world production deployment. ![](C:\Dev\Runbook\cockroachdb-runbook-template-main - work\diagnostic-support\res\node-hearbeat-laterncy.png)

  

- A  [live node count](https://www.cockroachlabs.com/docs/v25.2/ui-runtime-dashboard.html#live-node-count)  [`liveness.livenodes`](https://www.cockroachlabs.com/docs/v25.2/essential-metrics-self-hosted#liveness-livenodes)  showing any number of down nodes. An example below shows 10 nodes of the cluster affected by a gray network connectivity issue in a real-world production deployment. The cloud provider was performing an overnight maintenance service of some underlying network switching equipment that affected connectivity between 2 regions. TPC/IP connections were automatically failing over to redundant network circuits, however due to TCP/IP timeouts, CRDB was promptly detecting and reporting momentary liveness issues.

  ![](C:\Dev\Runbook\cockroachdb-runbook-template-main - work\diagnostic-support\res\live-nodes.png)

- Under-replicated or unavailable ranges. Under-replicated ranges is a more common symptom of a gray network failures because they typically have a narrow impact on a small number of CockroachDB VMs. Unavailable ranges may be reported in a less common yet possible situation when a gray failure affects multiple cluster nodes.

- Momentary P99 SQL latency (response time) increase



#### Observable in CockroachDB message/error log:

Operator should scan the CockroachDB messages in the log and note their timestamps to triage a potential networking disruption: 

- Warning `slow heartbeat took _._________s` is a symptom of unhealthy cluster nodes which may escalate to node liveness misses, which may subsequently lead to under-replicated or unavailable ranges.
- Error messages containing `failed node liveness heartbeat` , and/or  `failing ping request from node n__`  indicate a node liveness heartbeat miss (failure), which would result in `SUSPECT` node status and consequently lead to under-replicated or unavailable ranges.
- Message `connection is now healthy` indicates a prior liveness issue has cleared.



## 3. Tirage

The recommended triage protocol for troubleshooting a suspected gray network failure:

1. Observe the [symptoms](#2.-gray-failures-symptoms) via CockroachDB Console and message (error) log. Note the timestamps of the pertinent events. These timestamps will need to be correlated to the Linux metrics reported by external tools, so it's best to note all times in UTC.

2. Locate the raw SAR files on all CockroachDB VMs for the day of the incident. For example, on Red Hat Linux VMs, the raw `sar` data files are located in the `/var/log/sa/` directory by default. The file name for the day is `sar<dd>`, where `<dd>` is the two-digit day of the month. You will need to generate reports for all CockroachDB VMs in the cluster, so you may copy all raw `sar` files for the day (one file from each CockroachDB VM) to a single staging directory on any VM running the same version on Linux. Prepend the file names with the host/node suffix to avoid overwriting files.

3. Generate a graphical **ETCP** statistics report about TCPv4 network errors. Note that TCPv4 statistics depend on sadc option "-S SNMP" to be collected. If you are unable to generate this report, the most likely reason is you failed to follow the CRL recommended  [SAR](https://github.com/cockroachlabs/cockroachdb-runbook-template/blob/main/monitoring-alerts/sar.md)  configuration guidance.
   Generate the graphical reports for all CockroachDB VMs per the following psudo code:

   ```
   # Network interfaces used by CRDB nodes, optional, can be blank
   #network_interface="--iface=eno12399,eno1,eno2,eno3"
   network_interface="--iface=eth0"
   ...
   mkdir -p svg
   for file in ./*.sa17; do
       echo Processing  $file...
       froot=$file
   #   Network TCPv4 errors
       sadf -g $network_interface $file -- -n ETCP > svg/$froot.tcpv4errors.svg
   done
   ```

4. Open all  `*.svg` files in a browser, for example

   ![](C:\Dev\Runbook\cockroachdb-runbook-template-main - work\diagnostic-support\res\node-tcpv4-errors.png)

5. Observe `retrans/s` metric. This is the number of TCP/IP v4 re-transmits. In the above example  `retrans/s` metric briefly spikes to the rate of about 20/s around 5am UTC, same time CockroachDB reported liveness timeouts. Since TCP/IP is a reliable protocol, TCP retransmits undelivered packets. CockroachDB may not see a hard network "failure" because eventually retransmits succeed (the disruption only lasted ~10 or so seconds), but during that time CockroachDB saw a "slow" network that tripped nodes' liveness timeouts. While the liveness timeout messages do not necessarily translate into a business application disruption, in that case there were unavailable ranges for a brief period of time since multiple nodes were affected by connectivity issues due to underlying network equipment maintenance. 

   > ✅ If a spike in TCP/IP retransmits rate coincides with [symptoms](#2.-gray-failures-symptoms) in database, it's a conclusive evidence of gray failures in the underlying network.



## 4. Remediation

If you were able to correlate  [symptoms](#2.-gray-failures-symptoms) in database with TCP/IP retransmits, you confirmed the root cause of the issues is in the underlying network. These issues can only be resolved by platform operator, i.e. follow the cloud service provider support protocol.

Cockroach Labs technical support will not be in a position to provide assistance in case of connectivity issues in the cloud platform networking.

In practice, the gray network failures are usually short lived and cleared relatively quickly, causing a minimal or no workload disruption whatsoever. So CockroachDB operators may not need to take any emergency actions, limiting remediation to only a root cause analysis /explanation request filed with a cloud service provider.



## 5. Resources

- [Data Availability: Disruptions Control](https://github.com/cockroachlabs/cockroachdb-runbook-template/blob/main/system-overview/data-availability.md)
- [Node liveness issues](https://www.cockroachlabs.com/docs/stable/cluster-setup-troubleshooting.html#node-liveness-issues)
- [A TCP retransmission primer](https://www.baeldung.com/cs/tcp-retransmission-rules)
