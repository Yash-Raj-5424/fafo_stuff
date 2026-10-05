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

**Credit Card Authorization Flow**
- router receives req and applies routing (deter/priority) based on use-case
- validation, enrichment, transformation, issuer determination is done inside the cell and trxn is returned to router
- router sends req to appropriate card issuer for auth and response arrives back to it
- router uses determ routing to reach the cell holding the context of trxn for auth and cell validates the response
- trxn travels back to merchant's acquiring bank and final confirmation is sent back to POS terminal

**Mid-Transaction Failure**
- pymt processing uses an orchestrated microservices archi and it monitors health n detects failures
- this is how: detect failure and halt processing -> send trxn back to router -> router selects a healthy cell -> processing restarts in that cell using original trxn data
- trxn aren't resumed on failure rather restarted, as resuming would require old state info and it would require communication bw cells to share the state info of the trxn (here they avoid sync problems and consistency risk during failovers)
- cells stay loosely coupled. each cell run in its own db cluster, microservices inside cells commu with local cluster only and a rerouted trxn is processed with zero reliance on state from prev cell. when health degrades over a threshold, the orchestrator reports the cell as unavailable
- restarting is a minimal tradeoff as pymt is short againsst the dependency bw every cell (if we resumed the pymt instead)

**Recovering Semantics**
- **point of no return(PONR)** - exact bdry in a trxn lifecycle where system recovery emantics shift from idempotent reversals/repeats to permanent adjustments
- where to keep the PONR is a design decision. Eg: in card auth, PONR is kept as late as the paymt flow allows making the recoverable window as wide as possible
- trxn carry unique identifier across every retry and reroute and it is used by downstream services to avoid duplicates.
- they avoid bouncing everything back the moment a cell returns and shifting the traffic lets them drain the cell gradually.
- But how to prevent recovered cell from writing stale state after its trxn moved elsewhere
- Three mechanisms: **speed** - pymts are fast so trxn usually completed elsewhere by the time a failed cell returns. **idempotency identifiers** - duplicates avoided downstream. **paced recovery** - percentage-based traffic control governs when and how much work a recovering cell receives

**Design Tradeoffs**
1. **Duplicated services** 
- preserves cell independence and removes cross-cell network hops
2. **Dropped log records**
- *buffer truncation* - might lose some log records under load as those aren't critical trxnal events but just application logs
3. **Delayed global visibility**
- global observability always lags degrading visibility of one cell instead of whole platform
4. **Rejected transactions**
- consistency requirements depend on trxn type and business rule. when the trxn requires strong consistency, but the required data cannot be validated or turns to be inconsistent, they may reject the trxn to preserve data integrity.

> The cells change how many fail at once rather than how often failures happen since running more independent units produces more individual failures. (mgmt overhead and complexity are traded off for reduced impact of failure)

**Conclusion**
- two data strategies (rarely changing data pushed to every cell earlier and constantly changing one stays and trxn travels to it)
- router carries only routing logic no business logic making it dependable
- restart on failure instead of resume (avoid state mgmt of trxn bw cells)
- NOPR is kept at last for wide recovery window
- recovery semantics - business logic provides rules for partially completed trxn
