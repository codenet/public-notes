# topic-gossip-protocol

https://docs.datastax.com/en/cassandra-oss/3.x/cassandra/architecture/archGossipAbout.html
https://cwiki.apache.org/confluence/display/CASSANDRA2/ArchitectureGossip
https://en.wikipedia.org/wiki/Gossip_protocol#Examples

[[paper-2007-sosp-dynamo]] and #cassandra use gossip protocol. Also described in
[[paper-1989-epidemic-algo]].

Main problem we are solving is that we want to co-ordinate among all the nodes
without any centralization i.e, a master. In the #dynamo setting, whenever there
is an update, such as the load of a node changes, we can potentially do a
broadcast and tell everyone "hey btw my load has reduced from 1 to 0.3". 

**Why not broadcast?**

Broadcasting is decentralized but it has two problems:

* Too much network traffic: Everyone will be updating everyone else all the time
* Network partitions: Some nodes will not hear about other nodes at all for a
prolonged period of time in the case of network partitions because all messages
are point-to-point.

**Why do we need version numbers for each node?**

This is basically a vector clock. During gossip we'd like to know if we are
caught up about other nodes or not. For each node, we will compare incoming
version number with the version number I'm aware of. If the incoming version 
number is *after* my stored version number, I'll ask for the incremental updates. 
If my version number is higher, I'll send the diff in my ack.

**Why do we need seeds?**

Seeds are statistically more likely to be more up-to-date about the cluster
information. Since in each round, each node gossips with a seed with a
probability higher than gossiping to non-seed. When a new node joins, they get
their information from the seed.

**Why do we need generation numbers?**

Generation numbers keep track of node restarts. Why do we need to track this?
Maybe we don't want to do write ahead logging for any update? We only do WAL
once to write down the generation number. Next, version numbers can be updated 
without doing WAL?

**Is this robust to network partitions?**

Yes. Hopefully some other node is able to talk to a node unreachable to me.  I
will hear about the updates of an unreachable node over time.

**What if there is a full partition between two data centers?** 
**What is the impact of this on #dynamo?**

Data center A will think all nodes in B are unresponsive and vice-versa. In
Dynamo, nodes are only removed *explicitly* by an adminstrator. In the absence
of that, failures are assumed temporary.

**Why is it ok for information to lag behind?**

Let's say Node `A` has given its key ranges to Node `B`. But Node `C` is unaware of
this.  When a write request comes, we will ask `A` to co-ordinate the write.
What should happen then?

> In this scenario, the node that receives the request will not coordinate it if
the node is not in the top N of the requested key’s preference list. Instead,
that node will forward the request to the first among the top N nodes in the
preference list.

Well, `A` is definitely aware of the move of key ranges, so it hands off the
request to `B`.

So, I think the basic idea is when we actually *act* on the delayed information,
the information gets updated. If a busy worker tries to give its set of keys to
a lightly loaded worker, that worker can reject this request if it has become
heavily loaded.