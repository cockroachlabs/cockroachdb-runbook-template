# Procedure: Changefeed Management
Change data capture (CDC) provides efficient, distributed, row-level changefeeds into a configurable sink for downstream processing such as reporting, caching, or full-text indexing.

## Purpose
This document's purpose is to outline changefeed management tasks during routine changes to the cluster and database. Changefeed sinks and options are well documented [cockroachlabs.com](https://www.cockroachlabs.com/docs/stable/change-data-capture-overview.html) and will not be covered here.

## CockroachDB version used for this documentation
`v21.2.5`

## Changefeed during cluster maintenance 
Changefeeds work as jobs in CockroachDB. Jobs in CockroachDB is assigned to a single node in the cluster called the coordinator node. When the coordinator node is impacted by an upgrade or resizing activities, another node will take over the task. Keeping changefeeds running during cluster maintenance will incur additional overhead and risk so it is advisable to pause and resume all changefeeds using the following:

```sql
PAUSE JOBS (SELECT * FROM [SHOW CHANGEFEED JOBS] WHERE status = ('running'));
```

```sql 
RESUME JOBS (SELECT * FROM [SHOW CHANGEFEED JOBS] WHERE status = ('paused'));
```

If changefeeds must be running at all times even during cluster maintenance, use prometheus alerts to ensure that changefeeds do not fall behind or stop altogether.

- [Changefeed is falling behind](../monitoring-alerts/changefeed-is-falling-behind.md)
- [Changefeed is stopped](../monitoring-alerts/changefeed-is-stopped.md)
- [Changefeed restarted frequently](../monitoring-alerts/changefeed-restarted-frequently.md)
