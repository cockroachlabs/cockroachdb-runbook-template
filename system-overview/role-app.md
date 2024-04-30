# Role: App Role Setup and Authorization

### Overview

Organizations may set forth IT security practices that don't allow unrestricted administrative access to databases using an [*admin* role](https://www.cockroachlabs.com/docs/stable/security-reference/authorization#admin-role). In that case, routine Database Administration (DBA) functions need to be delegated to a custom designed [*public* role](https://www.cockroachlabs.com/docs/stable/security-reference/authorization#public-role), with system- or object- level privileges granted to authorize DBA actions.

This article builds on [Role: DBA](../system-overview/role-dba.md), completing the custom authorization implementation blueprint with examples of common DBA tasks:

- Create application roles - read-write (OLTP) and read-only (reporting)
- Create a database and authorize database users (application roles) in the new database
- Create an application schema

Instructions in this article shall be executed by a cluster **dba role** created in the article [Role: DBA](../system-overview/role-dba.md) (recommended) or by a cluster admin (root).





------

### Create Application Roles (users)

In this example a non-admin DBA user creates 2 new roles to facilitate application's database logins - a read-only reporting application role, and read-write OLTP application role.



> ✅ Note: The topic of *authentication* is entirely outside the scope of this article.
> This article is dedicated to DBA authorization. Therefore all authentication provisions are omitted to eliminate clutter.



```sql
--------------------------------------------------------------+
-- Connect as non-admin dba persona e.g. 'dba_staff_minnie'   |
--------------------------------------------------------------+

-- Reporting application, with read privileges only
CREATE USER app_ro_report WITH LOGIN;

-- OLTP application, with read and write privileges
CREATE USER app_rw_oltp WITH LOGIN;
```

The new application roles do not have access to business data in the cluster until explicitly authorized. However they have privileges inherited from `public` group (role), as explained in [Role: Checking and Reporting Authorizations](../system-overview/role-privileges.md#nuanced-points-about-roles-and-privileges).





------

### Create Database and Authorize Database Users

< work in progress >

```sql

-- Create a new operational data store and make DBA its owner, implicitly - with ALL PRIVILEGES

-- Note https://github.com/cockroachdb/cockroach/issues/121808
-- A database owner 
-- DROP   DATABASE IF EXISTS     ods  CASCADE;
CREATE DATABASE IF NOT EXISTS ods;
ALTER DATABASE ods OWNER TO dba;    -- make the non-interactive DBA group (role) the owner! 

-- Ensure the current database is set. Default privileges require the current database context.
USE ods;

-- Grant authorizations to 2 application roles - read-write and read-only
GRANT CONNECT      ON DATABASE ods                  TO app_rw_oltp; -- e.g. show schemas; show tables;
ALTER DEFAULT PRIVILEGES GRANT USAGE,
                               CREATE  ON SCHEMAS   TO app_rw_oltp;
ALTER DEFAULT PRIVILEGES GRANT INSERT,
                               SELECT,
                               UPDATE,
                               DELETE  ON TABLES    TO app_rw_oltp;
ALTER DEFAULT PRIVILEGES GRANT SELECT,
                               USAGE   ON SEQUENCES TO app_rw_oltp;
ALTER DEFAULT PRIVILEGES GRANT USAGE   ON TYPES     TO app_rw_oltp;
ALTER DEFAULT PRIVILEGES GRANT EXECUTE ON FUNCTIONS TO app_rw_oltp;

GRANT CONNECT      ON DATABASE ods                  TO app_ro_report; -- e.g. show schemas; show tables;
ALTER DEFAULT PRIVILEGES GRANT USAGE   ON SCHEMAS   TO app_ro_report;
ALTER DEFAULT PRIVILEGES GRANT SELECT  ON TABLES    TO app_ro_report;
ALTER DEFAULT PRIVILEGES GRANT USAGE   ON SEQUENCES TO app_ro_report;
ALTER DEFAULT PRIVILEGES GRANT USAGE   ON TYPES     TO app_ro_report;
ALTER DEFAULT PRIVILEGES GRANT EXECUTE ON FUNCTIONS TO app_ro_report;


-- Tighten up data access from permissive Public role group and Public schemas
REVOKE CONNECT ON DATABASE ods FROM public;
-- As of May 2024 the statement below requires admin (root) role
-- as a temporary workaround until this defect is fixed
-- (https://github.com/cockroachdb/cockroach/issues/121808)
REVOKE ALL ON SCHEMA ods.public FROM public;
```





------

### Create Application Schema

< work in progress >

```sql
-- Ensure the current database is set
USE ods;

-- Create a schema (optional)
-- The schemas in a new database must be created *after* the DEFAULT PRIVILEGES are set
CREATE SCHEMA IF NOT EXISTS ods.reporting AUTHORIZATION dba;
ALTER SCHEMA ods.reporting OWNER TO dba;  -- make the non-interactive DBA group (role) the owner!


-- ISSUE: Non-admin can't set search path per database. This could be helpful but requires admin:
-- ALTER DATABASE  ods SET search_path = reporting;
-- So working around it:
-- (takes effect on new connections)
ALTER USER  app_ro_report    SET SEARCH_PATH TO reporting;
ALTER USER  app_rw_oltp      SET SEARCH_PATH TO reporting;
ALTER USER  dba_staff_mickey SET SEARCH_PATH TO reporting;
ALTER USER  dba_staff_minnie SET SEARCH_PATH TO reporting;
-- For convenience of the current user `dba_staff_minnie`
-- also SET to take effect immediately in the current session
SET SEARCH_PATH TO reporting;


-- Create ODS schema (non-realistic yet sufficient 1 table example)
CREATE TABLE reporting.account (id INT NOT NULL PRIMARY KEY, balance INT NOT NULL);
ALTER  TABLE reporting.account OWNER TO dba;

CREATE SEQUENCE ordinal CACHE 10;
ALTER  SEQUENCE reporting.ordinal OWNER TO dba;

```





------

###### Related Articles:

###### 	 [Role: DBA Role Setup and Authorization](../system-overview/role-dba.md)

###### 	[Role: Checking and Reporting Authorizations](../system-overview/role-privileges.md)

