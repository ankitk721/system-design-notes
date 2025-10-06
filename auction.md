### Problem
Design a system to let users creation auction and have other users place bid on them. Ensure strong consistency, track record of bids placed even if lost.

### Dive deep areas
- High frequency bids without blocking users
    - Doing it via database would require that we put lock on the auction table or all the rows of Bid table for that auction and introduce a chokepoints to process them sequentially however it will drastically slow down our through put. 
    - We will go with a redis based option- redis will be speed layer and DB will be persistence layer. Redis will use atomic Lua scripts to CompareAndSet the value for bids in an auction. We will put the output after redis processing into Kafka of format `bidId, user, price, accepted?, auctionId`. 
    - What if the BidService (stateless redis caller) crashes before pushing to Kafka: We will use another Kafka queue in front of it so that requests can be replayed- and another instance will reach to same state by replaying the events from kafka.
- Auction closing
    - Save information about last-bid-of-auction in redis and have ttl-based-notification of redis to trigger closing worker.
    - Or crawl at some interval to see if open auction has had the last bid time + delay expired or not. You can potentially add a check in Bid-Service to not let bid pass if last-bid-time + diff > maxAllowedDiff and consider it closed to avoid getting abused.
