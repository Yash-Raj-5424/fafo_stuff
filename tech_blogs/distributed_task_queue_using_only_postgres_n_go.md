medium article link:
> https://medium.com/@md-talim/i-built-a-distributed-task-queue-from-scratch-with-go-and-postgresql-42b705503737

- postgres is the only queue and the source of truth - no redis, or any queue
- a task queue is simply - producers enqueue task and the workers take them out from front for execution
- a task has a type, payload and handler that knows how to execute it
- workers take tasks out of the queue and run the matching handlers. (goroutines are the workers in this project)
- **reaper** is a separate background process (a goroutine) -> takes tasks stuck on dead worker and puts them back in the queue
- the states the taks goes thru -> PENDING when created -> worker calims and it becomes RUNNING -> COMPLETED if the handler succeded -> PENDING if the handler failed or the worker died -> DEAD if ran out of attempts -> CANCELLED if u want thru the API while it's PENDING or RUNNING
- the postgres is used as the only queue and source of truth to avoid *dual write problem* like u face in a situation where u use db+queue. there'd be a need of transaction bw db and queue operation
- here the task is just the row in the same db as the business data then inserting task is a part of the same transaction as the business query - either both commit or neither does (traxnal outbox)
- the tradeoff is that postgres won't match Kafka or RMQ's throughput. correctness is given more importance here.
- **Atomic taks claiming** with `FOR UPDATE SKIP LOCKED` (how do we ensure a task isn't claimed concurrently by multiple workers)
- **Pessimistic Locking** given by `FOR UPDATE` locks the selected rows or blocks until the lock is released in case other trxn already acquired it
- `SKIP LOCKED` skip the rows that are already locked instead of waiting
> queue like tables are the use case for this

**workers**
- try to claim a task -> if found execute it and try to claim the next one -> if none found wait for poll interval and try again (it's what is used in this project)
- the worker keeps polling the db
- for retries, expo-backoff with full jitters is used and when attempts get exhausted it goes to DEAD state and can be manually retried through the API
**heartbeats and the reaper**
- what if the worker died mid task - no error task remains RUNNING forever
- a background goroutine periodically sets locked_at and updated_at to now() (sending heartbeats) -> if the worker dies, hearbeat stops -> then the reaper looks for RUNNING tasks that haven't had a heartbeat within the stuck threshold, puts those tasks back to PENDING or moves them to DEAD if attempts are exhausted. (reaper keeps scaning on an interval)
- the tasks are **at-least-once** executed i.e, idempotency keys are added at the enqueue time to keep the handlers idempotent
- on `SIGINT` or `SIGTERM`, workers stop claiming new tasks and give the in-flight tasks a configurable amt of time to finish. after that the conxn pool is drained. without this, every deploy would kill running tasks mid-way and leave the reaper to clean up the mess.

**Benchmarking**
- the project had 20 workers each say finishes a task in ~125ms on avg => 160task/sec was expected but measured result was ~20tps 
- earlier the worker only attempted one claim per poll tick no matter how quickly the task finished so 20 workers x 1 claim per sec = 20 tps. this meant if the worker finised a task in say 100 ms it would sit idle for the rest of 900ms.
- **the FIX:** keep claiming continuously while the work is available and fallback to the poll interval only when the queue is empty. after the fix : ~148tps

> the project didn't only use HTTP APIs but also a client library
- he wanted the task insert to be atomic with the business writes. with an HTTP API, that becomes impossible. HOW ?
- application writes the order in its traxn, then makes an HTTP call which inserts the task in another traxn on another connxn -> there's no way to put both under one commit -> same *dual-write problem*
- the key function of the client library is `EnqueueTx` -> begin trxn, business logic, insert to db, enqueue, commit => all stay atomic
- idempotent enqueue without poisoning traxn. enqueue uses INSERT... ON CONFLICT DO NOTHING with an idempotency key. in postgres, a failed statement aborts the "entire" trxn so if a duplicate insert raised an error, it would take down the caller's whole traxn. instead a replay quietly returns the existing task with duplicate:true.
- the HTTP server and teh worker binaries are now thin wrappers built form the library. they can't drift from it.