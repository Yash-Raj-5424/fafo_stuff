can be used for cache, distributed locks, leaderboards, rate limiting, sometimes a replacement of kafka etc.

it's a single threaded, in memory data structure server

single threaded is what makes it fast - no need to acquire locks for operations

newer versions offload I/O and bg work to other threads

**two persistence modes: periodic snapshots and Append Only File logs every write but only fsyncs once per sec(intentional)**

the core structure is a key-val store. key is string and every object is stored as value to these keys

data structures supported by redis: strings, Hashes (objects/dictionaries), Lists, Sets, Sorted Sets (Priority Queues), Streams(append-only logs), Geospatial Indexes

newer ones support bloom filters, JSON and time series. communication patterns supported are pub/sub, streams

speaks a simple wire protocol called RESP(REdis Serialization Protocol) - binary safety, fast parsing

redis can run as a single node with high availability replica or as a cluster

 in the cluster: architecture looks something like: 
 
 ``` key -> slot(total 16,384) -> node```

 cluster aware redis clients generally maintain a slot-> node mapping

 if a req reaches a wrong node, the response is MOVED redirection telling the client which node owns that slot

 ```slot = CRC16(key) % 16384```

Nodes share cluster state with each other (via a gossip protocol), so every node knows the full slot map.

**Hash tags:** for multi-key operations. example - {user:1}:name and {user:1}:posts here {user:1} gets hashed 

**Capabilities of Redis**
1. Caching

cache key is redis key and values are redis values. redis distributes it across nodes of our cluster. if we need more capacity, simply add nodes to the cluster

we use TTL on each key to handle staleness.
once memory is full, redis employs cache eviction policy (LRU etc.) until then it rejects the writes

**Hot Key problem**

---
2. Distributed Lock

redis' one shared server can be reached by all the app servers and every command executes atomically. the lock is just a key that everyone agrees on.

```SET lock:concert:343 my-token NX EX 30```


---
3. Leaderboards

