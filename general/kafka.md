1. How does kafka keep track of min-ISR?
	-  **The Pull Model:** Followers constantly send `FetchRequest` calls to the Leader (the same way a Consumer does).
		-**The Tracking:** The Leader tracks the **High Watermark** (the last offset the followers have successfully copied).
	- **The Eviction:** If a Follower fails to request data or fails to catch up to the Leader's latest offset within that 30-second window, the Leader "evicts" it from the ISR list.
	- **The Notification:** The Leader updates the metadata (in Zookeeper or KRaft) so that Producers know the ISR list has shrunk.
2. What happens if producer says min-ISR =2 but leader finds out no replicas in sync at some point?
	1. It refuses the write and sends back a `NotEnoughReplicasException`.
3. Why Kafka's pull based replication is better than if it were push?
	1. It helps throughput- without pull, leader would have to be responsible for pushing data out to even slow replicas and wait for slow one to accept its response, this would block leader's thread. Now it does't care, it only records a metric for lag of the replicas. 
4. What is high-watermark? is it same as offset?
	1. Related but not same. It is actually same as *committed-offset*. Think of it as "safe-to-read line" in logs. This is the largest offset that has been successfully copied to **all** replicas in the In-Sync Replica (ISR) set.
	2. We need this to avoid the *flicker-problem*
		1. It prevents the bug of non-repeatable-reads. Because if we didnt have this, a consumer could read a message which hasn't been stored durably to enough replicas and we have crash and see the offset going back in time and losing that message. By using the High Watermark, Kafka ensures that a consumer only sees data that is **guaranteed to survive** a leader failover.