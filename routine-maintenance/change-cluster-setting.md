# Procedure: Change Cluster Setting

### About this Procedure

A [Cluster Setting](https://www.cockroachlabs.com/docs/stable/cluster-settings.html) can be changed with a [SQL statement](https://www.cockroachlabs.com/docs/stable/cluster-settings.html#change-a-cluster-setting), most commonly via an [interactive SQL utility](https://www.cockroachlabs.com/docs/stable/cockroach-sql.html).

This section provides the insights how custom settings interoperate with default settings, specifically in the context of CockroachDB version upgrades when a *default value* of some *setting* changes from the previous CockroachDB version. This information ensures clarity and transparency, eliminating inadvertent configuration errors during the cluster lifecycle.



### Custom vs. Default Cluster Settings

The *default* values of cluster settings are compiled into the code.

A *custom* setting is any setting that was explicitly [changed](https://www.cockroachlabs.com/docs/stable/cluster-settings.html#change-a-cluster-setting) by an operator. A custom value that coincidently matches a default value is still a custom setting!

When a cluster setting is changed with `SET CLUSTER SETTING`, a record is created in the system table `system.settings`. Explicitly set custom Cluster Settings values stored in the `system.settings` *preempt the default values* of the cluster settings.

To see all custom cluster settings currently in effect, query the `system.settings` table, e.g.:

```sql
SELECT * FROM system.settings;
```



To revert a custom setting to its default value, either [RESET](https://www.cockroachlabs.com/docs/stable/reset-cluster-setting.html) it or set it explicitly to [= DEFAULT](https://www.cockroachlabs.com/docs/stable/set-cluster-setting.html#reset-a-setting-to-its-default-value). These two methods are semantically equivalent and result in the setting's record being removed from `system.settings`.

> Note that *[changing](https://www.cockroachlabs.com/docs/stable/cluster-settings.html#change-a-cluster-setting)* a custom cluster setting back to a value that matches the default value does *NOT* put the setting's default value in effect. This setting will remain a *custom* setting. This may have direct implications during a subsequent CockroachDB upgrade, as explained below.



### Default Values of Cluster Settings Change Can Change!

Over the lifecycle of a CockroachDB cluster, the default values of cluster settings can (in fact - likely to) change.

Per the current CockroachDB practice, default values for cluster settings can only change between major releases.

This is how a given cluster setting will be affected by an upgrade to a new CockroachDB version that *has a different default value* for that cluster setting:

- If cluster setting's *default* value is in effect, after an upgrade the *new* default value for this cluster setting setting will take effect.
- If cluster setting's *custom* value is in effect, this custom setting will remain in effect across upgrades and will not be affected by the new default.

 

##### Illustration

If you explicitly set it to the save value as the default, e.g.:
Version N:
Setting foo has default value 5.
You set it to 10, then "set it back" by running SET CLUSTER SETTING foo = 5;
You upgrade to N+2, where the default is now 20. Your cluster will be using 5, since that is what you set it to. (edited) 



If you use RESET or = DEFAULT however, the story is different:
Version N:
Setting foo has default value 5.
You set it to 10, then "set it back" by running RESET CLUSTER SETTING foo; or by running SET CLUSTER SETTING foo = DEFAULT;
You upgrade to N+2, where the default is now 20. Your cluster will be using 20, since after you reset it, there is no persisted explicit value. (edited) 

