# Procedure:  Release Upgrade and Downgrade

### About this Procedure

This section supplements to the documented [CockroachDB version upgrade](https://www.cockroachlabs.com/docs/stable/upgrade-cockroach-version.html) procedure with clarifications for topics that operators have raised with Cockroach Labs field engineers.

While the general outline of an upgrade procedure remains largely the same between releases, operators are strongly encouraged to check the documented guidance for *each* target release for possible release-specific actions, deprecated features, and backward-incompatible functionality changes.



### About CockroachDB Release Numbers

A CockroachDB version consists of three numbers and an optional suffix - [Y.R.P-S](https://www.cockroachlabs.com/docs/releases/index.html#release-naming). For example, `22.1.12`, `22.2.0-rc.3`

The leading two numbers taken together Y.R denote a CockroachDB major release. The major release version describes a stable, documented feature set.

The third number P denotes a patch (or maintenance) release. Patch releases include sets of fixes for software defects. Path release numbers increment monotonously and indicate the stability level progression within a major version.

A hyphen separated suffix `S` may be used for internal versioning purposes (not necessarily visible to users) and in previews. Production releases do not use a suffix in the version number. 



### CockroachDB Versions

There are two "version" contexts that operators should be versed in:

- **The version of CockroachDB software a node is running.** This is the version built into every CockroachDB node executable (binary). Under the normal operating conditions, the versions of the executable on all cluster nodes match exactly. However, during a limited period of time a cluster can be operated with different versions of CockroachDB node executables across the cluster nodes. This is an important aspect of the software design that enables upgrades without service interruptions, whereby individual nodes are upgraded one at a time in a rolling fashion. 
- **The version of on-disk format.** This is the format version of CockroachDB data and metadata stored in [Pebble](https://www.cockroachlabs.com/docs/stable/architecture/storage-layer.html#pebble) [SST](https://www.cockroachlabs.com/docs/stable/architecture/storage-layer.html#ssts) files. Under the normal operating conditions, the major versions of the node's executables matches the major version of the on-disk format. However, during a limited period of time, e.g. during a major release upgrade before the upgrade is finalized, a cluster can be operated with mismatching versions of CockroachDB node executables and the cluster's on-disk format version. This is another important aspect of the software design that enables upgrades without service interruptions.



##### CockroachDB Node Software Version

Operators can use either of the 3 methods to view the node's software version. Note that different cluster nodes may be running different version executables.

1) Via SQL interface:

```
defaultdb> select crdb_internal.node_executable_version();
  crdb_internal.node_executable_version
-----------------------------------------
  22.1
(1 row)
```

This built-in function returns the software version *of the node that the SQL client is connected to*. 

2. Via OS level (shell) command:

```sdfsdf
% cockroach version --build-tag
v22.1.12
```

3. Via CockroachDB Console

   ![](./res/overview.png)

   The software version of the node that *the Console is connected to* is in the upper right area (next to the cluster id) of the "Overview" tab.



##### On-disk Data Format Version

The version of the current *cluster-wide* on-disk format is reported via the cluster setting `version`:

```
defaultdb> show cluster setting version;
  version
-----------
  21.1
```

> ✅ The cluster setting `version` is maintained by the database software. Users should not attempt to modify this setting with `set cluster setting`.



##### Versioning Design Highlights

- All CockroachDB software upgrades are "rollable", by design. This applies to major and patch release upgrades. This is possible because the software is designed to allow operations for a limited time with mis-matched versions of the executable (binary) and/or on-disk data format versions.

- Every CockroachDB node software executable is able to read and write two versions of on-disk format - the on-disk format version that matches the major version of the node executable and the previous major version of the node executable. For example, 22.1 node software can read and write on-disk data formats 22.1 (matching major) and 21.2 (previous major). 

- Different major versions of on-disk data format are *incompatible*. The on-disk format can upgraded but can *not* be downgraded. This has a principled effect on available release [downgrade](#release-downgrade) options.

- Cockroach Labs is enforcing the development practice of maintaining the on-disk format and inter-node communication protocols unchanged within the series of patch releases of a major version. The current practice stipulates, however, that it is permissible to break backwards-compatibility under extraordinary circumstances such as discovered security vulnerability with an extreme impact. Historically, there had been no case of backward incompatible patch releases. 




### Release Upgrade

zzzz

##### Considerations for Production Deployment Release

< UNDER CONSTRUCTION >

zzz



##### Procedure

zzz



### Release Downgrade

< UNDER CONSTRUCTION >

zzz



### FAQ

**Q:**  We upgraded our cluster from `21.2.15` to `22.1.9`. The DB Console now shows version `22.1.9`, however the `show cluster setting version` displays `21.2-20`. Please explain the difference.

**A:**  This is a [difference between the versions](#cockroachdb-versions) of an executable (binary) and on-disk data format. Until a major release upgrade is finalized, these [versions may not match](#versioning-design-highlights). They will match after the upgrade is [finalized](https://www.cockroachlabs.com/docs/stable/upgrade-cockroach-version.html#step-5-finish-the-upgrade).



**Q:**  How do we interpret the result of `show cluster setting version` returning `21.1-1112`?

```
defaultdb> show cluster setting version;
   version
-------------
  21.1-1112
(1 row)
```

A repeated call returns `21.1-1126`, i.e. the version is changing over time. We upgraded to `21.2.15` and expect the cluster setting `version` to show`21.2`. Seeing `21.1-XXXX` is confusing. 

**A:**  You are observing a major version upgrade finalization going through *internal* on-disk format upgrades. This is a online background operation and requires no operator intervention. You can observe the progress details on the Jobs page in DB Console or with `show jobs`.

Major releases may include new features that require a change in the *metadata* format. When upgrade is finalized, the cluster setting `version` shows the matching major release number. In your case it will be `21.2`. At that point all features in the new release (`21.2`) are available to database users.

