# Discretized Streams: Fault-Tolerant Streaming Computation at Scale
Keywords: #spark #distributed #streaming

## Background:

## Summary:
This paper is a follow up of [[paper-2012-nsdi-spark]] and introduces another
abstraction called **D-Stream**. The idea is simple but interesting. Streaming
computation is typically done using "continuous stateful operators" i.e, new
data keeps arriving at a worker, the worker mutates its internal state, and 
sends outputs to downstream workers. 

Because of this stateful workers, failure recovery and straggler mitigation 
becomes hard. Basically, because of stateful operator, computation is no longer
deterministic and stateless where we can rerun tasks. To do fault recovery,
previous systems 
* either do *replicated execution*: same execution is happening on two
uncorrelated nodes. This is (1) wasteful because you are paying for twice the
hardware costs and (2) can't handle stragglers because since the two nodes
have to be kept in sync, the straggler effectively slows down the whole task.
* or do *upstream backup*: inputs and states are periodically sent to a 
backup node. This is also not ideal because (1) failure recovery is slow as 
after a failure the upstream backup must recompute the state while being unable 
to consume new events and (2) straggler mitigation is basically implemented in
same way as failure recovery- recalc the state, which can be a lot of work for
transient stragglers.

So their main idea is to get rid of stateful operators and instead use
*stateless* operators for streaming computation. Essentially, streaming
computation is broken down into small batch computations. Each batch computation
consumes just 1 second of stream, for example, and depends on the output of
previous 1 second batch computation. This *state* data dependency becomes
explicit in the dataflow graph (instead of being implicit in stateful
operators). They call this broken stream as *D-stream* shorthand for
**discretized stream**. D-stream is essentially a sequence of RDDs.

The programming model allows stateful transformations such as sliding windows
which are internally executed in stateless manner.

Advantages of D-stream are that:
* Streaming and batch computation can happen on the same system using similar 
semantics. So they can be mixed. Outputs of batch computation can be used in a
streaming computation and vice versa. Previously, companies managed two separate
systems for batch and streaming.
* Fault recovery and straggler mitigation can just work like in Spark. Without 
wasting resources (as in replicated execution). Recovery can be done in parallel
instead of the single node recovery done in upstream backup. Parallel recovery
is possible as nodes may recover separate work on different timeslices of
DStream and within a slice, on different partitions.

## What can be done better?
- They have high latency due to discretization. [[paper-2015-arxiv-flink-async-snapshot]] has lower latency.
- They can't handle high-frequency trading with latency targets below few
hundred milliseconds. 
- [[paper-2016-osdi-tensorflow]] criticizes the high memory usage of RDDs when 
failures are not that common.

