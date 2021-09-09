
 **< UNDER CONSTRUCTION >**

https://www.cockroachlabs.com/docs/v21.1/connection-pooling.html


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
