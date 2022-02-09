# Query Planning is Slow for Plan with Many Joins

## Version(s): 
All

## Issue Indicators/Example Errors
Indicators for this issue are:

The plan phase of a query takes a long time (on the order of multiple seconds or minutes) to plan.

The query plan involves many joins.

## Description
The cost-based optimizer explores multiple join orderings to find the lowest-cost plan. If there are many joins or join subtrees in the query, this can increase the number of execution plans explored, and therefore the exploration/planning time, exponentially.

## Solution
Set the `reorder_joins_limit` session variable to a lower value to limit the size of the subtree that can be reordered, for example:

```sql
SET reorder_joins_limit = 2;
If the customer is comfortable with the ordering in their query, they can set reorder_joins_limit to 0 for the shortest planning time.

If there is one query in particular that has slow planning time due to this issue, the customer can avoid interfering with other query plans by setting reorder_joins_limit to the desired lower value before executing the slow query, then resetting the session variable to the default after executing the query.
```

## Solution B
If setting and reseting the session variable is cumbersome or if there are multiple independent joins in the query where some may benefit from join reordering another option is to use a join hint.  If the join has a hint specifying the type of join to something other than the default “INNER” (i.e. “INNER LOOKUP, “MERGE”, “HASH” etc).   This works at the expression level and doesn’t affect the entire query (for instance if you have a union of two joins they are independent join expressions).

## References
- https://www.cockroachlabs.com/docs/stable/cost-based-optimizer.html#join-reordering