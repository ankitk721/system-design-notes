
- **Unidirectional Logic:** SSE is application-layer unidirectional (Server → Client), requiring a separate API endpoint for incoming bids while the stream handles real-time updates.
    
- **Transport Layer Reality:** Despite application-layer directionality, the underlying TCP connection is bidirectional, relying on low-level ACKs to maintain the link.
    
- **The Load Balancer (LB) Role:** Sits between the user and SSE server, managing two separate TCP legs. It detects client loss and "bubbles" the error up to the SSE server.
    
- **Non-Graceful Disconnects:** If a user loses signal (no `FIN` packet), the system relies on TCP retransmission timeouts. The lack of ACKs eventually triggers an `EPIPE` or `ECONNRESET` on the server.
    
- **Heartbeat Necessity:** You correctly identified that without a client-backchannel, the server must send periodic dummy data (heartbeats) to prevent LB idle-timeouts and force TCP ACK checks.
    
- **Scaling & Reconnection:** Since users may reconnect to a _different_ SSE server (stateless LB), the architecture uses a Redis-backed history buffer.
    
- **Catch-up Mechanism:** Clients send a `Last-Event-ID`. The new SSE server queries the Redis buffer for missed IDs to ensure zero data loss during handovers.
    
- **Connection Registry:** SSE servers maintain an in-memory map of `userId` to `connectionId` for fast routing of messages received from Redis Pub/Sub.
    
- **Backpressure Handling:** If a client is too slow for the bid volume, the server must choose to either drop intermediate updates or disconnect the stale client to protect system resources.