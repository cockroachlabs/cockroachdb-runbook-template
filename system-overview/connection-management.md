#####  < UNDER CONSTRUCTION, WORK IN PROGRESS >



# Client Connections: Pooling, Load Balancing, Failover and Failback

### Overview

In a database application environment, the application connectivity may become the weakest link without a robust connection management. For continuous service availability, database connectivity requires redundancy and agile connection management, including connection governance, load balancing, failover and failback.

*Connection pooling* serves two important functions:

- alleviates the open/close connection overhead, and
- provides workload concurrency governance.

*Load balancing* ensures:

- a balanced client connection distribution across all-peer CockroachDB cluster nodes, and
- client connections routing only to healthy nodes of the cluster.

CockroachDB only supports [Layer 4 load balancing](https://www.haproxy.com/blog/loadbalancing-faq/), which means TCP/IP connection is established directly between the client - application or connection pool - and a CockroachDB node.

A combination of a connection pool and a load balancer is not the only valid architectural approach. Other viable approaches exists - for example database driver (JDBC, ODBC) based load balancing and failover/failback can be implemented. 



### Load Balancing Example in a Self-hosted Environment (HAProxy)



### Load Balancing in CockroachDB Cloud

In CockroachDB Cloud, a regional [native load balancer](https://cloud.google.com/load-balancing/docs/load-balancing-overview) is deployed to route connections to nodes in that region. This load balancer is implemented in the network fabric and operates across AZs, with AZ loss survivability.   

Upon a new cluster connection request, the load balancer selects a healthy node in the region (as determined by `/health?ready=1`) for the client to connect to. While the load balancing policy attempts to spread the connections evenly, there is no guarantee of a perfect connections balance because the load balancer is not aware of the node's capacity to process the workload and because clients can close connections, creating an imbalance.



### Connection rebalancing after a cluster expansion

A special case of client connections balancing after a cluster expansion needs a special attention. An application that maintains a steady number of open connections will not fully benefit from a cluster expansion unless the added nodes get their equal share of client connections. Even after the load balancer's configuration is updated, as expected, with the references to the new nodes, they will not necessarily receive client connections and will be underutilized.

The new nodes will get connections only when an app/connection pool asks for new connections and that may not be happening if the client side has enough of connection slots in the pool and the pool is reusing the established connections. And even if the app adds connections, and the load balancer's policy is "roundrobin", the imbalance of connections would persist.

To handle this case efficiently and without disorderly connection termination, consider the following remedies:

- Instead of effective, yet simplistic `roundrobin` load balancing algorithm, configure a method that is aware of the current number of server connections , such as [HAProxy](http://cbonte.github.io/haproxy-dconv/1.7/configuration.html#4-balance)'s  `leastconn` , that routs the new connections to the server with the lowest number of connections. This will solve the problem partially, sending new connections to the new, still underutilized nodes.

- Configure a connection max lifetime in the connection pool. Most connection pools support that feature. This will force a periodic connection rebalancing.  `1 hour` may be a good starting point when fine tuning for specific application requirements. A low value may have an undesirable impact on the workload performance due to a re-connect overhead. The value selection should be coordinated with the [snapshot rebalancing rate](../routine-maintenance/change-rebalancing-rate.md) setting (or rather measurements of the elapsed time to complete node data rebalancing for the given rate setting), so *connections* are rebalanced to new nodes reasonably promptly following a cluster expansion.





---



 **< UNDER CONSTRUCTION,  BELOW ARE JUST SCRATCH PAD NOTES >**



Connection management issues to cover:

1. Load Balancing (topics: Layer 4 brief, inter, fall, rise, secondary servers, failover servers, global diagram incl supplimented by DNS a-record). Specifically cover client and server side connection timeouts and idle timeouts.
2. Connections get unbalanced. This is likely to happen over time even if the cluster topo doesn't change. But it's particularly painful post expansion. The uneven connections distribution will not self-heal. The solution there is max conn life in "hours" and [unverified] LB policy `leastconn`. Basically some mechanism to force conn recycling. This is a "slow" problem to develop and a gives operators hours to correct.
3. Connection Failover. [No need to worry about Failback, it's adressed in (1).] [Again, just my hunch] this can't be solved by connection life, because connection life in "minutes" may have an adverse impact on performance.] Connection failover is the thing for uninterrupted "everything", including a rolling maintenance. The solution there is primarily a robust application re-try logic (beyond what we currently document) and crdb config settings tuned for the app environment, e.g. lb settings. This is a "fast" problem to develop and has to be solved promptly, in small # of seconds. This is not exactly a [new topic](https://www.google.com/search?q=database+connection+failover).

Connection failover and failback - how to

Even connection distribution - how to, with and without pool. If pool, max-connection-lifetime (not idle connection timeout!) helps. Examples for PGX and npgsql.



> The additional `connection_wait` may help is some cases, yet it won’t solve the problems

that sounds right. this solution is primarily meant for customers who cannot or will not add the retry logic you are suggesting for connection failover, and who wish to see zero errors when a node is shutting downour team (sql-exp) has received numerous support escalations of this nature, which is why we are now creating this setting



> connection life in “minutes” may have an adverse impact on performance

which performance impacts are you thinking of?

reconnect overhead. client has to wait. server has to do more work

it’s possible v21.2 might have reduced the overhead enough to make ~10 minutes for connection life not have much of an impact on performance

i would not advise customers to force reconnect every couple of minutes. Think hundreds of connections. And pray reconnects don't happen at the same time. 



[https://www.cockroachlabs.com/docs/stable/recommended-production-settings.html#load-balancing](https://www.cockroachlabs.com/docs/stable/recommended-production-settings.html#load-balancing)

---------------



**< UNDER CONSTRUCTION >**

CONNECTION POOLING

https://www.cockroachlabs.com/docs/v21.1/connection-pooling.html

------------



ACTIVE CONNECTION REBALANCING, notes

Guidance about **connection rebalancing**, e.g. after a cluster expansion? Example/Situation:

- The application has many connections, all active most of the time.
- Cluster is orderly expanded (application needs more horse power). the load balancer now knows about new nodes.
- The rebalancing is competed and new nodes can take connections
- But! the new nodes will get connections only when app/connection pool asks for new connections and that's not happening because the client side has enough of connection slots in the pool and the pool doesn't close any. Client connections stay status quo and new nodes are underutilized.
- Another But! Even if the app adds connections, they will be round robined so the imbalance of connections persists.

Options:

- Set a connection max lifetime in the connection pool. Many conn pools support this option (e.g. pgx, npqsql, hikari). It feels the right number is ~1 hour [and tune from there]. A reasonable trade off between maintaining connections balanced and avoiding re-connect overhead.

- load balancer routs based on node load, measured as # connections / node, rather than round robin. That would help partially. Only with additional connections that would go to the new nodes.

- if a connection pool is not used, the application has to invalidate its connections periodically, to force connection rebalancing. Same logic a s max connection life in a connection pool, if used. FWIW, an aggressive idle connection timeout is a bad idea for more than one reason.

  



------------




Cockroach Labs performed lab testing of various customer workloads and found no improvement in scalability beyond:

```
connections = (number **of** cores \* 4)
```

Many workloads perform best when the maximum number of active connections is between 2 and 4 times the number of CPU cores in the cluster.

For more information, please refer to the [connection pooling documentation](https://www.cockroachlabs.com/docs/v21.1/connection-pooling.html).




Opening database connections is computationally expensive and does not necessarily improve throughput. In order to get the best performance from CRDB, we recommend the following steps:

1. Use a connection pool to start up a specific number of database connections initially. This will create the maximum number of connections you&#39;ve allowed for use.
2. Get a connection from the pool.
3. Do some database work.
4. Return the connection to the pool.

This will ensure that each piece of &quot;work&quot; your application is trying to do no longer needs to connect individually and has the added benefit of the pool maintaining the connections in the background.

Below is an example of how to use the steps above:

##### JDBC Connection Pool Example

```
HikariConfig config = new HikariConfig();
 config.setJdbcUrl(&quot;jdbc:postgresql://[connection string]&quot;);
 config.setUsername(&quot;username&quot;);
 config.setPassword(&quot;password&quot;);
**config.setMaximumPoolSize(****40); // step 1**

 HikariDataSource ds = new HikariDataSource(config);

**Connection conn = ds.getConnection(); // step 2**
 // step 3 **conn.commit(); // step 4** 
```

##### Go Connection Pool Example

```
// **Set** connection pool configuration, with maximum connection pool size.
 config, err := **pgxpool.ParseConfig(****&quot;postgres://max:roach@127.0.0.1:26257/bank?sslmode=require&amp;pool\_max\_conns=40&quot;) // step 1**
 if err != nil {
 log.Fatal(&quot;error configuring the database: &quot;, err)
 }

 // Create a connection pool to the &quot;bank&quot; database.
**dbpool, err := pgxpool.ConnectConfig(context.Background(), config)** Step 2if err != nil {
 log.Fatal(&quot;error connecting to the database: &quot;, err)
 }// Step 3
 
 defer dbpool.Close() //step 4
```
