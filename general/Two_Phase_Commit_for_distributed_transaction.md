what its for
how it works
what the big deal with prepare and why split it into two
take example of airline seat reserve and then confirm seat  plus confirm hotel -> cant have partial booking that doesnt work

what are the log statemetns like 
Prepared(t1, p1, p2..)

commit (t1)
aborted(t1)
completed(t1)

what are the issues- leader crashes, participant crashes

leader crashes-> design for High Availabiliity Leader using Distributed Raft Consensus based coordinator fleet

why raft- so they can automatically pick leader who can run the show
whats with the commit messages in raft setup? Instead of broadcasting at whim as a single co-ordinator would have, in this setup such leader cordinator would first get consensus from (n+1)/2 nodes in raft manner that if it can proceed or not. Once found yes, it will proceed. Raft does its own local log of operation and then sends append RPC to followers- and based on what responses it gets, it marks a commit cursor and sends that notification to followers too.

can we lose state if leader crashes? No, raft ensures that when new election is held other nodes would find out who doesnt have up to date commited messages and dont let them be the leader


