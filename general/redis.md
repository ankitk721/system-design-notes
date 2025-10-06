#### General

#### Geo-spatial
- Uses 52 bit interleaving based geohash of lat/long to come up with *score* for each location/point and uses it with SortedSet to provide efficient lookups for range queries.
- Off the shelf solutino for GEOADD, GEORADIUS, GEOSEARCH. Uses bounding box + post-filtering to pick matching items withing given radius.
- For estimatation consider 100 bytes per item (16 bytes UUID, location, skip-list, hashmap overhead, pointers etc)
- Can handle large volumne/frequeny of updates compared to Quad-tree (one of the main decision makers)

##### Challenges
- Hot Spot
  - Due to uneven data distribution, hot spotting can occur. e.g. NYC will have huge number of entries and all of them will be in a narrow range in sorted set. We can't shard effectively based on geohash to overcome this because the GeoHash will be very similar and will end up putting those into same instance.
  - What if we partition data altogether using high level prefix (state level)-> still the same problem because popular states/cities will be heavily loaded. 
  - Summary is that Redis-GeoHash isn't space filling uniformaly.

- Cross-shard boundaries
  - for users at the border of a shard(e.g. state), you'll need to query bordering shards too which can kill the low latency guarrantee.

- No auto-adaptive sharding strategy
  - sharding happens at application level- by geohash prefix, city, region etc. Doesn't auto-split dense regions or compact sparse ones.

- Memory Bound
    - Redis is memory bound, quad tree (backed by disk based system like Postgres, RocksDb) can scale beyond memory.