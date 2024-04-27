# Role: Checking and Reporting Authorizations

### Overview

CockroachDB follows the PostgreSQL authorization model, which is based on a unified ***role*** concept that represents individual database users as well as groups of users. A role is effectively a container of privileges. Roles can be members of other roles (effectively "groups"), inheriting parent role's privileges.

A role can be authorized with privileges through several mechanisms

- with an explicit privilege grant
- with a privilege inherited from a parent (group) role
- with a database -scoped default privilege
- with a cluster -scoped system privilege

This article provides an operator with guidance (commands) for checking and reporting the level of authorization of any role in a CockroachDB cluster, across all mechanisms for receiving privileges.



### Nuanced Points about Roles and Privileges

CockroachDB operators should be aware of the following insights:

- Following the PostgreSQL model, every database in CockroachDB cluster has a `public` schema with permissive default privileges and the default schema `search_path` pointing to it. Unless these privileges are explicitly removed, any database user (role) will be able to, for example, create/delete tables in any database.  The  [Role: Database Administrator](../system-overview/role-dba.md#tightening-permissive-built-in-authorizations-of-all-non-admin-public-roles) article includes instructions for tightening the default database access. Note that in v23.2 and later, a [cluster setting](https://www.cockroachlabs.com/docs/stable/cluster-settings) `sql.auth.public_schema_create_privilege.enabled`  can be set to `false`  to deny the the default `CREATE` privileges on the `public` schema. 
- The `SHOW DEFAULT PRIVILEGES` command output depends entirely on the current database. Ensure the current database is set as expected with `SELECT current_database();`.
- The command `SHOW GRANTS ON ROLE` shows all role "groups" and its members. While the [built-in admin role](https://www.cockroachlabs.com/docs/stable/security-reference/authorization#role-admin) "group" is included for admin users, the *[built-in public role](https://www.cockroachlabs.com/docs/stable/security-reference/authorization#public-role) "group" that "parents" all non-admin users in the cluster is not included*. Since all non-admin roles inherit the privileges granted by default to the `public` role, the actual user authorization may be obscured. For example, the ` SHOW GRANTS ... `  command for any non-admin role will not surface the in-effect privileges inherited from `public`. To those, use `SHOW GRANTS FOR public;` 
- There is a [known issue](https://github.com/cockroachdb/cockroach/issues/121808), whereby a non-admin user creating a new database does not own the `public` schema intrinsic in all databases. This results in two limitations - a non-admin database owner can not `DROP` its own database, nor can run `REVOKE ALL ON SCHEMA ods.public FROM public;` to tighten up the permissive access to `public` schema in all databases. To work around these limitations, run either command as an `admin` (`root`) user.
- A `public` schema in a database can not be dropped. This is a known limitation and a diversion from the PostgreSQL authorization model.



### Checking the Effective Role Authorization

To check if a privilege is granted to a role in a database, use the `has_database_privilege()` [built-in function](https://www.cockroachlabs.com/docs/stable/functions-and-operators.html#compatibility-functions). For example, to check if a user `dba` has `CONNECT` privilege to database `postgres`, use:

```sql
SELECT has_database_privilege('dba', 'postgres', 'CONNECT');
```



### Report All Effective Privileges Granted to a Role

The following query reports all granted privileges to a **public** role via all privilege granting mechanisms. In particular, it explicitly includes the implied inheritance from the intrinsic parent role ("group") `public`.

For example, for a role `dba_staff_minnie`, (following the example in the article [Role: Database Administrator](../system-overview/role-dba.md#tightening-permissive-built-in-authorizations-of-all-non-admin-public-roles)) use:

```sql
WITH RECURSIVE traverse_inheritance (role_name) AS (
  (SELECT 'dba_staff_minnie' as role_name
   -- all non-admin roles are "public", however role (group) "public" is not reported by inheritance query
   -- so adding 'public' to surface all inherited privileges from the implicit public role group membership
   UNION ALL
   SELECT 'public' as role_name) 
  UNION ALL
  SELECT groups.role_name FROM [SHOW GRANTS ON ROLE] groups JOIN traverse_inheritance ON groups.member = traverse_inheritance.role_name
)
-- Individual privileges granted to, and inherited by a role 
SELECT grantee, privilege_type as privilege, database_name as database, COALESCE(schema_name, '') as schema,
                                                                        COALESCE(relation_name, '') as relation FROM [SHOW GRANTS] grants
  JOIN traverse_inheritance ON  grants.grantee = traverse_inheritance.role_name
UNION ALL
-- Default privileges (current database only!)
SELECT grantee, privilege_type, current_database() || ' (DEFAULT PRIVILEGE)', '', object_type FROM [SHOW DEFAULT PRIVILEGES] defgrants
  JOIN traverse_inheritance ON  defgrants.grantee = traverse_inheritance.role_name
UNION ALL
-- System privileges granted to, and inherited by a role
SELECT grantee, privilege_type, 'SYSTEM PRIVILEGE', '', '' FROM [SHOW SYSTEM GRANTS] sysgrants
  JOIN traverse_inheritance ON  sysgrants.grantee = traverse_inheritance.role_name
ORDER BY grantee;

```



The following query reports all granted privileges to an **admin** role via all privilege granting mechanisms. The only difference from the public role query above is that an `admin` role query does not need to explicitly account for the intrinsic parent role `admin`, which is automatically reported by `SHOW GRANTS ... `

For example, for a role `root`, use:

```sql
WITH RECURSIVE traverse_inheritance (role_name) AS (
  SELECT 'root' as role_name
  UNION ALL
  SELECT groups.role_name FROM [SHOW GRANTS ON ROLE] groups JOIN traverse_inheritance ON groups.member = traverse_inheritance.role_name
)
-- Individual privileges granted to, and inherited by a role 
SELECT grantee, privilege_type as privilege, database_name as database, COALESCE(schema_name, '') as schema,
                                                                        COALESCE(relation_name, '') as relation FROM [SHOW GRANTS] grants
  JOIN traverse_inheritance ON  grants.grantee = traverse_inheritance.role_name
UNION ALL
-- Default privileges (current database only!)
SELECT grantee, privilege_type, current_database() || ' (DEFAULT PRIVILEGE)', '', object_type FROM [SHOW DEFAULT PRIVILEGES] defgrants
  JOIN traverse_inheritance ON  defgrants.grantee = traverse_inheritance.role_name
UNION ALL
-- System privileges granted to, and inherited by a role
SELECT grantee, privilege_type, 'SYSTEM PRIVILEGE', '', '' FROM [SHOW SYSTEM GRANTS] sysgrants
  JOIN traverse_inheritance ON  sysgrants.grantee = traverse_inheritance.role_name
ORDER BY grantee;

```

