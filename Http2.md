
### Http/1.1

Http/1.1 has support for multiple requests on a single TCP connections and "pipelining" them but it suffers from a major issue of Head of the Line Blocking (HOL) because of which a slow response will force everyone else to also wait before getting delivered back to client. This is because it operates at L7 level blocking and does't have an inherent way to identify and distinguish response themseleves to link them back to request- so effectively protocol just forces all requests and response to go in same order otherwise there is no way to assemble which response was for which request.

### Http/2

HTTP/2 breaks down requests and responses into smaller, numbered **frames**. These frames from different requests (called **streams**) are interleaved on the same single TCP connection.


## Head-of-Line (HOL) Blocking Summary

- **HTTP/1.1 Failure (Application-Layer HOL):** HTTP/1.1 Pipelining required the server to deliver responses in the exact **order of request**. A slow response (Request 2) would block all subsequent, ready responses (Request 3) on the single TCP connection.
    
- **HTTP/2 Solution (Multiplexing):** HTTP/2 solved this L7 HOL blocking by using **Stream IDs**. Requests/responses are broken into frames, allowing responses to be **interleaved** and delivered out of order.
    
- **Slow Process Isolation:** In HTTP/2, a response that is simply **slow to process** at the server is isolated; it does not block the delivery of frames from other faster streams.
    
- **The Unsolved Problem (Transport-Layer HOL):** HTTP/2 still suffers from **TCP HOL Blocking**. If a **single data packet** is lost, the entire TCP connection stalls until the packet is retransmitted because TCP guarantees in-order delivery.
    
- **Scope of L4 HOL:** This stall affects **all concurrent streams** on the connection, regardless of which stream the lost packet belongs to.
    
- **HTTP/3 Context:** This final bottleneck is fixed in HTTP/3, which uses **QUIC** (built on UDP) to enable independent, per-stream loss recovery.

