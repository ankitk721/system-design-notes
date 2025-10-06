### Dive deep
1. How will you enable fast timeline rendering at scale?
2. How will you handle scatter gather latency if user is following celebrities?
3. How would you shard the systems?
4. How would you provide large video file uploads?
5. How would you protect latency from suffering due to buffering of large videos?
6. How would you handle loading 5 thousands followers for a user?
7. Would you use pull CDN vs push CDN? What is the caching strategy here? what is the key? How do you compare?
8. How do you ensure not losing the data?
9. How would you handle invalidations? pub/sub (redis), versioning, event driven
10. Cache strategies: Write-through, Write-around, Write-behind.
11. Eviction policites: LRU, LFU, TTL, Customer scoring
12. Consistency tradeoffs: strong vs evantual
13. Cache-stampede-prevention: Lock, early refresh, thundering herd
14. Multi-level cache: CDN Edge + Redis + In-process

### Design

![](diagrams-v2/facebook-feed.png)
