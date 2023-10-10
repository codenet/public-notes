# Object Storage on CRAQ
Keywords: #replication

The idea of CRAQ is pretty simple. Similar to how [[paper-2010-atc-zookeeper]]
scales reads over [[paper-2014-atc-raft]] by letting clients read from any of 
the server, CRAQ scales reads over [[paper-2004-osdi-cr]] by letting clients 
read from non-tail servers. 

The interesting difference is that CRAQ is able to maintain linearizable reads
whereas [[paper-2010-atc-zookeeper]] gives up on linearizable reads and only
supports monotonic reads.

The way it works is that let us say we have 4 servers for a chain:

``` 
A --> B --> C
```

Here, A is the head and C is the tail. In normal [[paper-2004-osdi-cr]], writes 
appear at A, and acknowledged at C. Reads are served by C. This is clearly
linearizable because the actions are linearizable in terms of C. But for read 
heavy workloads, C becomes a bottleneck.

The idea of this paper is simple: let us maintain multiple versions at non-tail 
node B for each key. When we get a write from the head, we maintain the old
versions and mark the new version as dirty. When tail acknowledges a write to
the client, it passes this acknowledgement back to the chain. Intermediate nodes
B and C can delete versions that are older than the acknowedged version and mark
the acknowledged version as clean. Tail only ever has one version for a key.

Reads can now be done at B and C. The reads at the tail C is same as before.
Reply directly with the only clean value. 

### Reads to non-tail

The interesting case is when a read comes at B. If we only have one clean 
version, we can immediately reply back with the clean version. If we have some
dirty versions, we need to first ask C about their version number since their
acknowledgement may not have made to us. We respond back to the client with the
version number that C told us about.

Let us see why we need to coordinate with the tail. Let us say that B is allowed
to respond directly to the read requests made by the client.

B responds with the latest dirty version. Here, we show three different clients
but label them with which server they are talking to:

```
X:  |--Wx0--|
A:                   |----- Wx1 -----|
B:                      |-Rx1-|
C:                              |-Rx0-|
```

The client talking to B saw a 1 because it was already replicated to it, but the
client talking to C saw a 0 because the write didn't come to the tail yet. This
is not linearizable.

Another option is that B responds immediately with the latest clean version.
```
X:  |--Wx0--|
A:                  |- Wx1 -|
B:                           |--Rx0--|
```
Here, the server B thinks that Wx1 is dirty, but actually it has already been 
acknowledged by the tail.

One interesting point is that actually C might acknowledge a write *after*
responding to B about an older clean version. This gives us the following 
history:

```
X:  |--Wx0--|
A:                  |---- Wx1 ----|
B:                         |----Rx0--|
C:                           |-Rx1-|
```

**Interestingly, this is still linearizable**. The behavior is same as that of a
single server C. The coordination between C and B is just like an increased
network delay.
 
A surprising point about CRAQ is that even **for write-heavy workloads**, we can
get higher read throughpt.  For example in Figure 6 when there are no writes,
read throughput is 3x that of regular CR since each node in the chain can
directly respond with the only clean version without requiring any
co-ordination. As we increase writes/s, the number of read requests containing
dirty versions will increase (shown in Figure 8). 

One would expect that the read throughput will drop to the CR scheme since we
have to pretty much coordinate each read with the tail now. But, it turns out
that although the number of requests to the tail is the same, the size of the 
response is much smaller. In read requests, tail has to respond with the full 
object. In version query, tail has to just send the version number of the
object. Therefore, we still see higher read throughput. 

This paper like many other papers is trading off latency for throughput. As we 
increase the length of the chain, we can get an increase in the read throughput,
but at the expense of increased write latency. Although, it is improving read 
latency over [[paper-2004-osdi-cr]] for read-heavy workloads. Since in CR, we
were forced to read from the tail, the tail may be in a different data-center.
In this scheme, we can read from a non-tail node that is closer to us. 

#### Comparisons with [[paper-2014-atc-raft]]

Pros:
* Chain replication balances the load much better than Raft. In Raft, the leader
has to send all the messages to everyone else. This can be expensive if reads
and writes are heavy.
* CRAQ wins in proximity. A client can read from a nearby data center. In Raft,
the client is forced to read from the leader which may be far. 

Cons:
* Chain replication cannot handle stragglers. If even one of the node is slow 
in the chain, it will completely destroy the write throughput and latency.
* Chain replication needs a third party to handle network partitions and faults.
Since it does not have a leader election mechanism, it can end up with
split-brain situations if it tries to remove a node on its own.

Typically, both of them are used in conjunction. Raft/zookeeper will manage the
"control space" like what is the chain and how are objects sharded across the
chains. Maybe there is a service running that monitors the health of the chain 
like doing heartbeats with nodes on the chain, etc.