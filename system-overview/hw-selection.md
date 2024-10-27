# Server Hardware Selection

### Overview

This article provides hardware practical provisioning guidance to self-hosted customers who operate CockroachDB on their own hardware ("bare metal") servers. The current guidance is by [compute (CPU and RAM)]() and storage. As an independent software vendor, Cockroach Labs can only supply guidance agnostic of hardware manufacturers.

CockroachDB on "bare metal" is commonly deployed on 1U / 2U industry standard servers (ISS). ISS refers to servers that built using off-the-shelf components and adhere to technical specifications universally adopted by the industry. All established hardware manufacturers have a line of ISS.

The overwhelming majority of CockroachDB production deployments are on rack ISS powered by x86-64 Intel® Xeon® Processors. ISS examples, in no particular order: Dell PowerEdgeR6x0/R7x0, HPE ProLiant DL380 GenXX, Cisco UCS C-Series, Lenovo ThinkSystem STX50.



### Compute (CPU and RAM) Tier Selection Considerations

#### General

>  👉  CockroachDB software is not NUMA aware. The standing Cockroach Labs' guidance is to **align CockroachDB nodes to NUMA nodes** for best performance. Cockroach Labs deployment reference on hardware servers with multiple NUMA nodes relies on either virtualization (all major hypervisors validated – kvm, esxi, hyper-v) or containerization (typically via Kubernetes) to facilitate CockroachDB nodes to NUMA nodes.

> 👉 The recommended memory to core/vCPU ratio is 4 GB/vCPU. For example, the recommended RAM for a 16 core/vCPU server is 64GB.

#### **Server Size (processor cores)**

CockroachDB is written in Go programing language. At present, CockroachDB should not be operated on more than 48 vCPUs/HT cores for a few reasons:

- CockroachDB nodes running on modern processors with 48 or more cores/vCPUs will inevitably straddle at least 2 NUMA nodes. Due to lack of NUMA support in the Go runtime, a CockroachDB node process spanning multiple NUMA zones in hardware may experience, by some estimates, a quadratic performance overhead when accessing RAM.
- A general lack of optimizations in the Go runtime for scheduling concurrency towards 64 vCPUs/HT cores, incurring contention inside the Go scheduler at that level of parallelism.

#### **Number of NUMA nodes**

Cockroach Labs has an extensive customer experience with 2 sockets / 2 NUMA nodes servers. The newer high-end servers have multiple NUMA nodes per socket, so the server presents 4 NUMA nodes. While the Cockroach Labs' general guidance stands for any number of NUMA nodes, i.e. to align CRDB nodes to NUMA nodes, Cockroach Labs has lesser field insights from CRDB on bare metal servers with more than 2 NUMA nodes.

#### **Processor Categories**

Since CockroachDB software is CPU intensive, there is a benefit in selecting the highest productivity compute tier (CPU + RAM). For Intel Xeon based ISS, selecting a higher productivity processor category appreciably increases the compute density (operations per-core and per-server). The best technical fit for  compute intensive workloads are Platinum top-of-the-line processors in the Xeon family. Platinum processors have the highest core count, highest clock speed, support the highest number of memory channels and the fastest memory speeds. For comparison, Xeon Silver processors are considered a good fit for disk IO centric applications, which doesn't match CockroachDB software profile.

This is only a technical consideration that needs a price/performance narrative to support a holistic BOM decision.

Cockroach Labs does not provide model or spec level processor or RAM selection guidance.

#### Per-server productivity estimate

For ROI estimates, it may be useful to assess per-server productivity increase between candidate server BOMs. Per-server productivity is a compound measure of per-core efficiency increase and a larger number of cores per processor.

Operators can quickly assess productivity of their server hardware by reviewing published results of [SPEC® CPU 2017](https://www.spec.org/cpu2017/) - industry-standard benchmark suites that measures and scores general compute-intensive performance. All major hardware manufacturers submit [benchmark results of all their servers](https://www.spec.org/cpu2017/results/cpu2017.html) to the SPEC organization. SPEC provides [search](https://www.spec.org/cgi-bin/osgresults?conf=cpu2017) capabilities over the large volume of submitted results.

The relevant measure to compare productivity of 2 servers is the **base rate** reported on SPEC CPU 20217 Integer Rate (also called “[SPECint](https://www.intel.com/content/dam/www/central-libraries/us/en/documents/2022-07/looking-beyond-specrate-2017-paper.pdf)” for short) Result sheet. The base rate measures overall system performance engaging all available processors. It’s a pure CPU throughput benchmark. Higher base rate result is better.

For example,

| Server                                          | SPECrate2017 Integer Rate Result (Base)                      |
| ----------------------------------------------- | ------------------------------------------------------------ |
| PowerEdge R640 (Intel Xeon Silver 4214 2.20GHz) | [133](https://www.spec.org/cpu2017/results/res2019q2/cpu2017-20190429-12797.pdf) |
| PowerEdge R660 (Intel Xeon Silver 4416+)        | [361](https://www.spec.org/cpu2017/results/res2023q2/cpu2017-20230421-35966.pdf) |
| Increase:                                       | 361 / 133 = 2.7 times                                        |

While it has been shown that performance of compute intensive databases generally tracks with SPECint results, the real world database performance depends on many factors and will not realize the full 2.7 times increase for at least 2 reasons - the SPECint is run on physical cores (with hyperthreading disabled) while CockroachDB is operated on HT cores, and the real-world CockroachDB workload, although compute heavy, includes network and disk IO. Therefore, for the first order of approximation deduct 30-40%. In the above example, an R640 to R660 upgrade which yields an estimated per-server productivity increase of about x2 times.

> 👉 The material in this section is not an alternative to an actual CockroachDB cluster performance validation of a candidate bill of materials (BOM) configuration. Every major manufacturer of server hardware operates a network[ ](https://i.dell.com/sites/csdocuments/Shared-Content_data-Sheets_Documents/en/dell-solution-center-customer-brochure.pdf)[Customer Solution Centers](https://i.dell.com/sites/csdocuments/Shared-Content_data-Sheets_Documents/en/dell-solution-center-customer-brochure.pdf) to speed up customers’ decision making with hands-on validation of a candidate server BOM.



