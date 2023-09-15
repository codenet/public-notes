# Lightweight Asynchronous Snapshots for Distributed Dataflows
Keywords: #streaming #distributed #flink

## Background:

This arXiv report describes the snapshotting mechanism used in Flink. Flink
supports *stateful* streaming, in contrast to
[[paper-2013-sosp-spark-streaming]] which only supports *stateless* streaming
computation. 

Spark supported only stateless streaming computation by translating streaming
computation into mini batch computation. All the "state" is passed as arguments
to next Spark tasks. This works most of the time but is not always desirable
when state is large. 

More importantly, Spark streaming is not "truly real-time". It adds an
additional delay of breaking things down into mini-batches. Whereas, Flink is
"truly real-time".

## Summary:
Because Flink supports stateful streaming computation, the challenge is in
making Flink fault-tolerant as now we need to also worry about the state.

Checkpointing is hard because the checkpoint needs to be "consistent". At any
given point, different workers will be in different points in their execution 
and in different points in stream. 

[[paper-2013-sosp-mcsherry-naiad.md]] makes these checkpoints "synchronously" by
"stopping the world". We first stop getting new inputs from the event sources,
output queues of all the vertices are flushed, then each vertex is asked to 
checkpoint its state. Only after all this, we can resume the execution. This is
obviously quite bad-- we stopped the world; after resuming the world, the 
downsteam vertices are just sitting idle. Checkpoints can cause input-output 
latency of 10s of seconds (according to Section 6.3 in
[[paper-2013-sosp-mcsherry-naiad]]).

This paper talks about an asynchronous snapshotting algorithm based on 
[[paper-1986-tocs-chandy-lamport]]. The algorithm assumes that all the channels
preserve FIFO ordering. The sources periodically put checkpoint barriers with a
sequence number. Because of FIFO ordering, basically all messages can now either
be "pre-snapshot" or "post-snapshot". 

When a node in the task DAG receives a barrier on one of its input streams, it
temporarily stops processing this stream and waits for receiving barriers on ALL
of its input streams. When it receives barrier on all of its input streams, it
snapshots the local state, puts barrier in all of its output streams, and then
resumes consuming input streams. 

> [Flink](https://nightlies.apache.org/flink/flink-docs-release-1.14/docs/learn-flink/fault_tolerance/)
can now snapshot local state incrementally when using RocksDB for local storage,
i.e, checkpoint changes from the last checkpoint, and can do it inside a forked
process.  Since forked process will use copy-on-write, the original process does
not have to wait for the checkpoint to finish.

This produces consistent checkpoint because every node ensures that it consumed 
ALL of the pre-shot messages and NONE of the post-shot messages.

**Failure recovery**: When any worker fails, the easy approach is to recover ALL
workers from last completed checkpoint. If we were doing checkpoints every 3
seconds (default in Flink), in the worst case we get 3 seconds of delay in messages.

A more advanced approach would recover nodes in DAGs that are related to failure
and let unrelated parts of DAG running. This is quite non-trivial.