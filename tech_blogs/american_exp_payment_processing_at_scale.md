# Built for Reliability: How American Express Processes Payments at Scale

- when a trxn arrives, it is routed to one of several independent processing units and moves through a chain of microservices. can't keep retrying or move the trxn to another server.
- they handle this with a **cell-based archi**
- **core payments ecosystem**

```
card terminal (payment req) => acquiring back => Amex (cells+global trxn router) => card issuer (holds account balance) => approval or decline travels back same way to card terminal
```

- they began modernizing this platform to cloud-native infra. so failures were expected often. two patterns possible: real time response was required and at scale, so event-driven processing didn't fit well
- monolith could be better but scaling would be poor

- In **cell-based archi** a cell is a complete, self-sufficient copy of the payment processing satck (microservices, db, DNS all in one boundary)
```
- deploys independently and processes payments on its own
- owns everything
- forms a single failure domain
- easy maintenance with rest of the platform carrying on
- no sync cross-cell dependencies in critical path
```

- Reference data replicates across cells, observability data aggregate across cells, and router instances communicate with each other across cells. no blocking call bw cells during trxn processing

```
client => cell router => cells(with everything)
```

> cells divide a system by failure. one cell can contain many microservices

**data locality**
- the required data should reside inside a cell
- immutable data is setup once n stays fixed
- semi-static data changes time to time (exchange rates, merchant category codes, country codes etc)
- dynamic data changes with every traxn
- alternative is a fall-through read - a lookup that misses the local cache and travels to a central system of record while trnxn waits. 
- pushing data ahead benefits: 1st trxn avoids paying for a cold cache, critical path avoids sync calls, replication work runs outside trxn path
> **they use push and distribute rather than pull then cache** i.e, central store pushes the data to the cells even before it is needed
- for **dynamic data**, an async replication may leave stale data in a cell. so they move trxn to the data itself (instead of data to trxn).
- **deterministic routing (one of 2 modes)** - decisions follows from the trxn's own content rather than from cell load or availability(partner, market, paymt type). this routing is selective and other traxn with less data requiremnts can be routed freely

> **Global Transaction Router** makes the decision at the front door

- router also supports priority-based routing where cells carry a specific ordering. (traffic => highest priority healthy cell)
- msg-based replication conitnues throughout so failover data exists in more than one place. **every in-flight trxn proceeds without waiting for replication to finish**

**Global Transaction Router**
- routes traffic and enforces the cell bndry
- trxn enter to a cell thru it and moves to other cell thru it as well (it's single commu path bw cells - so cells are connected globally but remain independent)
- this might be the **single point of failure**. how to mitigate it ? this is how:
- the router just knows msg parsing and routing logic - the business logic stays outside
- minimal dependencies, router state lives in non-persistent stores (so, stateless)
- other deps run async like async logger with buffer truncation (no trxn blocking)
- configs are loaded in memory and updated async
- multiple instances, multiple regions, multiple connxn - run in parallel across regions
- they avoid active standby as there's no state to manage for routers