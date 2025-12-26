### Dive deep areas
1. How would you support realtime persistent rendering of this on game UI?
2. How would you handle network glitch to redis call before being acked back- would you retry and double count?
	1. Since we use `ZADDSCORE` sorted set, we run this risk. There are two ways to avoid it
		1. Use some idempotency key attached to each increment operation and in redis use a Lua script to check idempotency key existence and score update atomically. e.g. `idempotencykey: userId_competitionId_time`. We could put all the keys in a single Set with TTL of 24 hours. This would mean that we need to handle complications/risks of coming up with this id.
		2. Use CDC (preferred). Use a DB as main source of truth for "total-score" and just keep reflecting that into redis, instead of redis being the source of truth. We use Cassandra for LSM-tree based writes which converts random writes into into sequential disk IO and use a Kafka in front of it for buffer. In Cassandra, the way to do it is, a [[*Debezium*]] connector utility reads commit logs of Cassandra nodes and pushes them onto a Kafka, from there we can have another kafka consumer which pushes to Redis.
3. How would you ensure fault tolerance?
	1. We use Kafka with minInSync replicas >1 and use redis with replicas. Kafka provides replaybility when restoring redis. 
	2. To prevent write-loss during a failover, I would configure the Kafka consumer to use the Redis `WAIT` command to ensure the update reaches a replica before committing the Kafka offset (remember that redis by default does asynchornous pull based replication and waits for replicas to ask for updates every xx seconds- which is how long it will take before it realizes its either not leader or its partitioned etc. despite having given the ack back to caller). However, since we are using CDC to stream the **total score**, the system is naturally self-healing; even if a single update is lost during a failover, the subsequent state-update for that user will correct the leaderboard idempotently. Even if some users do not play again for this self-healing to re-trigger, I can have a background job to daily reconcile to 10000 users.
4. How would you handle this for global customer base?
5. Scaling
	1. Throughput
		1. To handle **1M writes/sec**, you need at least **10–12 shards** just for the CPU capacity, regardless of memory size. (More in redis section)
6. Sharding
	1. Kafka : UserId based? Because we need a user's score to be updated in order in redis so their score dont accidentally end up going out of order and ending up in stale state.
	2. Redis: To avoid scatter, gather, we should bucketzie the redis whereby we treat score based segements as differetn redis shards e.g. 1-1000 points in shard1, 1001-2000 in shard2...and so on.This would ensure that we can quickly find adjacent ranked users. We can still find the rank easily by knowing about the count of each bucket ahead of us and position of user in lcoal bucket. This info can be stored in a small Hashmap in redis. 
		1. Problem with it: Some buckets might get too hot since we cant control score distribution of users, to handle this, we will do a dynamic resharding whenever our monitoring system oberves that count of users or cpu is going above a threshold. In resharding, we will find a split point and create two new shards from that split, going forward any new writes will go to those new shards. We can hence move the active users as a read-repair-like mechanism and then have a background worker to shift idle users into new shard.
7. How to handle nearby friends positions without a huge scatter gather?
	1. precompute the feed for each of the friends when a user updates. Cache friendsships also in redis to avoid hammeing DB for every score update. 

[ USER CLIENT ] 
      │
      ▼
[ API GATEWAY ] ──▶ [ KAFKA (Ingestion Topic) ]
                          │
                          ▼
                    [ DB WORKER ] ──▶ [ CASSANDRA/SCYLLA DB ]
                                            │
                                            ▼
                                    [ CDC CONNECTOR ]
                                     (e.g. Debezium)
                                            │
                                            ▼
[ REDIS WORKER ] ◀── [ KAFKA (Change Event Topic) ]
      │
      ├─▶ [ GLOBAL REDIS SHARDS ] (Rankings)
      └─▶ [ SOCIAL REDIS SHARDS ] (Friend Lists)