```
https://stripe.dev/blog/how-we-built-it-real-time-analytics-for-stripe-billing
```

- they developed a **real-time streaming** analytics for billing and the latency in subscription updates is less than as low as 15 mins. earlier with **batch-processing** it was around 24hr avg lag.
- they broke the problem into three main components: 1. rebuilding data archi for real time subscription updates 2. upgrade data aggregation system to reflect in real-time on dashboard 3. allow customers to adjust the metrics definitions freely without impacting real-time or historical analytics

- Low-latency analytics required an event-driven pipeline from beginning to end
- storing the subscription in a data structure is quite complicated. the business needs to know every info about the customer subscription
- the most straightforward approach would be to re-analyze all the subscription info of the user from the beginning in batches, ons a set cadence. this made real-time analytics quite impossible
- they needed to create a new pipeline that could transform the updates into events (used Apache Flink - stored highly compressed version of subscription history as a `state`).
- for old customers, replaying billions of these states in order was a challenge. for this they used Apache Spark that let run the streaming transformation logic as a data job allowing processing large amt of events in parallel and output the results as flat files. this job efficiently generated initial flink state and also feed a redundant offline data pipeline for validation and data export uses

**low-latency aggregation using a query engine**
- the earlier system used Apache Pinot as OLAP db, they preaggregated subscription data offline in a scheduled batch job but it could not quite provide real-time analytics
- they needed to remove the preaggr. step. they had to be able to analyze both historical and current states of all subscriptions maintaining the responsive queries
- Pinot released a v2 engine that could perform **windowed** aggr queries. these queries segment the data within multiple windows and then perform aggr. ops on them. the new engine also allowed more complex joins, simultaneous calculations without offline preaggr.

**customizable metric definition along with real-time updates required balancing flexibility and consistency**
- they had allowed the users the flexibility to adjust the definitions of metrics (say MRR formula) and this flexibility in streaming analytics needed a balance bw data consistency and the flexibility
- a change in any definition had to be reflected consistently across all historical and incoming data. (for a long time customer it would mean to reflect in large amount of historical data)
- the solution:
- initiate a batch process to align historical data with new definition when a customer changes any metric definition
- the new events continue to process using old definitions concurrently in real-time
- while processing these are buffered in memory in Flink application
- once historical reprocessing is done, they patch the Flink's state using recalculated historical data and allow Flink to reprocess the stored events on top of the updated history
- then they transition the dashboard to display fully updated data and stop using old metric for processing further. Throughout the dashboard stays responsive even when the customer is changing the metric definition and sees a consistent view of their data
