# Role: Database Administrator

### Overview

Organizations may set forth IT security practices that don't allow unrestricted administrative access to databases using an [*admin* role](https://www.cockroachlabs.com/docs/stable/security-reference/authorization#admin-role). In that case, routine Database Administration (DBA) functions need to be delegated to a custom designed [*public* role](https://www.cockroachlabs.com/docs/stable/security-reference/authorization#public-role), with system- or object- level privileges granted to authorize DBA actions.

This article provides a list of the minimally required privileges, grouped by common DBA tasks. An operator can lean on guidance in this article to design a custom authorization model per Organization's requirements and regulations.

The two articles - [Role: Application](../system-overview/role-app.md) and [Role: Application](../system-overview/role-app.md) - are providing blueprint elements to aid implementations of custom authorization models that limit the use of `admin` superusers after DBA users are authorized in the cluster.



### Minimally Required CockroachDB Release

> ✅ The guidance in this article applies to **CockroachDB release 23.1 or later**. It has been tested with v23.1.17.
>
> The [privilege-](https://www.cockroachlabs.com/docs/stable/security-reference/authorization#privileges) based authentication model was introduced in CockroachDB v22.2 and has been continually improving. The granularity and completeness of the system of privileges emancipated to a production deployment grade authorization model in v23.1. 



> ✅ There are no plans to back port privileges introduced in v23.1 to v22.2.
>v23.1 privileges not available in v22.2 are marked in the instructions below, for operator's awareness.





------

### DBA Group and Individual Roles

This non-admin authorization implementation blueprint proposes 2 role profiles:

-  A `DBA` group (role) is created as a "container" of privileges to centralize management of privileges granted to all staff DBA users in the cluster. It simplifies management and ensures consistency of privilege grants to staff DBA users. The `DBA` role is non-interactive, may not be used to create a database session.
- Staff DBA users are identified by interactive roles that inherit authorizations via the DBA group membership.



> ✅ Note: The topic of *authentication* is entirely outside the scope of this article.
> This article is dedicated to DBA authorization. Therefore all authentication provisions are omitted to eliminate clutter.



All instructions in this section must be executed as an **admin user** (role).

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





-------------

### Authorization of Administrative Actions

The required privileges grants are grouped by common DBA tasks. The same privilege may appear in different task groups. A privilege can be granted repeatedly. 
Customize as it suits Organization's IT practices.

All instructions in this section must be executed as an **admin user** (role).



#### Authorize DBA to View System / Metadata Tables

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



#### Authorize DBA to View and Operate on Cluster Metadata

```sql
GRANT SYSTEM  VIEWCLUSTERMETADATA       TO dba; -- view range, distribution, store, Raft information 
GRANT SYSTEM  REPAIRCLUSTERMETADATA     TO dba; -- e.g. ALTER RANGE                                         NOT in 22.2
```



#### Authorize DBA to View and Modify Cluster Settings

```sql
GRANT SYSTEM  VIEWCLUSTERSETTING        TO dba; -- e.g. SHOW CLUSTER SETTINGS
GRANT SYSTEM  MODIFYCLUSTERSETTING      TO dba; -- e.g. SET CLUSTER SETTING ...
```

Note: A more restrictive is available in place of `MODIFYCLUSTERSETTING` (limited to settings prefixed with sql.*):

```sql
GRANT SYSTEM  MODIFYSQLCLUSTERSETTING   TO dba; -- e.g. SET CLUSTER SETTING sql...                          NOT IN 22.2
```



#### Authorize DBA to Create Databases

```sql
GRANT SYSTEM  CREATEDB                  TO dba; -- e.g. CREATE DATABASE ...;
```



#### Authorize DBA to Manage non-admin Roles (users)

```sql
GRANT SYSTEM  CREATEROLE                TO dba; -- auth to manage role (user) lifecycle                     NOT in 22.2
GRANT SYSTEM  CREATELOGIN               TO dba; -- auth to manage usr's pwd policies and ability to connect NOT in 22.2
```

Note:  For completeness, the authorized user management actions by DBA may need to include an ability to [temporarily] disable user SQL access. For example, to execute "close database gates", or while making disruptive schema changes in a development environment in controlled/graceful manner.

The privilege controlling user's ability to connect is `NOSQLLOGIN`. This privilege is unique in being a "negative" privilege. Therefore enabling a non-admin DBA role to be able to grant `NOSQLLOGIN` to other roles (like an application user) is not possible, because `GRANT SYSTEM NOSQLLOGIN TO dba WITH GRANT OPTION;` will harm DBA's own ability to connect.

This means that *only a cluster admin role* (like `root`) *can disable other user's ability to connect* with

```sql
GRANT SYSTEM  NOSQLLOGIN TO app_or_interactive_user_role;
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



#### Authorize DBA to Create and Manage External Changefeed Connections

```sql
GRANT SYSTEM  EXTERNALCONNECTION        TO dba;
```





------

### Tightening Permissive Built-In Authorizations of All Non-Admin (Public) Roles

##### `public` role

CockroachDB follows the PostgreSQL authorization model which includes a [built-in public role](https://www.cockroachlabs.com/docs/stable/security-reference/authorization#public-role), effectively a group. All non-admin roles (users) inherit group privileges from the `public` role. These privileges are intrinsically granted to the built-in `public` and may be overlooked by administrators in charge of designing the authorization scheme for a cluster deployment. The implicit `public` privileges inherited by each non-admin role are reported only with

```sql
SHOW GRANTS FOR public;
```

Note that the inherited privileges are not shown with the ` SHOW GRANTS ... `  command for any "regular" (non-admin) role.

A specific concern may be with the `CONNECT` privilege inherited by every role, as can be confirmed with  `SELECT * FROM [SHOW GRANTS FOR public] WHERE privilege_type = 'CONNECT'`.  While the `CONNECT` privilege does not control SQL connections to a database, it [grants the ability to view database's metadata](https://www.cockroachlabs.com/docs/v23.2/security-reference/authorization#supported-privileges) and applies to all databases in the cluster.

An operator may elect to revoke the built-in `CONNECT` privilege from `public` in all databases, including built-in (`postgres`, `defaultdb`) and all databases created by DBAs. See a complete example at the bottom of this section.

##### `public` schema

Also following a PostgreSQL model, all CockroachDB databases have a default schema called `public`, accommodated by the default search path.

The `public` schema is open to all roles (users) for DDL and DML statements. All cluster users, can create tables, read and write records in the `public` schema without any explicitly granted privileges in that database.  

An operator may elect to revoke all privileges on `public` schema from `public` role, in all existing databases, including built-in (`postgres`, `defaultdb`) and all databases subsequently created by DBAs.



The example below limits the metadata and data access in a cluster by removing implicit `public` role privileges and permissive `public` built-in schemas in built-in databases `postgres` and `defaultdb`. Execute as an **admin user** (role):

```sql
REVOKE CONNECT ON DATABASE postgres FROM public;
REVOKE ALL ON SCHEMA postgres.public FROM public;

REVOKE CONNECT ON DATABASE defaultdb FROM public;
REVOKE ALL ON SCHEMA defaultdb.public FROM public;
```

> ✅ Repeat the same 2 `REVOKE` statements for each new database created in the cluster.  





------

###### Related Articles:

###### 	 [Role: Application](../system-overview/role-app.md)

###### 	[Role: Checking and Reporting Authorizations](../system-overview/role-privileges.md)

