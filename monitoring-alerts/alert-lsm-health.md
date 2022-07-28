# Alert: Node LSM Storage Health

### Purpose of this Alert

CockroachDB is using a LSM-Tree Pebble storage engine (a custom RocksDB re-write in Go). The health of an LSM tree can be measured by the *read amplification*, which is the average number of SSTables being checked per read operation.

A node reporting a high read amplification is an indication of a problem on that node that is likely to affect the workload.

A `rocksdb.read-amplification` in the single digits is characteristic of a healthy LSM tree.

A `rocksdb.read-amplification` in the double/triple/quadruple digits suggests an inverted LSM.

Possible root causes of LSM inversion and its harmful effects are outlined in  [the common problems experienced by CockroachDB users](../most-common-problems/README.md) section.



------

### Monitoring Metric
```
rocksdb.read-amplification
```



### Alert Rule

| Tier     | Definition                                    |
| -------- | --------------------------------------------- |
| WARNING  | Read Amplification exceeds 50 for 1 hour      |
| CRITICAL | Read Amplification exceeds 150 for 10 minutes |



### Alert Response

The actual response varies depending on the alert tier, i.e. the severity of potential consequences.

- Check the history of read amplification via DB Console

- Check the periodic `Compaction Stats [default]` messages in the CockroachDB logs for confirmation of the degree of the Read Amplification

- Compaction may be "starved of CPU" if a high Read Amplification coincides with a high CPU utilization.  Address the high CPU utilization by reducing the workload concurrency. Compaction should "catch up" and the Read Amplification should be be gradually reduced to normal

In a severe case, a manual intervention may be required. The options are:

  1. [Run the offline manual compaction on the problem node](../emergency-procedures/lsm-compact.md)
  
  1. [Replace the problem node](../emergency-procedures/node-replace.md)
  
  1. [Wipe the problem node](../emergency-procedures/node-wipe.md)
