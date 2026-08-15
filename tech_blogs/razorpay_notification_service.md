Razorpay's notification service handling the sms, emails, webhooks was working reliably.

but with exponential incr in tranxns challanged the webhooks (web apps send info to other apps in real time when a specific event happens)

the existing flow was as follows:

```
req --> api pods (validate) --> SQS <-- worker (reads) and notif is sent
  |  
  V
 write to db -> pushed to datalake

scheduler(read retry reqs from db) --> push to SQS for processing
```

> this could already handle 2K TPS but based on load the perf starts degrading as p99 reaches ~4secs

# challanges faced on scaling up
1. reads degraded with incr in data and with incr in writes even replica doesn't help much
2. with limited IOPS even the worker pods couldn't be scaled much (tried increasing the db size 2x, 4x, 8x etc.. but wasn't a long term soln)
3. ops couldn't be completed within SLAs as db couldn't be scaled infinitely
4. on special events, load incr unexpectedly eg. festivals, IPL, World Cup etc.

# steps took:
* prioritize incoming load
* reducing db bottleneck
* managing SLAs with unexpected res times from customer's servers
* reducing detection and resolution times

**1. Prioritizing incoming load**

- not all notif reqs were eqaully critical & one type shouldn't impact other
- created priority queues for notifs based on impact & SLAs and pushed accordingly (critical, default, burst events with high TPS)
- introduced Rate Limiter to queues to prevent SLA breaches (api -> rate limiter -> q1, q2, rate limit queue)
- rate limits could be customized and event prioritizn with rate limiting prevents DOS attacks and helped to build a multi-tenant system

**2. Reducing DB bottleneck**
- earlier they were auto-scaling the worker pods with traffic incr but it slowed the overall SLA as IOPS increased.
- they needed a horizontal scalable db or archi change to handle increasing load
- fix was async. writing data to DB
- introduced a stream b/w DB writes and workers. msg -> stream (idempotent) -> writer writes to DB

**3. Managing SLAs with unexpected res times from customer's servers**
- QOS for customers - decr the priority of a customer if res from their server take more and reset if responds well later
- api --> QOS passed ? --> queue/QOS queue based on yes/no

**4. Reducing detection and resolution time**
- worked on observability - 
- 1. grafana dashboards for anomaly detection, alerts on rogue events, rate-limited events, high response time customers, success rate
- 2. dashboards on logs for error patterns
- 3. distributed tracing to understand various components of the system

**Article Link**
> https://engineering.razorpay.com/how-razorpays-notification-service-handles-increasing-load-f787623a490f

