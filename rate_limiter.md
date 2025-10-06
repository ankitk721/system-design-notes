### Question
*Design a distribtued rate limiter to throttle api request per user across services*

### Dive deep areas
- Token bucket vs leaky bucket
    - Token bucket fills up tokens in a bucket at a rate and allows request while there is still some tokens left. Leaky bucket withdraws 
    at a fixed rate and if bucket capacity is reached, spill over/throttle/429 happens. Hence it makes more sense for us to go with
     TokenBucket to allow for bursty loads.
- Redis based counters
    - Use redis tokenBucket counter perKey `userId: {tokensRemaining: <>, lastInvocation:<>}`. 
    - Use Lua script to atomically
        - read current token counter
        - refill based on refill rate and time spent since last (if enough time has passed to be refilled)
        - decrement if tokens available
        - return allowed/not-allowed signal
    - Without Lua script, two servers might hit redis and execute these operations interleaved with each other and run into race conditions.
- Sliding window
    - Sliding window vs Fixed-Window/Bucketized-Time: Sliding window is going to be more strictly enforced and accurate strategy, however it would require you keep track of every request's time stamp and hence increase the amount of memory required and if we use Redis Sorted Set, it would also make it CPU-exhausiting and reduce throughput. Fixed-Window approach (i.e. simple counter) would drastically reduce the memory required and the speed of operations/throughput. We would go with latter for space and time efficiency. Even though the downtime is that within 1-time-period(minute) you could have more than required tps used by caller if they are at end of first minute and beginning of second minute.
- Fail over/Fault tolerance
    - Use redis sharding to allow for high throughput. We can shard by userId/clientId key.
    - Use consistent hashing/client-side routing based on hash-function to reach to correct redis node. 
    - Use redis replicas to fail over- tradeoff here is between ensuring durability of write vs latency of operation. If we can be lenient about replica not having fully caught up, then we can provider faster response at the risk of inaccurate throttle limits. `WAIT 1 100` command of redis ensures that `wait for 1 replica to ack your write or timeout in 100 ms`
    - If redis complete crash, use local count for failover. 
- Local vs Distributed throttling
    - Our RateLimitingServers can keep a local count of tokenBucket and repeatedly sync it from the redis server to reduce the load on redis server and improve the performance since the redis is a network hop away.

- Coarse grained vs Fine grained
    - Conversation boils down to tradeoff b/w *Memory Usage* vs *Precision*. 
    - For precision, we need to keep counter at lowest granular level (e.g. second, millisecond) and it increase the keyspace and memory size. For coarse grained, we could keep counter at a level higher to save space.
    - A good combination is to use local with high precision, since we're keeping track of last few seconds and maybe small count of keys, and redis for coarse grained to save space and throughput.
- Cleaning up stale entries:
    - why is it needed? Because we will have on entry per user/key but even if they do not access the api regulary, our lazy refill will not clean it up. If we have a service with millions of users, it will start to leak memory- to counter this, we will set the TTL for 1 hour and every time we update the token bucket, we will update the TTL so redis can natively remove these values.
- Degrade mode
    - We can fail open to let in the request when we fail to determine limit or fail-close when we run into failure. This is a tradeoff discussion.
    - Fail open would preserve CX but leave the door open for abuse.
- Token refill strategy.
    - Via a cron refill worker: More cpu utilization by having consntantly someone crawl and update tokens. 
    - Lazy refill: when needed, we refill and calculate, CPU efficient with minor latency impact if at all.
- Per-region limits