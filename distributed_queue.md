### Basics

#### Segments
Broker will store messages in an append only immutable log on disk with offset position. Broker will name the segment log/data files using minimum messge-offset e.g. `0000100.log`. This will enable broker to find out which log file will have the offset query by consumer on read path. There will be `0000100.index` and `00001000.timestamp` file within the same segment too which will have sparse mapping of Offsets/Timestamps and their corrresponding byte-location. This will enable Broker to provide a quick seek to an offset or timestamp.

#### Broker discovery
An initial list of servers is supplied as seed to all producer and consumer, this list will be ensure to be of healthy hosts and updated based on host status. Producer/Consumer will talk to these nodes to get metadata about partitions of a topic and leaders of those partitions- after that they will connect to the actual broers they need to.

#### Co-ordinator
Co-ordinator is the node which performs the role of picking a leader consumer who can do partition assignment among consumer groups and co-ordinator also keeps tab on status of consumers by doing health checks and if any consumer dies, it triggers a consumer rebalancing by sharing the information about consumer level commited offsets

#### Replication


### Dive Deep
- Fault tolerance
    - Broker died
        - Any other InSyncReplica (replica who is known to be synced upto a threshold) is promoted to be new leader.
    - Producer died
        - Producer retries from the sequence number where it was and publishes messages with its producerId and sequence number.
    - Consumer died
        - A new consumer is spun up for replacement who starts from the commited offset of dead consumer.
- Tradeoff b/w high throughput vs message durability
    - We will provide a configuration to tune the durability by setting up how many replicas of a partition should confirm the ingestion before producer is sent an ack back. `acks=zero/one/all` would respectively provide `fire and forget/ atleast let leader save it/ atleast <min-ISR> replicas to save it` before confirming to producer that message is saved. Higher the count, higher the latency.
    - We also provide `min.insync.replicas=2` to drive how many replicas should save it before sending producer back.
    - Note that this means even if leader has messages with offset upto 100, it might only consider upto 90 as commited as rest are yet to be syned/fetched by replicas. This is called High-watermark/HW and is used in consumer read path where leader only vends till HW offset and not after that.
- Idempotent processing
    - This is to ensure exactly once delivery (note: its different from exactly once processing which would require that even consumer have some sort of support for transaction to atomically commit offset and process message or rollback the whole thing). For exactly once delivery, Kafka lets producer pick the seqence number for a message and broker would dedupe on `producerId + seqNumber`.
- Consumer rebalancing
- Role of co-ordinator broker
- Leader election/ consensus for Partition, Co-ordinator
- How to efficiently fetch message for given offset
    - Build a store `.index` file along with `.log` data file when saving segment to disk. Index file is a sparse file containing `offset: diskByteLocation` and use binary search to arrive to the disk block containing such location.
    - `mmap`: Use Memory Mapping for both Index and Data file. By doing this OS maps the file into viirtual memory enabling our broker to read/write file like a byte array and avoiding sys-calls like `read()` and `seek()`. OS will cache hot parts in its cache.
    - This enables Zero Copy I/O reads.
    - Even if file is in GBs, only accessed parts are paged in.
- How to avoid replicas from bombarding the primary for resource fetch to stay in sync
    - replicas to `fetch` for a batch of messages and provide offset, primary uses this offset to build internal awareness of which replica is lagging behind by how much
    - a bach retrieval like this will substantially reduce the TPS b/w primary and replica
- How to achive high throughput on writes
    - Instead of writing every single message to disk (i.e. flushing/`fsync`-ing), we let producer write to the files (`.index`, `.log`) using `write()` on file-descriptors, this will land the content in OS cache and OS will flush it to disk in batch when it feels like it- based on memory pressure, time-spent etc. By doing this we avoid `fsync()` which would have blocked the call until data hits disk platter or SSD. 

### Appendix

| `acks` Value | Producer Waits for...           | Is it Blocked Until Commit?     |
| ------------ | ------------------------------- | ------------------------------- |
| `acks=0`     | Nothing (fire and forget)       | No                              |
| `acks=1`     | Only leader's write             | No (doesn’t wait for replicas)  |
| `acks=all`   | Leader + *all* in-sync replicas | **Yes** (waits until committed) |


