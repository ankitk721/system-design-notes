### Dive deep areas
1. How would you serve the leaderboard in a fast manner?
2. How much tps/volume can you handle in this? 
3. How would you shard?
4. How would you handle fast retieval of friends score?
5. How would you support realtime persistent rendering of this on game UI?
6. How would you handle network glitch to redis call before being acked back- would you retry and double count?
	1. Since we use `ZADDSCORE` sorted set, we run this risk. There are two ways to avoid it
		1. Use some idempotency key attached to each increment operation and in redis use a Lua script to check idempotency key existence and score update atomically. e.g. `idempotencykey: userId_competitionId_time`. We could put all the keys in a single Set with TTL of 24 hours. This would mean that we need to handle complications/risks of coming up with this id.
		2. Use CDC (preferred). Use a DB as main source of truth for "total-score" and just keep reflecting that into redis, instead of redis being the source of truth. We use Cassandra for LSM-tree based writes which converts random writes into into sequential disk IO and use a Kafka in front of it for buffer. In Cassandra, the way to do it is, a [[*Debezium*]] connector utility reads commit logs of Cassandra nodes and pushes them onto a Kafka, from there we can have another kafka consumer which pushes to Redis.
7. How would you ensure fault tolerance?
	1. We use Kafka with minInSync replicas >1 and use redis with replicas. Kafka provides replaybility when restoring redis. 
8. How would you handle this for global customer base?
9. 