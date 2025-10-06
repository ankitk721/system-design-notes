### Question
*Design a system which supports retrieving top K most viewed videos of youtube in any given time window*

### Follow up
 - Sliding window or arbitrary
 - Do we need to keep accurate granular information from past? We can apply tier-storage and keep moving those to s3 or glacier otherwise to save cost.
- How much latency we're talking on query front?
- Can we live with some error at the cost of frugal infra?
- How much delay before events show up in results (within 5 seconds/ realtime or not)

### Basics
- Use distributed queue like Kafka to intake the video view events for fault tolerance and scalability.
- On the consumer side of this queue, create aggregations at minute level and create minHeap to contain top 1000 entries. Repeatedly flush the minHeap onto a durable store. This is the raw count approach.
- On the query time aggregate the topK values across the minutes (where entry is `videoId:minute:count`) and pick merged winners. This is lossy aggregation but mostly okay for our purposes since we only care about winners and we're trading off with operating on 100s of millions of videos on the fly.

### Dive deep areas
- How would you handle large time windows?
    - Preaggregate larger windows and store their results. Either by Map Reduce or have a worker spinning off every x minutes (spins up every 5 minutes to aggregate winners for now to now-60 minutes by aggregating stuff in background and svaing them at hourly level granularity)
    - Downsample larger windows: 5 minute level at Day level, hourly level at month level. 
    - Query planner picks appropriate tables to construct the query.
- How would you make it space efficient if we had 100s of billions of videos
    - Use space efficient sketch (`count-min` sketch or `top k` sketch)
    - Tradeoff b/w memory and accuracy/error-rate: More functions or wider range of value for function would mean more memory, less collision.
- How would map reduce do its job here?
    - Dump 1 minute aggreations into S3
    - Maybe no map reduce in redshift so you use Spark where each node will map original events to a key `adId_hour:<count>` and then share those with key based hash-partitioned reducers who get full view of respective keys from multiple mappers. Eventually reducer will export to S3 `s3://bucket/ad_clicks_hourly/dt=2025-05-25/hour=13/part-0000*.parquet`. There is network overhead, spark drivers involved which probably control some of the co-ordination issues.
- How would you handle super popular video and its hot-spots
    - Pre-aggregate at early stages in lifecycle like at CDN etc. or extend pipeline to have two stage aggregation
    - If needed, shard based on `videoId+minute` so different minutes of same video will go different shards.
- Do you garbage collect Sketches if you're using them?
    - Yes, always only keep upto last 60 sketches of 1 minute each and at every minute insert a new one and evict the oldest one. Use time based TTL ring buffer pattern.


