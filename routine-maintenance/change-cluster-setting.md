# Procedure: Change Cluster Setting

### About this Procedure

A [Cluster Setting](https://www.cockroachlabs.com/docs/stable/cluster-settings.html) can be changed with a [SQL statement](https://www.cockroachlabs.com/docs/stable/cluster-settings.html#change-a-cluster-setting), most commonly via an [interactive SQL utility](https://www.cockroachlabs.com/docs/stable/cockroach-sql.html).

This section provides the insights how custom settings interoperate with default settings, specifically in the context of CockroachDB version upgrades when a *default value* of some *setting* changes from the previous CockroachDB version. This information ensures clarity and transparency, eliminating inadvertent configuration errors during the cluster lifecycle.



### How Custom Cluster Settings are Managed

The default values of cluster settings are compiled into the code.

When a cluster setting is changed with `SET CLUSTER SETTING`, a record is created in the system table `system.settings`. Explicitly set custom Cluster Settings values stored in the `system.settings` *preempt the default values* of the cluster settings.

To see all custom cluster settings currently in effect, query the `system.settings` table, e.g.:

```sql
SELECT * FROM system.settings;
```



### Default Values of Cluster Settings Change Can Change!

Over the lifecycle of a CockroachDB cluster, the default values of cluster settings can (in fact - likely to) change.



 of an existing one that has been set? to date, no, i don't think we've ever done that.
We change defaults in major versions, which would take effect int he new binary if you haven't set an explicit value.
Major versions delete settings quite often too, sometimes with a deprecation period and notice if they were marked public, but otherwise with no notice.
But i don't think we've changed the value of an explicitly set setting before (doing so would require writing a custom upgrade migration to find and change it). (edited) 

Alex Entin
Here is a scenario. In an old cluster, changed a default for setting S temporarily, then changed it back.
Then upgraded to a new version that changed the default for S.
In this case S will remain at the old/current setting after an upgrade.
I can predict the behavior with a simple test - is there a setting S record in system.settings.
Did I get this right?

David Taylor
Depends on how you set it back, but yes, your test is correct


David Taylor
  17 hours ago
If you explicitly set it to the save value as the default, e.g.:
Version N:
Setting foo has default value 5.
You set it to 10, then "set it back" by running SET CLUSTER SETTING foo = 5;
You upgrade to N+2, where the default is now 20. Your cluster will be using 5, since that is what you set it to. (edited) 


Alex Entin
  17 hours ago
thx!!! that's what i meant, exactly


Alex Entin
  17 hours ago
all set


David Taylor
  17 hours ago
If you use RESET or = DEFAULT however, the story is different:
Version N:
Setting foo has default value 5.
You set it to 10, then "set it back" by running RESET CLUSTER SETTING foo; or by running SET CLUSTER SETTING foo = DEFAULT;
You upgrade to N+2, where the default is now 20. Your cluster will be using 20, since after you reset it, there is no persisted explicit value. (edited) 


Alex Entin
  17 hours ago
yeap yeap, sure


Alex Entin
  17 hours ago
typical "reset" among users is an explicit set

David Taylor
  17 hours ago
as you guessed, the difference is whether there is still a row in system.settings after you "set it back"; with SET = <the same value as default> there is, with RESET or SET .. = DEFAULT there is not (those both map to a DELETE)
