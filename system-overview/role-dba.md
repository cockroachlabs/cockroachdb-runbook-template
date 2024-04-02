# Role: Database Administrator (via system privileges)

### Overview

Organizations may set forth IT security practices that don't allow unrestricted administrative access to databases using an [*admin* role](https://www.cockroachlabs.com/docs/stable/security-reference/authorization#admin-role). In that case, routine Database Administration (DBA) functions need to be delegated to a custom designed [*public* role](https://www.cockroachlabs.com/docs/stable/security-reference/authorization#public-role), with system- or object- level privileges granted to authorize DBA actions.

This article provides a list of the minimally required privileges, grouped by common DBA tasks. An operator can lean on guidance in this article to design the custom authorization model per Organization's requirements and regulations.



### Minimally Required CockroachDB Release

> ✅ The guidance in this article applies to **CockroachDB release 23.1 or later**. It has been tested with v23.1.17. 
>
> The [privilege-](https://www.cockroachlabs.com/docs/stable/security-reference/authorization#privileges) based authentication model was introduced in CockroachDB v22.2 and has been continually improving. The granularity and completeness of the system of privileges emancipated to a production deployment grade authorization model in v23.1. 



> ✅ There are no plans to back port privileges introduced in v23.1 to v22.2.
>
> v23.1 privileges not available in v22.2 are marked in the instructions below for operator's awareness.



### DBA Group and Individual Roles

A `DBA` group (role) may be created to centralize management of privileges granted to all DBAs. With interactive users (roles) inheriting authorizations via the DBA group membership.

> ✅ This article is dedicated to DBA authorization; all authentication provisions are omitted to eliminate clutter.



The following must be executed as an admin user (role):

```sql
------------------------------------
-- Connect as root (admin) persona !
------------------------------------

-- Create a "database administrator" group (role),
-- to serve as the central place for DBA authorizations.
-- This group (role) is not meant for interactive use, thus NOLOGIN. 
CREATE ROLE dba WITH NOLOGIN;

-- Create individual roles (users) for acting staff DBAs, with interactive LOGIN
CREATE USER  dba_staff_minnie WITH LOGIN;
GRANT dba TO dba_staff_minnie;

CREATE USER  dba_staff_mickey WITH LOGIN;
GRANT dba TO dba_staff_mickey;

```

To remove DBA authorizations from a user (e.g. `dba_staff_mickey`):

```sql

REVOKE dba FROM dba_staff_mickey;

```



### Authorization of Administrative Actions

-------------

The required privilege grant are grouped by common DBA tasks. The same privilege may appear in different task groups. A privilege can be granted repeatedly. 
Customize as it suits Organization's IT practices.

#### Authorize DBA to View System Tables and Current Settings

```sql
-- Authorize DBAs with grants of system (cluster) level privileges to the DBA group.

GRANT SYSTEM  VIEWSYSTEMTABLE           TO dba;   -- e.g. select * from system.settings;                      NOT IN 22.2
GRANT SYSTEM  VIEWCLUSTERMETADATA       TO dba;   -- e.g. select * from crdb_internal.kv_node_status;
GRANT SYSTEM  VIEWACTIVITY              TO dba;   -- e.g. select * from crdb_internal.cluster_locks;
```

Note: a more restrictive alternative is available in place of `VIEWACTIVITY`:

```sql
-- a more conservative variant
GRANT SYSTEM  VIEWACTIVITYREDACTED      TO dba;   -- e.g. select * from crdb_internal.cluster_locks;
```



#### Authorize DBA to View and Modify Cluster Settings

```sql
GRANT SYSTEM  VIEWCLUSTERSETTING        TO dba; -- e.g. SHOW CLUSTER SETTINGS
GRANT SYSTEM  MODIFYCLUSTERSETTING      TO dba; -- e.g. SET CLUSTER SETTING ...
```

Note: a more restrictive is available in place of `MODIFYCLUSTERSETTING` (limited to settings prefixed with sql.*):

```sql
GRANT SYSTEM  MODIFYSQLCLUSTERSETTING   TO dba; -- e.g. SET CLUSTER SETTING sql...                          NOT IN 22.2
```



#### Authorize DBA to Create Statement Bundles

```sql
GRANT SYSTEM  VIEWCLUSTERSETTING        TO dba; -- e.g. explain analyze (debug) <select statement>
GRANT SYSTEM  VIEWDEBUG                 TO dba; -- e.g. explain analyze (debug) <select statement>
GRANT SYSTEM  VIEWSYSTEMTABLE           TO dba; -- e.g. \statement-diag download 954984026515832833         NOT IN 22.2
```



#### Authorize DBA to Observe and Cancel all User Activity (sessions, queries)

```sql
GRANT SYSTEM  VIEWACTIVITY              TO dba; -- enable to see all in SHOW SESSIONS / SHOW QUERIES
GRANT SYSTEM  CANCELQUERY               TO dba; -- enable CANCEL SESSION / CANCEL QUERY
```



#### Authorize DBA to View and Control Cluster JOBs

```sql
GRANT SYSTEM  VIEWJOB                   TO dba; --                                                          NOT in 22.2
GRANT SYSTEM  CONTROLJOB                TO dba; --                                                          NOT in 22.2
```



#### Authorize DBA to View and Operate on Cluster Metadata

```sql
GRANT SYSTEM  VIEWCLUSTERMETADATA       TO dba; -- view range, distribution, store, Raft information 
GRANT SYSTEM  REPAIRCLUSTERMETADATA     TO dba; -- e.g. ALTER RANGE                                         NOT in 22.2
```



#### Authorize DBA to Manage non-admin Roles (users)

```sql
GRANT SYSTEM  CREATEROLE                TO dba; -- auth to manage role (user) lifecycle                     NOT in 22.2
GRANT SYSTEM  CREATELOGIN               TO dba; -- auth to manage usr's pwd policies and ability to connect NOT in 22.2
GRANT SYSTEM  NOSQLLOGIN                TO dba; -- auth to prevent a user from connecting to cluster
```





### Authorization of Data Management Actions

----------

< UNDER DEVELOPMENT >

