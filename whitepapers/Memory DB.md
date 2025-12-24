> what it is


>why not redis

Multi-AZ Transaction Log

WAL vs WBL? how come databases can live with WAL if SPOP like ops are a risk

WBL is helpful when operatino is non-deterministic e.g. `remove a random value from set`, doing it WAL would lead that each replica removes different value, however, doing WBL would mean that we only log the low level output e..g `removed-value-x` and replicate that to keep everyone in sync. 

>Leader election? how does it work? how does it avoid stale replicas from being leader?

The transaction-log service orchestrates the election and leasing in a shorter way than ususal. Leaders are supposed to send lease-renewal request messages to this, replicas keep an eye on those messages and if not received in a given time, they contend for leadership. At that point they have to tell log-service which is the latest message they have and if log service determines thats they are not caught up, they are not allowed to content. 
NOTE: This provides guarrantees that native redis doesn't. Native redis can not ensure that 1) only consistent failover (up to date replica) is elected, and 2) no duplicate leaders. Because there is no leasing system there (? tbc) and the way leader election works is all primaries are in touch with each other using binary gossip over cluster bus port of redis. Once they detect a primary is not responding, they randomly pick a replica for that primary's shard. This is no good. 

>Recovery

Off-box snapshotting to avoid impacting the availability of data serving node as redis snapshot operation requires that a new child process is forked with Copy On Write mapping from parent(to keep data up to date while backup is ongoin) and extra buffer to hold data for backup- it typically requires twice the memory size as a result and is very CPU intensive.

> restoration

To restore the data, replicas load the snapshot from S3 and from the last point of snapshot, they replay the distributed transaction log until they are caught up, at that point they advertise via transaction log to others that they are caught up. 

