#### What is linearizability
The property of your database that makes it behave as if its a single register thats processing the operations. Effectively in a natural timeline if a write has been processed, then any subsequent read should always reflect that write (regardless of how distributed your system is). It is strongest consistency guarranttee in distributed databases.

1. How is different from Serializability isolation level?
- Isolation level is a property of transactions. Serializable being that they are processed in a manner whereby they are appearing in a serial manner. Linearizability is property of individual item/record, that once it has been updated, subsequent reads see the updated value.

2. Where is it used?
- Primary applications are where you need this kind of strong consistency of reflecting updates immediately with no staleness such as leader election. Zookeeper is a linearizable storage.

3. How can we typically build it?
- Since we can't have a single host to fault tolerance and we can't have single leader with all writes routed to leader because it might get network partitioned and not know about it and we might end up with split brian where two leaders are accepting writes and our linearizability guarrantees are out of the window. Hence a distributed consensus works best here to pick leader and then leader commits with quorum like setup of raft. That ensures a total order of operation which is honored by everyone and is resilient to leader crashes too.

4. Why is quorum not sufficient to ensure this?
- Quorum means that number of replicas to consult for read is r and write w then r + w > n. And when reading you fetch data from all `r` replicas and then you could still end up in senario where if 5 total replicas, 
   ```
   read 1 : R1 [has v2], R2 [has v1] -> result v2 is latest
   read 2:  R2 [still has v1], R3 [has v1]  -> result v1 is latest
   ```
   This is because your both reads get in the middle of when write is ongoing. So quorum on its own is not sufficient for this guarranttee.

5. How does spanner ensure linearizability
- It forces each follower to wait until it has received Paxos log representing operations until the user's request timestamp. To ensure that it does not drop messages in flight- there is also a likely option where if its not sufficiently up to date, follower will be redirect query to leader.

6. Write throughput for a strongly consistent/linearizable system like Spanner?
- Note since each shard needs to ensure total order of operations, it can NOT have parallel writing of records, effectively paxos-consensus call/log becomes the choke point. Since each which will further be constrained by how much network roundtrip we have. 

in this case lets assume 
`leader : IAD` and consensus requires that quorum across `IAD+ PDX`, per shard throughput is
```
| RTT to quorum               | Max write throughput |
| --------------------------- | -------------------- |
| 160 ms + 50m                | 5 writes/sec         | e.g. Tokyo to IAD(leader) for write request, then IAD to PDX to building quorum
| 50 ms                       | 20 writes/sec        | e.g. IAD(leader) to PDX
| 30 ms                       | 33 writes/sec        | 
| 10 ms (co-located replicas) | 100+ writes/sec      |
```

If we can have 1000 shards, out throughput will be 1000 x <number from above>


6. Read throughput
- note: reads do not go through consensus. read-only replica of spanner can serve snapshot read at very high throughput (10k-100k+).
- strong reads are bottlenecked by leader's read handling capacity 100s to 1k.
- snapshot reads can be very high throughput (give me *likely an old* timestamp's version, no need to consult anyone else, highly cacheable)
- stale reads: throughput not as high as snapshot read (give me the *latest* version for it). Replica might have to check thats its caught up and maybe redirect to leader even which will hurt throughput.