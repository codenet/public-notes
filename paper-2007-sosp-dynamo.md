# Dynamo: Amazon’s Highly Available Key-value Store
Keywords: #kvstore #availability #scalability #storage #performance
- [Dynamo: Amazon’s Highly Available Key-value Store](#dynamo-amazons-highly-available-key-value-store)
	- [Background:](#background)
	- [Summary:](#summary)
		- [Why performance? What kind of performance?](#why-performance-what-kind-of-performance)
		- [Why key-value store over a DB?](#why-key-value-store-over-a-db)
		- [Why availability over consistency?](#why-availability-over-consistency)
		- [How to deal with inconsistencies?](#how-to-deal-with-inconsistencies)
			- [Version histories (Figure 3)](#version-histories-figure-3)
			- [Reconciling inconsistencies](#reconciling-inconsistencies)
	- [Decentralized design](#decentralized-design)
		- [Distributing the keys and finding them](#distributing-the-keys-and-finding-them)
		- [What to do when nodes fail? Or when a new node joins?](#what-to-do-when-nodes-fail-or-when-a-new-node-joins)
		- [How does \[\[topic-gossip-protocol\]\] work?](#how-does-topic-gossip-protocol-work)
## Background:

## Summary:
The main focus of this paper is performance and availability. They trade off
consistency and durability to achieve these. But they also provide some knobs to
increase durability to trade off performance. 

### Why performance? What kind of performance?
Amazon has strict SLAs. For example, one page load request may send requests to
~150 services such as recommendation service, discount service, etc. to prepare
a webpage. Each of these services need to respond very quickly. Since services 
are stateful they need a storage layer that can respond very quickly too.

Each service manages its own Dynamo instance. Dynamo aims to provide 99.9
percentile latency less than 300 ms.

### Why key-value store over a DB?
Services rarely use fancy joins etc. Just keeping a blob accessible by key is
enough. For example, their shopping cart service is only keeping the items in 
the cart in a blob file with key as user id.

DB is harder to shard and manage. They want to build a completely decentralized,
highly available storage layer which continues to run even while data centers 
are being destroyed by tornadoes.

### Why availability over consistency? 
Consistency slows things down because you have to wait for the majority of
replicas to respond. So, there is a lot of chit-chat on the critical path
(get/put). Further, get and put runs as slow as the fastest `ciel(N/2)` nodes.
This is because we need `ciel(N/2)` responses required to form a majority. If we
can't form a majority immediately, we will have to retry leader elections etc 
which can further slow things down. 

This breaks down the SLA requirements. Since, we care about 99.9 percentile
latency and since some servers / network / etc is breaking all the time, we can
not achieve low latency if we cared too much about consistency. See *PACELC
theorem*.

So typically in Dynamo setup, we assume that a write is complete if any single
node has received the write. This significantly improves the write availability
and write performance.

### How to deal with inconsistencies?
Inconsistencies can occur if let's say a user adds stuff to their cart from two
different client devices like a laptop and a phone. These writes may go to two 
different replicas holding the user data. The writes will be considered complete
as soon as these replicas complete the writes.  Now, even though the writes were
fast (not having to wait for majority of the replicas), these replicas hold
inconsistent data. 

It may be possible that the second write is actually "after" the first write,
i.e, the second replica is aware of the first write, in such cases the second
write should *win* and overwrite the first write.  But it is also possible that
the first and the second write were indeed concurrent or because of network
failures the replicas could not see each other's writes for a long time.

#### Version histories (Figure 3)
To figure out which one of the two it is, Dynamo keeps track of the order in
which each key is written by associating a ***version vector*** with each key.
More specifically, each write is considered immutable: multiple **versions** of 
the same key may be present in Dynamo at the same time. 

Upon calling `get(k)`, a `value, context` is sent back by Dynamo.  This
`context` contains the vector clock of the object that is being returned.
Writes are done by `put(k, context, value)`; `context` in `put` is used to 
create the version history, such as in Figure 3.

> Q: What if I directly call `put` without a `get`?
>
> A: I think empty context will be considered as *after* the latest context of 
the server handling the request. So in Figure 3, if I get a new `put` request 
with an empty context, the version will become [Sx4, Sy1, Sz1]. This  is of
course risky because the client may be overriding all the writes made so far.
The paper does not clarify this. Web services are probably themselves
*stateless* keeping all the state in Dynamo. So they always do a get first
before a put.

> Q: What if multiple clients are concurrently writing and the request gets
handled by the same server? Will they overwrite each other?
>
> In other words, what we are saying is that two clients did a `get`. Both 
clients got `[Sx1]` as the vector clock in `context`. Now, when both clients 
do a `put(k, [Sx1], v1)` and a `put(k, [Sx1], v2)` what will happen? 
> 
> A: Server likely has a queue of requests, so it will apply one of the updates
first.  Let's say now we have `k: v2` with it's clock `[Sx2]`. So the question
is what will happen if it now receives a stale update from `put(k, [Sx1], v2)`.
>
> The server has a few options. Either we can just reject the update by throwing 
an exception on `put`. Or we can reconcile the new update with the last one.
It is unclear what the paper actually does. 
> 
> One way to resolve this could be to add client in the vector clock as well. As
then, the two put requests from clients `p` and `q` will be `put(k, [Sx1, Cp1],
v1)` and `put(k, [Sx1, Cq1], v2)` and they must be reconciled because there is
no ordering among the clocks as they are originating from different clients.
>
> [How cassandra deals with this.](https://stackoverflow.com/questions/44534564/cassandra-concurrent-writes)
> Basically, last writer wins.
> 
> Riak implements dotted version vectors to deal with this as described in
[[report-2010-arxiv-dotted-vv]].

#### Reconciling inconsistencies
When the version histories have diverged, such as in Figure 3, we need to
somehow reconcile inconsistencies.

There are two proposed ways to **reconcile inconsistencies**: 
* **Syntactic reconciliation**: Here client can configure Dynamo to let the
write with latest physical timestamp win. This is ok sometimes such as when we
are maintaining client sessions in Dynamo for example. 
* **Semantic reconciliation**: Here the client is provided the diverged versions
and uses semantic information to collapse version histories. For example, let's
say two writes added different things to the cart, client can collapse this
history to add both the things in the cart.

## Decentralized design

Another important feature of Dynamo is that it is completely decentralized, i.e,
without any special master, such as in [[paper-2003-sosp-gfs]]. So, it has to do
things that GFS master did:
* Who has the key (chunk)?
* Give out leases for handling writes
* Load balancing
* Failure detection and recreating three copies of each chunk

Dynamo does most of these things in a decentralized manner. It explicitly does not
do two things: 
* Giving out leases: Dynamo follows a *write anywhere* design as described above
to provide high-availability. They are ok with having "split brain" since
inconsistencies will be reconciled later.
* Failure detection: Dynamo leaves it to admins for explicitly adding and
removing nodes.  Maybe a node is just restarting: it will be wasteful to move
all its keys out and then moving a new set of keys to the node.

### Distributing the keys and finding them

The basis of decentralization approach is from [[paper-2001-sigcomm-chord]]
where we use consistent hashing to put the keys on a token ring. Servers also
place themselves on the same token ring. 

Dynamo makes a few changes over [[paper-2001-sigcomm-chord]]. It split the hash
space into `Q` token ranges, so the worst case routing table is O(Q). Workers
place themselves at token range boundaries, i.e, while balancing load, workers
steal one or more token ranges.

A key is held by three servers that follow the key hash in the clockwise
direction on the token ring. These three servers are close on the token ring but
we would want them to have uncorrelated failures for durability. For example,
the three servers should be in different data centers. 

A client downloads the token ring configuration (the routing table) from any of
the dynamo node. It finds which range a key lies in and then find the worker in
its own routing table. If that node is unreachable, it can ask the worker next
in the token ring.

### What to do when nodes fail? Or when a new node joins?
Token redistribution
Merkle trees

### How does [[topic-gossip-protocol]] work?
https://docs.datastax.com/en/cassandra-oss/3.x/cassandra/architecture/archGossipAbout.html
https://cwiki.apache.org/confluence/display/CASSANDRA2/ArchitectureGossip
https://en.wikipedia.org/wiki/Gossip_protocol#Examples