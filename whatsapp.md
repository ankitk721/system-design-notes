### Question
*Design a scalable messaging system where users can send 1-1 and group messages in realtime.*

### Basics
- Users connect via Edge/Chat server using websocket connnection.
- Users send and receive messages and acks on this connection.
- Backend messaging server will lookup its own in-memory subscription and centeralized globally replicated redis session-manager to find whereabouts of the other user and which region its connected to.
- Backend messaging server will send message to the message server load balancer of other resion via internal link.
- Other region will find the actual server connected to the user and deliver the message on that port using internal subscription of `userId:connectionId`
- A kafka queue, sharded by chatId, will also be populated with message whose consumer will write to dynamoDb global table's local instance.
- Ack will be sent back and persisted in the DB.
- If user is fouud to be offline at any point, they will be notified by APN or Firebase.
- Presence DB will be populated using `ping/pong` heartbeat events sent by the client and over a data points of 2-3 intervals (~1 minute).

### Dive deep areas
1. Connection model: Long polling vs Websockets
    - Long poll would have higher latency due to creating a new request all the way and incuring a ronud-trip delay for new message. Websocket would be instant.
    - Websocket would waste resources in header transfer repeatedly, Websocket will be efficient.

2. Delivery guarrantee: Receipts, deduplication
3. Message ordering
    - Use snowflake based timestamp. `msg_id = epoch_ms << 22 | machine_id << 12 | sequence_counter`
    - Can be atomic counter at machine level if messages are very close to each other.
    - Machine id can be somehting assigned at the startup by zookeeper- ephemeral node for each machine and machine dying removes the ephemercal node. Each assignment goes through ZK leader hence ensureing linearizability and sequentialy and no-race-condition/double-booking.

4. User presence
    - A redis entry with TTL of 60 seconds which can updated every time we hear ping/pong from client `EXPIRE presence:u1 60`
5. Chat history storage
    - Use DynamoDb global table for low latency and durability. 
    - Conflict resolution logic- DynamoDb uses last write wins. Do we need to use vector clock or this suffices?
    - Note that since DynamoDb Global Table doesn't provide strong consistency across regions, if we need strong consistency, we will need to route all writes through a single region which will increase latency. 
6. Cross continent chats
    - Use PoP/Cloudfront/Akamai for instant websocket connection setup : Android/iOS will connect to `ws://chat.whatsapp.com` using technique of GeoDNS where by DNS server will respond with the IP address which is close to the user and will vary even for same domain-name. This will help reduce latency of connection. 
    - PoP will look up user's session information (which region and which server they are connected to) using globally-replicated Redis/DynamoDb. Present/session is volatile - we don't need strong consistency , just fast and decent replication would do.
7. Scaling: Sharding chat rooms or user-connections
    - Shard by chatId (`hash(min(u1, u2) + max(u1, u2))`) to help in scaling as well as lower latency- all messages for a chat would be co-located and range queries would be quick for finding last-N messages of a chat.
    
8. How to detect disconnection/pop-host dying
    - TCP RST/ TCP based disconnection (Passive): If server has graceful shutdown, it might send `FIN`-> client gets a clean reconnection. But if server dies abrupty, no packet is sent. TCP only knows its dead when it tries to send something but doesn't get any Ack back. or TCP timout triggers which could take as long as minutes depending on client's OS. Because kernel is what actually drives TCP packet communication and if it misses acks, it can retry by exponentially backing off by `3s, 5s, 10s... often upto minutes` before giving up. Hence TCP by iteself can be slow to detect a slow peer. 
    - Ping Pong mechanism (Active Disconnection): Every 10 seconds send a `ping`. If no `pong` received in 30 seconds, consider it dead and reconnect. When reconnecting GeoDns will route to PoP host which is healthy. Request for missing messages to be loaded by making pagination like query with last-seen-message.

9. Use PoP/Edge-server local to user to better connection reliability
    - Without PoP, sender's initial connection and TCP handshake, TLS has to happen over a network with potentially 1000s of KMs away. 
    - High latency and increase chance of packet drops over longer path- with PoP, backend takes the responsibility of handling packet losses instead of users and users see a snappy experience.