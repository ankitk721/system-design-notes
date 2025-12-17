[[Tow phase commit]] is to ensure atomicity and consistency guarrantee when a transaction involved more than one nodes. For a single node, you can easily lock the row or fail the conflicting operation at commit-time to provide atomicity. However, doing it across nodes require a network call to communicate and decide if transaction can be committed or not.

> how it works

It involves one node acting as co-ordinator to orchestrate the network calls among participants and owning the response to caller. It splits the entire transaction into two parts `prepare` and `commit`. During prepare, call is made to each participant node to check if that node can make this transaction or not- the participant node can say yes and co-ordinator will proceed if all yes-es are received from the participant nodes. Or participant can say no if there are any constraint violations, conflicts or any other hw/sw failures. If no is received then co-ordinator aborts the transaction and lets caller know. If all yes then it locally records in log that it is proceeding with COMMIT and then broadcasts to participant nodes the same. Those nodes already had data ready to commit and upon hearing this, they commit into their logs.

> what the big deal with prepare and why split it into two

take example of airline seat reserve and then confirm seat  plus confirm hotel -> cant have partial booking that doesnt work. Also prepare has to be fully 100%, participant should ensure that it writes to WAL on disk before saying yes and also acquires any locks needed for this transaction to succeed. And it also puts transactionId in its in memory table with status as `Prepared(in-doubt)`so it can later quickly look it up when commit broadcast arrives.

> what are the log statemetns like 

`Prepared`(t1, p1, p2..)`
`commit (t1)`
`aborted(t1)`
`completed(t1)`

>what are the issues- leader crashes, participant crashes

leader crashes-> design for High Availabiliity Leader using Distributed Raft Consensus based coordinator fleet

why raft- so they can automatically pick leader who can run the show
whats with the commit messages in raft setup? Instead of broadcasting at whim as a single co-ordinator would have, in this setup such leader cordinator would first get consensus from (n+1)/2 nodes in raft manner that if it can proceed or not. Once found yes, it will proceed. Raft does its own local log of operation and then sends append RPC to followers- and based on what responses it gets, it marks a commit cursor and sends that notification to followers too.

can we lose state if leader crashes? No, raft ensures that when new election is held other nodes would find out who doesnt have up to date commited messages and dont let them be the leader


