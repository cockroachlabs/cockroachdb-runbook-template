# Clock Management

### Overview



This article highlights the importance of [good timekeeping](#importance-of-good-timekeeping) in CockroachDB operations and offers clock configuration guidance for homogeneous and multi-cloud environments. 

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

2. Accurate time. This can mean a large 1 second clock step and possible going back in time, if the local clock is ahead of reference time.

*In both cases the data consistency/correctness will be preserved by CockroachDB's internal synthetic HLC clock.* However the operators need to be concerned about 2 often undesirable realities associated with accurate timekeeping (option 2). CockroachDB nodes may spontaneously exit to protect ACID guarantees, causing a commotion in the cluster, if clock steps result in a large clock offset between nodes. And customer's applications may exhibit unexpected behavior if they read the local Linux clock or use SQL time functions like `now()`.

> 👉 Cockroach Labs has a strong preference for option 1 - monotonic time. The priority is for time to be smooth and coordinated across nodes rather than allowing a synchronized "jump" with UTC.

Time smoothing of the leap second can be implemented using NTP's *slew* or *smear*. Either is acceptable to CockroachDB while a large *step* is undesirable. Continue to [NTP Sources](#ntp-sources) for platform specific configuration guidance.

#### Time Continuity for Live CockroachDB VM Migrations (Memory Preserving Maintenance)

[Live VM migrations](https://en.wikipedia.org/wiki/Live_migration) are supported by all hypervisors that can be used to run CockroachDB clusters:

- [KVM](https://en.wikipedia.org/wiki/Kernel-based_Virtual_Machine) (AWS and GCP public clouds)
- [Hyper-V](https://learn.microsoft.com/en-us/windows-server/virtualization/hyper-v/manage/live-migration-overview) (Azure public cloud)
- [ESXi](https://www.vmware.com/products/cloud-infrastructure/vsphere/vmotion) (VMware vSphere private cloud)

Live VM migrations is an infrastructure level method of enabling platform's operational continuity. It moves a VM running on one physical hardware server to another server without disrupting normal operations.

Live VM migration can support 2 operational tasks - (1) runtime *VM load balancing* (e.g. VMware DRS) and (2) *VMs' memory-preserving maintenance* of the underlying hardware servers running the hypervisor software.

> 👉 Cockroach Labs advises *against* enabling runtime VM load balancing whenever CockroachDB operator has control over this configuration option. In particular, Cockroach Labs advises to keep the Migration Threshold in VMware DRS at a conservative setting for VM migrations during vSphere maintenance only.

Therefore, the focus of this section is narrowed to the best practices for memory-preserving maintenance of CockroachDB VMs.



### Clock Configuration Guidance

[Summary table for multi-cloud goes here]

How NTP operates:  https://doc.ntp.org/documentation/4.1.0/ntpd/

#### NTP Client Side Configuration

The following Linux configuration guidance applies to CockroachDB guest VMs and non-containerized hardware servers running CockroachDB software.

There are three commonly used NTP clients available on Linux - *timesyncd*, *ntpd*, and *chronyd*.

*Chrony* has many advantages over other NTP clients and is the only NTP client recommended by Cockroach Labs. `Chrony` is the [default NTP user space daemon in Red Hat 7 and onwards](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/system_administrators_guide/ch-configuring_ntp_using_ntpd). `Chrony` is the [recommended optional client](https://documentation.ubuntu.com/server/how-to/networking/serve-ntp-with-chrony/#serve-ntp-with-chrony) by Ubuntu. CockroachDB platform health checks conducted by Cockroach Labs will flag non-crony NTP clients.

`Ntpd` is now deprecated and should not be used in new CockroachDB installations. Starting with Red Hat 8 [ntpd is no longer available](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/considerations_in_adopting_rhel_8/infrastructure-services_considerations-in-adopting-rhel-8). By default, `ntpd` is no longer installed in Ubuntu. `Ntpd` may, however, persist through Linux upgrades. Cockroach Labs encourages customers to plan to replace legacy `ntpd` clients with `chrony` to maintain a good platform configuration hygiene.

Timesyncd is simplistic and cannot reliably ensure the required precision in multi-regional topologies. Timesyncd is the default NTP client in Ubuntu. Ubuntu's [posture](https://documentation.ubuntu.com/server/explanation/networking/about-time-synchronisation/#about-timedatectl) is "`timesyncd` will generally keep your time in sync, and `chrony` will help with more complex cases." 

Cockroach Labs defers the NTP client configuration decision to the underlying planform vendors when CockroachDB is operated in private cloud or containerized environments. This includes VMWare vSphere, AWS EKS, GCP GKE, etc.

####  NTP Sources

CRL always referred customers to redhat's docs on this question. the "slew" and "smear" options from this doc are acceptable for CRDB, while the "step" options are not. https://access.redhat.com/articles/15145
Red Hat has put together a comprehensive guide on the 2016 leap second issue to prepare and prevent any problems or downtime while using Red Hat Enterprise Linux.

If possible, using the algorithm that google calls the "standard smear" (also used by amazon) is preferred. https://developers.google.com/time/smear

NTP ONLY slews the time if the time offset does NOT exceed a setting called "step threshold" (128ms by default), otherwise it just steps it. That is a practical setting for a quick sync if clocks are way apart. 128ms is evidently not a good setting for us, yet I'd like to understand why they are thinking 2s.


Adjusting Small Time Offsets

As long as the system time offset determined by the filter algorithm is below a certain limit (the so-called step threshold, 128 ms by default), the system time is adjusted slowly and smoothly in a way that both the time offset, and the system clock drift become as small as possible, so that a new significant time offset doesn't even accumulate. See also

NTP docs: Step and Stepout Thresholds
http://doc.ntp.org/current-stable/clock.html#step
So during normal operation, the system time is adjusted in a way that applications don't even notice the small corrections. However, how quickly and accurately a time offset is determined and compensated depends on the jitter seen by the filter algorithms from the last polling actions.

The rate for the system time adjustment is limited to 500 ppm, i.e. 500 microseconds per second, or 1.8 seconds per hour, which is usually sufficient for real applications.


Handling Large Time Offsets

If a large time offset is observed which exceeds the step threshold then the system time is about to be stepped to correct this.

Normally this should only happen once, immediately after ntpd has started, if the system time is not yet very accurate, so only in this case the system time is stepped quickly to get the time offset below the step threshold limit and continue with the smooth adjustment.

However, if the system time has already been accurately disciplined, but afterwards a system time offset is detected that exceeds the step threshold, ntpd waits for the so-called stepout interval (300 s by default since ntpd 4.2.8, 900 s by default until ntpd 4.2.6) to see if the large time offset persists, and then checks if the time offset is above or below the so-called panic threshold (1000 s by default). See:

NTP docs: Panic Threshold
http://doc.ntp.org/current-stable/clock.html#panic
If a large time offset occurs while 'ntpd' is already running, this can be due to one of the following reasons:

An operator has changed the system time. This requires admin rights, so it's the operator's own problem if thinks he has to mess up the timekeeping.
Another time synchronization software is running. It's never a good idea to have more than one program running in parallel to discipline the system time, so all programs but a single one should be disabled.
The system timekeeping is broken. There have been cases where the time on a Windows server lost more than 30 seconds whenever a huge database application ran some maintenance tasks in the night. A program like ntpd is unable to compensate this, so the bad programs should be fixed instead.
So if the system time offset still exceeds the panic threshold after the stepout interval, ntpd terminates itself with a message saying something like, “set clock manually”. The reason behind this behavior is that ntpd means, “The system time has been changed. That must have been done by the administrator who should know what he's doing. So I can't do anything else and terminate myself.”.

So a huge time offset that exceeds the panic threshold is accepted only once at startup, if ntpd is started with the '-g' option (which is usually the case).

Smearing

A completely different approach to correct clocks for leap second on multiple systems in a controlled manner is to suppress the information about upcoming leap second on the NTP server and slowly slew the served NTP time instead. The clients don't know a leap second will be inserted, they don't make any corrections for it themselves and simply follow the server time which eventually brings them back to UTC. This idea was described by Google in 2011 as a [leap smear](http://googleblog.blogspot.com/2011/09/time-technology-and-leaping-seconds.html).

On the client side no special configuration is needed. But care must be taken to ensure the clients use for synchronization only NTP servers that smear leap second in the exactly same way and not normal servers which announce leap second and also make the backward step when leap second is inserted. 


