# Clock Management

### Overview

This article highlights the importance of [good timekeeping](#importance-of-good-timekeeping) in CockroachDB operations and offers [clock configuration guidance](#clock-configuration-guidance) for [homogeneous](#ntp-sources) and [multi-cloud](#multi-cloud-environments) environments. 

### Importance of Good Timekeeping

CockroachDB’s persistence model is a multi-versioned append-only distributed log that rolls forward with time. This [blog](https://www.cockroachlabs.com/blog/living-without-atomic-clocks/) provides good insights into the role of clocks in CockroachDB operations and why a tight synchronization of clocks of all CockroachDB VMs in the CockroachDB cluster is extremely important.

#### Impact of Clock Synchronization Precision

Clock skew between CockroachDB cluster VMs can impact SQL workload response times. Generally, larger skews degrade query performance.

CockroachDB relies on the clocks of all of a CockroachDB cluster’s VMs to stay within a configurable maximum offset from each other. This is necessary to provide ACID guarantees, specifically non-stale reads, i.e. reads that do not return the latest committed value.

In CockroachDB, every transaction starts and commits at a *timestamp* assigned by some CockroachDB node. When choosing this timestamp the node does not rely on any particular synchronization with any other CockroachDB node. This becomes the timestamp of each tuple written by the transaction. CockroachDB uses multi-version concurrency control (MVCC) - it stores multiple values for each tuple ordered by timestamp. A reader generally returns the latest tuple with a timestamp earlier than the reader's timestamp and ignores tuples with higher timestamps. This is when *clock skew* needs to be considered. If a reader's timestamp is assigned by a CockroachDB node with a clock that is behind, it might encounter tuples with higher timestamps that are, nevertheless, in the reader's “past", i.e. they were committed before the reader's transaction but their timestamps were assigned by a clock that is ahead. This problem is solved with the *max-offset*. Each reading transaction has an *uncertainty interval*, which is the time between the reader’s timestamp and the reader’s timestamp plus the max-offset. Tuples with timestamps above the reader’s but within the max-offset are considered to be ambiguous as to whether they're in the past or the future of the reader. Encountering such an uncertain value can lead to the reader transaction having to restart at a higher timestamp so this time all the previously uncertain values are definitely in the past. There are various optimizations for limiting this uncertainty, yet generally clock skew can lead to poor performance because it requires transactions to restart.

#### Maximum Offset Value and its Impact on Performance

The default maximum clock offset value is 500 ms. CockroachDB enforces the maximum offset with a specially designed clock skew detection mechanism built into the intra node communication protocol. CockroachDB nodes periodically exchange clock signals and computing offsets. If a CockroachDB node detects a drift of over 80% of the maximum offset (e.g. 400 ms, assuming the default maximum offset value) vs. half of other nodes, it spontaneously shuts down the node to guarantee database read consistency.

In principle, an increase the maximum offset may have an adverse effect on user workload performance as it increases the probability of uncertainty retries. In most cases, retries are handled [automatically](https://www.cockroachlabs.com/docs/stable/transactions#automatic-retries) on the server side, yet in some cases the application may receive [retry errors](https://www.cockroachlabs.com/docs/stable/transaction-retry-error-reference.html#error-reference) and the transaction must be  retried  on the [client side](https://www.cockroachlabs.com/docs/stable/transactions#client-side-intervention). A decrease generally has a positive impact.

>  👉  If the current Cockroach Labs clock management guidance is followed, it is generally safe to reduce the maximum offset to 250 ms from its default value. This default can be explicitly overwritten with the *--max-offset* flag in the node start command.

It is impossible to quantify or predict the exact performance impact as it depends on the specific SQL workload and user driven concurrency. The impact may range from non-measurable to dramatic.

The worst-case scenario is when clocks are in fact well synchronized and the workload is concurrent with long-running transactions. This occurs when transactions are multi-statement, have long running selects with large scans and produce large result sets.

When a read from one of the long-running transactions encounters a write with a higher timestamp but within the max offset, i.e. the *uncertainty window*, it will generally be forced to restart.

With a larger uncertainty window the probability of contention is higher. If retries are observed, for example, with a 250 ms maximum offset, a user should expect more retries with a setting of 500 ms.

As noted earlier, the size of the result set impacts the probability of retries. If an application client implements single statement transactions with a small result set, the server will auto-retry and the client will experience a relatively limited increase in the response time. Yet transactions with large result sets may “spill” the result set records to the client which would inevitably increase the probability of a retry and the cost of each retry as that retry error will be propagated to the client. 

#### Time Smoothing for Leap Second Handling

Reader's familiarity with [leap second clock adjustments](https://en.wikipedia.org/wiki/Leap_second) is implied. In this article, the term clock "jump" is synonymous to the NTP technical term "step".

Timekeeping practiced by different government and business entities can take one of the two forms when it comes to a clock "jump" during a leap second adjustment:

1. Monotonic time. Instead of stepping back during a leap second, time will be slowed leading up to the leap second in such a way that time will neither jump nor move backwards. Disadvantage: The time leading up to / following the leap second will not accurately reflect the real time.

2. Accurate time. This can mean a large 1 second clock jump, for example for leap second adjustment, and possible going back in time, if the local clock is ahead of reference time.

*In both cases the data consistency/correctness will be preserved by CockroachDB's internal synthetic HLC clock.* However the operators need to be concerned about 2 often undesirable realities associated with accurate timekeeping (option 2):

- If a clock jump results in a large clock offset between nodes, CockroachDB nodes may spontaneously exit to protect ACID guarantees, causing unnecessary commotion in the cluster.
- Customer's applications may exhibit unexpected behavior if they read the local Linux clock or use SQL time functions like `now()`.

> 👉 Cockroach Labs has a strong preference for option 1 - monotonic time. The priority is for time to be smooth and coordinated across nodes rather than allowing a synchronized jump with UTC.

Time smoothing of the leap second can be implemented using NTP's *slew* or *[smear](https://googleblog.blogspot.com/2011/09/time-technology-and-leaping-seconds.html)*. Either is acceptable to CockroachDB while a large *step* is undesirable. Continue to [NTP Sources](#ntp-sources) for platform specific configuration guidance.

#### Time Continuity during Live CockroachDB VM Migrations (Memory Preserving Maintenance)

[Live VM migrations](https://en.wikipedia.org/wiki/Live_migration) are supported by all hypervisors that can be used to run CockroachDB clusters:

- [KVM](https://en.wikipedia.org/wiki/Kernel-based_Virtual_Machine) (AWS and GCP public clouds)
- [Hyper-V](https://learn.microsoft.com/en-us/windows-server/virtualization/hyper-v/manage/live-migration-overview) (Azure public cloud)
- [ESXi](https://www.vmware.com/products/cloud-infrastructure/vsphere/vmotion) (VMware vSphere private cloud)

Live VM migrations is an infrastructure level method of enabling platform's operational continuity. It moves a VM running on one physical hardware server to another server without a VM reboot.

##### Uninterrupted Clock Pre-requisite Requirement

The key technical requirement for allowing CockroachDB VMs to migrate live without a service disruption is *guest VM clock continuity*:

> ✔️ Problem: Guest clock continuity can't be guaranteed during a live VM migration.
>
> Migration starts with an iterative pre-copy of the VM's memory. Then VM is suspended, the last dirty pages a copied, and the VM is resumed at destination. The downtime can range from a few milliseconds to seconds. 
>
> While a VM is suspended, its clock does not run. The moment a CockroachDB VM is resumed at its destination, its clock is behind by the amount of suspension time, i.e. it's arbitrarily stale. Therefore a SELECT statement starting immediately after a migration can be assigned a stale timestamp. In other words, the database cannot guarantee consistency (“C” in ACID) during a live migration.
>
> ✅ Solution: If live CockroachDB VM migration is required, all CockroachDB nodes must be configured to use the clock of the  hardware host via a precision clock (PTP) device.  The hardware clock is continuously available through the entire live migration cycle. CockroachDB nodes can be configured to use the hardware clock with the `--clock-device` flag in the [node start](https://www.cockroachlabs.com/docs/stable/cockroach-start.html#flags) command.
>
> Access to the hardware clock via PTP device is currently supported by [VMware vSphere](https://techdocs.broadcom.com/us/en/vmware-cis/vsphere/tools/12-5-0/add-a-precision-clock-device-to-a-virtual-machine.html), [AWS EC2](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/configure-ec2-ntp.html), and [Azure](https://learn.microsoft.com/en-us/azure/virtual-machines/linux/time-sync). However, neither AWS EC2 nor Azure hardware clock meet [time smoothing](#time-smoothing-for-leap-second-handling) requirement and consequently may not be used with CockroachDB.

Live VM migration can support 2 operational tasks - (1) runtime *VM load optimization* (e.g. VMware DRS) and (2) *VMs' memory-preserving maintenance* of the underlying hardware servers.

Cockroach Labs advises *against* enabling runtime VM load optimization whenever CockroachDB operator has control over this configuration option. Best practices [guidance is available](#configuring-cockroachdb-for-planned-cloud-maintenance) for memory-preserving maintenance of CockroachDB VMs in cloud platforms meeting pre-requisite requirements.

### Clock Configuration Guidance

#### NTP Client Side Configuration

The following Linux configuration guidance applies to CockroachDB guest VMs and hardware servers running CockroachDB software.

> ✅ TLDR;  Cockroach Labs recommends `chrony` as the NTP client on CockroachDB VMs/servers.

There are three commonly used NTP clients available on Linux - *timesyncd*, *ntpd*, and *chronyd*.

- `Chrony` has many advantages over other NTP clients and is the only NTP client recommended by Cockroach Labs. `Chrony` is the [default NTP user space daemon in Red Hat 7 and onwards](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/system_administrators_guide/ch-configuring_ntp_using_ntpd). `Chrony` is the [recommended optional client](https://documentation.ubuntu.com/server/how-to/networking/serve-ntp-with-chrony/#serve-ntp-with-chrony) by Ubuntu. CockroachDB platform health checks conducted by Cockroach Labs will flag non-crony NTP clients.
- `Ntpd` is now deprecated and should not be used in new CockroachDB installations. Starting with Red Hat 8 [ntpd is no longer available](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/considerations_in_adopting_rhel_8/infrastructure-services_considerations-in-adopting-rhel-8). By default, `ntpd` is no longer installed in Ubuntu. `Ntpd` may, however, persist through Linux upgrades. Cockroach Labs encourages customers to plan to replace legacy `ntpd` clients with `chrony` to maintain a good platform configuration hygiene.
- `Timesyncd` is simplistic and cannot reliably ensure the required precision in multi-regional topologies. `Timesyncd` is the default NTP client in Ubuntu. Ubuntu's [posture](https://documentation.ubuntu.com/server/explanation/networking/about-time-synchronisation/#about-timedatectl) is "`timesyncd` will generally keep your time in sync, and `chrony` will help with more complex cases." 

> ✅ Cockroach Labs defers the NTP client configuration decision to the underlying planform vendors when CockroachDB is operated over a managed Kubernetes service, such as AWS EKS, GCP GKE, Azure AKS.

####  NTP Sources

A NTP server is a computer system that acts as a reliable source of time to synchronize NTP clients. In this article, the term clock "NTP server" is synonymous to "NTP source".

Configure the NTP clients on CockroachDB VMs to synchronize against NTP sources that meet the following best practices:

- Choose geographically local source with the minimum synchronization distance (network roundtrip delay). This will improve resiliency by reducing the probability of network disruption to a source. For muti-region and multi-cloud topologies it means VMs in each region should synchronize against regional NTP sources, as long as regional sources comply with Cockroach Labs best practices guidance.
- Only use NTP sources that implement [smearing or slewing of leap second](#time-smoothing-for-leap-second-handling).
- If memory preserving maintenance is required, configure NTP sources with uninterrupted time.
- Configure NTP sources with redundancy. At least 3 NTP servers is a common IT practice for configuring each NTP client.
- NTP configurations of all CockroachDB VMs in the same region must be identical.

##### AWS EC2

> ✅ Configure all in-region EC2 CockroachDB VMs to access the [local Amazon Time Sync Service](https://aws.amazon.com/blogs/aws/keeping-time-with-amazon-time-sync-service) via `169.254.169.123` IP address endpoint. This is usually the default EC2 Linux VM configuration. For a backup, or to synchronize CockroachDB VMs outside of AWS EC2, use  the [public Amazon Time Sync Service](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/configure-time-sync.html) via `time.aws.com`. The local and public AWS sources automatically [smear UTC leap seconds](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/set-time.html#leap-seconds). [AWS leap smear](https://aws.amazon.com/about-aws/whats-new/2022/11/amazon-time-sync-internet-public-ntp-service/) is algorithmically identical to [GCP leap smear](https://developers.google.com/time/smear#standardsmear).
>
> 👉 Avoid using the [EC2 PTP hardware clock](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/configure-ec2-ntp.html) because it does not smear time, i.e. does not meet [time smoothing](#time-smoothing-for-leap-second-handling) requirement.

##### GCE

> ✅ Configure all in-region GCE CockroachDB VMs to access the [internal GCE NTP Service](https://cloud.google.com/compute/docs/instances/configure-ntp#configure_ntp_for_your_instances) via `metadata.google.internal` endpoint. This is usually the default GCE Linux VM configuration. For a backup, or to synchronize CockroachDB VMs outside of GCE, use [Google public NTP service](https://developers.google.com/time/faq) via `time.google.com`. The local and public GCE sources automatically [smear UTC leap seconds](https://googleblog.blogspot.com/2011/09/time-technology-and-leaping-seconds.html).  [GCP leap smear](https://developers.google.com/time/smear#standardsmear) is algorithmically identical to [AWS leap smear](https://aws.amazon.com/about-aws/whats-new/2022/11/amazon-time-sync-internet-public-ntp-service/).

##### Azure

> ✅ Azure infrastructure is [backed by Windows Server](https://learn.microsoft.com/en-us/azure/virtual-machines/linux/time-sync) that provide guest VMs with [accurate time](https://learn.microsoft.com/en-us/troubleshoot/windows-server/active-directory/time-service-treats-leap-second). I.e. the native Azure-provided Time sync service for Linux VMs does *not* [smear or slew leap second](#time-smoothing-for-leap-second-handling). Configure all in-region Azure CockroachDB VMs to synchronize either to [Google public NTP service](https://developers.google.com/time/faq) via `time.google.com` or to [public Amazon Time Sync Service](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/configure-time-sync.html) via `time.aws.com`. Either service is acceptable as primary. Configure the other one as a backup. All CockroachDB VMs in the same region must have an identical NTP configuration.

##### VMware VSphere

> ✅ The [CockroachDB on VMware vSphere](https://www.cockroachlabs.com/guides/cockroachdb-on-vmware-vsphere) white paper provides clock configuration best practices for guest VMs and ESXi servers. 



#### Multi Cloud Environments

| Multi-Cloud CockroachDB Cluster                           | Recommended NTP Sources                                      |
| --------------------------------------------------------- | ------------------------------------------------------------ |
| EC2 + GCE                                                 | VMs in EC2-backed regions: follow [AWS EC2](#aws-ec2) earlier in this section.<br />VMs in GCE-backed regions: follow [GCE](#gce) earlier in this section.<br />Operators can "mix and match" AWS and GCE NTP sources because they are compatible. |
| EC2 + Azure<br />EC2 + vSphere<br />EC2 + Azure + vSphere | VMs in EC2-backed regions: follow [AWS EC2](#aws-ec2) earlier in this section.<br />VMs in Azure-backed regions: use [public Amazon Time Sync Service](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/configure-time-sync.html).<br />VMs in vSphere-backed regions: use [public Amazon Time Sync Service](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/configure-time-sync.html). Alternatively use internal proprietary sources if they are guaranteed to implement standard AWS/Google smearing. |
| GCE + Azure<br />GCE + vSphere<br />GCE + Azure + vSphere | VMs in GCE-backed regions: follow [GCE](#gce) earlier in this section.<br />VMs in Azure-backed regions: use [Google public NTP service](https://developers.google.com/time/faq).<br />VMs in vSphere-backed regions: use [Google public NTP service](https://developers.google.com/time/faq). Alternatively use internal proprietary NTP sources if they are guaranteed to implement standard Google/AWS smearing. |



### Configuring CockroachDB for Planned Cloud Maintenance

A *maintenance event* refers to a planned operational activity that requires guest VMs to be moved out of the host server.

All public and private cloud service have a required planned maintenance. CockroachDB operators shall develop an operational procedure to handle planned maintenance events. A custom procedure is required for each cloud platform in case of a multi-cloud deployment.

> 👉 Relying on default / out-of-the-box platform behavior during a maintenance event is generally unacceptable as it's likely to be disruptive to CockroachDB cluster VMs.  A custom procedure or configuration adjustments are usually required for public and private cloud platforms.
>
> Q: What are the potential ramifications of allowing live migrations of CockroachDB VMs without configuring an uninterrupted clock?
>
> A: Without a clock continuity guarantee, the database will be denied the means to guarantee transactional consistency and the applications may be served a stale read without any indication of that happening. Since a probability of a compromised consistency exists, however small, it is not acceptable for applications using CockroachDB as a system of records.

Here is a summary of recommendations for handling maintenance events, by cloud platform:

| Cloud Platform | Live Migration / Memory Preserving Maintenance Mode          | Uninterrupted Clock Support via PTP                          | Outline for Handling Maintenance Events                      |
| -------------- | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| **AWS**        | Not available                                                | [Supported](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/configure-ec2-ntp.html). However, the EC2 hardware clock does not smear time, i.e. does not meet [time smoothing](#time-smoothing-for-leap-second-handling) requirement. | Memory preserving maintenance is not available. Follow [special provisions for AWS EC2](#aws-ec2-special-provisions). |
| **GCP**        | [Supported](https://cloud.google.com/compute/docs/instances/live-migration-process) | Not available                                                | Uninterrupted clock is not available in GCE, therefore CockroachDB VMs *can not* be allowed to live migrate. Follow [special provisions for GCE](#gce-special-provisions). |
| **Azure**      | [Supported](https://learn.microsoft.com/en-us/azure/virtual-machines/maintenance-and-updates#maintenance-that-doesnt-require-a-reboot) | [Supported](https://learn.microsoft.com/en-us/azure/virtual-machines/linux/time-sync). However, the Azure hardware clock does not meet [time smoothing](#time-smoothing-for-leap-second-handling) requirement. | Azure-provided uninterrupted clock does not smooth the leap second. Therefore CockroachDB VMs *can not* be allowed to live migrate, at least *near a leap second adjustment events*. Follow [special provisions for Azure](#azure-special-provisions). |
| **VMware**     | [Supported (vMotion)](https://www.vmware.com/products/cloud-infrastructure/vsphere/vmotion) | [Supported](https://techdocs.broadcom.com/us/en/vmware-cis/vsphere/tools/12-5-0/add-a-precision-clock-device-to-a-virtual-machine.html). vSphere ESXi servers shall use time synchronization sources configured with [time smoothing](#time-smoothing-for-leap-second-handling) for leap second handling. | Live Migration (vMotion) is supported. Follow instructions in [CockroachDB on VMware vSphere](https://www.cockroachlabs.com/guides/cockroachdb-on-vmware-vsphere) white paper. |



##### AWS EC2 Special Provisions

> ✅ Perform a proactive rolling (one-at-a-time) CockroachDB nodes restart, [stopping-and-starting or rebooting](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/schedevents_actions_retire.html) each CockroachDB VM scheduled for maintenance, depending on the root volume being EBS or instance store. This procedure must be executed during the scheduled maintenance event window, upon receiving a [scheduled maintenance event](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/viewing_scheduled_events.html) notification. Operators can create [custom maintenance event windows](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/event-windows.html) to avoid maintenance during peak database use periods.  A stop-and-start or reboot of a VM scheduled for maintenance will relocate it to a new underlying hardware host.

##### GCE Special Provisions

> 👉 By default, the GCE VM types that run CockroachDB nodes are set to *live* migrate during maintenance. However, uninterrupted clock is not available to CockroachDB VMs in GCE. Therefore operators *must* [disable live migrations](https://cloud.google.com/compute/docs/instances/setting-vm-host-options#available_host_maintenance_properties) for all CockroachDB VMs, i.e. explicitly set the host maintenance policy from `MIGRATGE` to `TERMINATE`. Live migration needs to be disabled individually for each created CockroachDB VM because there is no way to disable it for all VMs created from a template.
>
> ✅ CockroachDB operators are encouraged to [monitor the upcoming maintenance notifications](https://cloud.google.com/compute/docs/instances/monitor-plan-host-maintenance-event#overview_of_maintenance_notifications) and [manually trigger](https://cloud.google.com/compute/docs/instances/trigger-host-maintenance-event#trigger-host-maintenance-event) CockroachDB VM maintenance after [orderly stopping CockroachDB node](../routine-maintenance/node-stop.md) and ensuring only one node is down for maintenance at a time.

##### Azure Special Provisions

> ✅ TODO

##### VMware VSphere Special Provisions

> ✅ Follow detailed instructions in the [CockroachDB on VMware vSphere](https://www.cockroachlabs.com/guides/cockroachdb-on-vmware-vsphere) white paper.

