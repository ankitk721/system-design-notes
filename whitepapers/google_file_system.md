#### Google File System
- optimizes for sequential access for read or write at the end. Arbitrary write in middle isn't optimized because it is rare access pattern.
- use-case mostly large multi-B files, 
- large chunks, lazy allocation 
- interaction of client with Master, cacheability, pre-fetch
- replicas of chunk server
- Master Service 
    - keep things in memory - typically 64 bytes sized per chunk and can easily handle large number of files
    - maintains WAL like OperationLog for all critical operations and helps in persisting the state for replaying (to recover) and also by providing a consistent order of operation

- Data flow
    - Client asks master which chunk server has the lease for chunk c1. if no-one has it master will grant one to a replica of its choosing. Also gives location of other replicas.
    - Client caches this and sends *data* request to all replicas. Once all replicas have ack-ed, client will let primary know to *log* the write. Note that replicas when processing data request, keep all data in memory and only flush once it is asked to be used.
    - Primary assigns a sequence number for this which will drive the order of operations. 
    - Replica keeps renewing lease by piggybacking on heart-beat messages communicated b/w replica and master.