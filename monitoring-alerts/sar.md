# Troubleshooting Preparedness: System Activity Report Configuration

### About SAR

The System Activity Report (`SAR`) is a Linux monitor that can collect detailed system performance metrics, enabling reporting and analysis of nearly all aspects of system utilization. It is  a part of the `sysstat` package.



### Configure SAR on All CockroachDB VMs / Servers!

CockroachDB operators are strongly advised to install and ***continuously run `SAR` on all CockroachDB node servers or VMs***. It enables operators to identify potential issues early, fine-tune performance, and correlate cluster-level to the underlying platform performance metrics - a critical capability, often required to determine a root cause of an incident.

> ✅ The recommended configuration below will not add a measurable system overhead.



##### 1. Install SAR

Install `sysstat` package, if not already installed, on all Linux servers or VMs that run CockroachDB nodes. The `systat` package contains utilities and tools that can be scheduled with `cron` to monitor and historize system performance and usage activity data. `sysstat` is available on all commercial Linux platform supported by CockroachDB.

##### 2. Modify Default SAR Schedule

Upon installation, the `sysstat` package configures SAR metrics collection at 10 minute resolution. This is too coarse, not capturing meaningful insights into the system activity and resource utilization.

Change the `cron` schedule to run SAR job at 1 minute interval (the lowest available):

- Edit  `/etc/cron.d/sysstat`
- Enter 1 minute interval 
  `*/1 * * * * root /usr/lib64/sa/sa1 1 1`

- Save the file. There is no need to restart `cron ` after modifying a crontable because `cron` automatically checks its crontables' mod times every minute and reloads the modified ones.

##### 3. Expand Default SAR Metrics Collection

Increase SAR capture to include statistics about IPv4 network errors (retransmits, bad TCP checksums, etc.).  These metrics are necessary to instrument network failures, including more subtle gray failures. These metrics are collected with  `sadc`'s option `-S SNMP`  and are accessible with  `ETCP`  network statistics report.

- Stop SAR service
`systemctl stop sysstat`

- Edit the SAR configuration file (`/etc/sysconfig/sysstat` in RHEL,  `/etc/sysstat/sysstat` in Ubuntu) and enable additional network statistics reporting with the following line 
  `SADC_OPTIONS="-S DISK -S SNMP"`
  Save the file.

- Start SAR service
`systemctl start sysstat`
