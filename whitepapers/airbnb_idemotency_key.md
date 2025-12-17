>what it is for

To build [[idempotent]] behavior ensure that requests do not end up doing work multiple times when intention is to only do it once. e.g. double-click or a button, service failing and retrying etc.

> what it would still need

It requires that all services along the path able to honor a concept of [[idempotency]] key and use it to determine if they have already done the work or not. 

> how it works

Every operation will get an idempotency key minted by client (client can use Operation, EntityId, etc as the fields to encode it). So that duplicate operation leads to same idempotency key. On backend side, all services have a Db call wrapped in a library whereby the library checks if incoming idempotency key has been processed already or not and if there is a response present, that is sent. It also keeps in mind whether there was retryable failure- and in that case, it lets the call go to dependency. In the end, idempotency record gets updated with response.

>what race condition thing they found in leader/master replication

Due to replication delay b/w leader and follower, it was found that sometimes follower didn't get the info abnout a new idempotency key operated on by client. So if client quickly did a duplicate operation and request looked up in follower for it, follower will not know about previous information and let the duplicate call go through. To overcome this, they decided to always write only into master. 

> how was the master scaled and how was partition key safe(beacause high cardinality)

Because all the writes were in master, it started getting unscalable. They used idempotency key based sharding in master to share the load- this worked because the idempotency key was typically uniform since it was UUID.

https://medium.com/airbnb-engineering/avoiding-double-payments-in-a-distributed-payments-system-2981f6b070bb