# Dotted Version Vectors: Logical Clocks for Optimistic Replication

Others: Why Logical Clocks are Easy: Sometimes all you need is the right
language.

The point is actually simple but important. In [[paper-2007-sosp-dynamo]], the
version vectors are incremented by the replica that co-ordinates the write. But
the issue is let us that two clients read version vector as `<x1, y5>` and
then wrote back with writes co-ordinated by `x`. The new vector would be 
`<x3, y5>`. `<x2, y5>` write is made by a different client but it is
subsumed by `<x3, y5>` i.e, we lost a concurrent write! 

Another approach is to use client IDs in the version vector to correctly
maintain concurrency across clients. But this approach does not scale. If we
have 1000s of clients, the version vector would become large.

So the crux of the matter is that we want to correctly track the causality and
concurrency among the clients but we want the size of the vector to be
determined by the number of replicas.

The suggestion of the paper is to additionally "dot" the version vector with a
single event. In the previous example, client 1 and 2 had read `<x1, y5>`.  When
the client 1 writes, it is stored with version `<x2, y5>`. When the client 2
writes, it is stored with version `<x1.3, y5>`.

As we know from [[paper-1989-vector-clock]], the version (clock) is just a
counter to indicate how many puts (events) have we seen  on each replicas.  For
instance, `<x2, y5>` indicates that we have seen puts: x1, x2, y1, y2, y3, y4,
y5.

`<x1.3, y5>` indicates that we have seen x1, **x3**, y1, y2, y3, y4, y5. `.3`
extends `x1` with one new event **x3**, i.e, it lets us store non-contiguous
causal histories.

## Thoughts
Q: How come extending the version vector with just *one event* is sufficient?  Why
do we not run into the case where we need to merge `<x1.5>` and `<x2.7>`?  Doing
this merge is not possible because we would have seen put events x1, x2, x5, x7.
Dotted version vectors cannot capture this history.

A: The reason is that it is not possible for anyone to "see" `<x1.5>` and `<x2.7>`
without seeing `x4.6` (or something similar that together captures all the
events). Since x is the replica that is generating the dotted part, and it is
the one who is informing both clients and other replicas of its versions. The
replica itself is guaranteed to have seen `x4.6` (or something similar) between
`<x1.5>` and `<x2.7>` .

Clients must do a get to know the version it is writing on before doing a put.
Clients are **not allowed** to do back-to-back puts.  Back-to-back puts can be
problematic: let us say client 1 saw `<x1>` in first get. It does the first put
and the version is now `<x1.5>`. In the meantime, client 2 does a put to move
version to `<x2.7>`. Now, if client 1 does another put, we will have to assign it
`<x1.5.8>` because its causal history before the put is x1, x5. Forcing a get
between the two puts ensures that the client will essentially be doing a
semantic merge after seeing all the versions therefore superceding all the
versions.

Indeed, [this
writeup](https://github.com/ricardobcl/Dotted-Version-Vectors#consecutive-and-concurrent-writes)
tries to get away from this limitation of requiring get before each put. The 
solution is to add an acknowledgement which pretty much behaves like a get in 
returning all the concurrent versions:

> Each entry of the DVVSetAck is a 4-tuple (id, base, dots, values). The base
represents the largest contiguous causal events without values, beginning at 0.
The dots are all the individual causal events without values, that aren't
contiguous with the base. Finally, values are the single events (dots) with
values, represented by tuples (dot, value).

Essentially, instead of keeping one dot after put we will have to keep multiple dots.

[This writeup](https://github.com/ricardobcl/Dotted-Version-Vectors) shows a
more compact representation of multiple dotted version vectors. Essentially, if 
we have two values `v1: <x1.2>` and `v2: <x1.3>` then we know that they saw
identical versions at `x1` and together they capture the history till `x3`.
They can be stored as `x3, [v2, v1]`. This is read as we are caught up till the
version 3. The entry at version 3 is `v2`, at version 2 is `v1`, and `v1`, `v2`
are concurrent to each other.