# Distributed snapshots: determining global states of distributed systems
Keywords: 

Powerful algorithm used in [[paper-2015-arxiv-flink-async-snapshot]]. The
difficulty is that we don't want to pause computation. We want to take a
consistent checkpoint while the computation is running. Main thing is to define
consistent checkpoint. Consistent checkpoint never occurred at the same time
during the computation. 

In any computation that we checkpoint:
* Initial state can lead to the consistent checkpoint
* Consistent checkpoint can lead to the final state

Paper also says that we can use this to check for *stable properties*. Stable
properties are basically system invariants:
* Once the program terminates, it remains terminated;
* When system enters a deadlock, it stays inside deadlock;
* If system has one token, it will always have one token.

These snapshots can monitor whenever the stable property has becomes true.