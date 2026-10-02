# Scaling PostgreSQL to power 800 million ChatGPT users
- initially they had a single primary Azure PostgreSQL flexible server instance and nearly 50 read replicas spread over multiple regions globally.
- for high traffic, initially they increased instance size and added more read replilcas
- in practice they often faced upstream issues like cache miss, expensive multi-way joins, write storms -> this caused in sudden spikes in db load, retries triggered a vicious cycle increasing the loads degrading the services

**the vicious cycle under load**
```
[cache layer failure]   [expensive queries] [write spike]
                    |
                    |
                    V
            [postgres overload]
                    |
                    V
            [reqs become slow or time out]
                    |
                    V
            [more reqs due to retries] - this further incr the load
```

- though reads were scalable, the MVCC impl of postgres made the writes less efficient
- to mitigate this they migrated the writes to sharded systems like Azure Cosmos DB and new workloads too default to these sharded systems
- they didn't shard postgres as their workloads are read heavy and given that they've done extensive optimizations (~M QPS) and it would be time consuming to shard existing application workload
- **reducing load on the primary:** they reduced redundant writes and introduced lazy writes wherever appropriate also when backfilling table fields, they enforced strict rate limits
- **query optimization:** optimized common OLTP anti-patterns. if joins are necessary, they considered breaking down the query and moved complex join logic to the application layer. Also check the SQL produced by ORMs and find long-running idle queries in pgsql. configuring timeouts like `idle_in_transaction_session_timeout` is essential to prevent them from blocking autovacuum.
- **single point of failure:** a single writer was single point of failure. they offloaded the reads to replicas while write ops could still fail, the impact would be reduced; it was no longer a SEVO as reads remained available. they run primary in High-Availability(HA) mode with hot standby, a continuously synchronized replica that's always ready to take over serving traffic. also they had mulitple replicas in each region to handle replica failures
- **workload isolation:** they split reqs into low-priority and high-priority tiers and routed them to specific instances so that even though a low-priority-resource-intensive req couldn't degrade the perf of high-priority reqs
- **connection pooling:** - they had incidents that exhausted all available connections caused by connection storms. they deployed PgBouncer as a proxy layer to pool db connections. this also reduced avg connection time from 50ms to 5ms. they co-located the proxy, client and replicas in the same region to minimize network overhead as inter-region connections and reqs could get expensive
- they run multiple k8s deployments behind the same k8s service which load-balances traffic across pods
- **caching:** to overcome a burst of cache misses they imlemented a cache locking(and leasing) mechanism so that only a single reader fetches data from db (multiple readers miss the same cache key, one of them acquire lock and reads from db and repopulates the cache, all others wait)
- **scaling read replicas:** primary streamed WAL data to read replicas but incr replicas increased the pressure on network bandwidth and CPU causing unstable replica lag. they are experimenting with Azure PostgreSQL on cascading replication where intermediate replicas relay the WAL to downstream replicas. however it introduces operational complexity(failover mgmt) they're still testing
- **rate limit:** they implemented rate-limiting across multiple layers - application, connection pooler, proxy and query to prevent sudden traffic spikes from overwhelming db instances and triggering cascading failures. also avoided very-short retry intervals to avoid retry storms. also enhanced the ORM layer to support rate limiting when needed and fully  block some specific queries
- **schema management:** sometimes a small schema change might trigger full table rewrite so they only permit lightweight schema changes and enforce a strict 5-sec timeout on schema changes. creating and dropping indexes concurrently is allowed. schema changes are restricted to existing tables, a new table must be in alternative sharded systems like Azure CosmosDB rather than postgres.