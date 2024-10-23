1. What are all the prerequisites to perform a successful cluster/database level restore in following scenarios?
    - When source and targets are from different regions.
    - When the source is in 3 regions and restore is happening on 2 regions.
    - Please share any more odd scenarios faced by other customers.

    [ANSWER]:
    
    
    
2. Cluster to cluster restore scenarios:

    - How can we exclude the source cluster settings while doing the cluster restore.

    - How can we restore only the cluster settings without any data.

    [ANSWER]:

      

3. How can we restore a table with different name in the same database?

    [ANSWER]:

    

4. How to exclude the `system.users` table data when we do a cluster level restore? This is required while restoring production data to non-prod.
    - We know we can take backup prior to cluster restore but do we have any other option to achieve this?

    [ANSWER]:

    

5. What are the options for Protected timestamps and how it works.

    - Do we need to set any parameter to have this enabled ? What would be the impact to the database performance in-case if we pause the backup job for sometime.

    [ANSWER]:

    

7. How is GC TTL parameter related with Incremental Backup?
    - low value for this GC TTL /frequent INC backup
    - High value for this GC TTL/ Less frequent INC backup

    [ANSWER]:

    

8. What is the preferred way of scheduling the backup?  Cron job or scheduling as jobs at database level

    [ANSWER]:

    

9. Why taking backup with `AS OF SYSTEM TIME '-10s'` is recommended? What is the difference if we run the backup without `as of system time '-10s'`?

    [ANSWER]:

    

10. Backups are by default encrypted or do we need to encrypt them explicitly.

    [ANSWER]:

    https://www.cockroachlabs.com/docs/stable/take-and-restore-encrypted-backups.html

    

11. Does cockroach takes care of compressing the backup?

    [ANSWER]:

     

12. What is the expected behavior of "latest" keyword in RESTORE command?

     - When we have full and successive incremental backups in backup collection along with adhoc backups of schema/table then in that case if we use latest for restoring the database then it is not working

    [ANSWER]:
    
    


13. How the key revisions are backed up while doing the incremental backups with revision history.

    [ANSWER]:
    
    


14. If my lease holders are always in East-1 and East-2, what is the advantage of taking locality aware backup with all 3 regions ( East-1,East-2 and West-2)

    [ANSWER]:

    

15. How is block corruption handled in CockroachDB?

    [ANSWER]:

    

16. What is the impact to the node/cluster if SST file/s removed at OS level?

    [ANSWER]:

    

17. How metadata corruption is handled?

    [ANSWER]:

    

18. How to list the backups from multiple S3 buckets in case of locality aware backups.

    [ANSWER]:

    

19. How to create external connection for Locality aware backups (if we use more than one S3 buckets) ?

    [ANSWER]:
    
    
