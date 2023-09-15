# Virtual Time and Global States of Distributed Systems
Keywords: 

## Background:
* [[paper-1978-cacm-logical-clocks]]
* [[paper-1986-tocs-chandy-lamport]]

Poset: partially ordered sets are written as $(S, \leq)$. For $s_1, s_2 \in S$,
it is possible that $s_1 \not\leq s_2$ and $s_2 \not\leq s_1$.

Lattice: A poset is a lattice if it has finite meet and join operators $\land$
and $\lor$ which are closed, i.e, if $x, y \in S$ then $x \land y, x \lor y \in
S$.

Sublattice: Sublattice of a lattice is itself a lattice:
* its elements are a subset of elements in the lattice,
* the sublattice has the same meet and join operators as the lattice.

The second point is an important point. It may be possible to define new meet
and join operators to make a subset of elements a lattice. But that will not be 
considered a sublattice of the original lattice. This is discussed
[here](https://math.stackexchange.com/questions/310926/sub-lattices-and-lattices).

## Summary:

### Vector clocks
The main point of this paper is that the causality of events is *not **isomorphic***
to the logical clocks proposed by [[paper-1978-cacm-logical-clocks]]. The
logical clocks proposed by Lamport correctly captures that if $e_1 \rightarrow
e_2$, i.e, $e_1$ happens-before $e_2$, then $C(e_1) < C(e_2)$. But, vice versa
is not true, i.e, if $C(e_1) < C(e_2)$, then it is possible that $e_1
\not\rightarrow e_2$.

The reason is that Lamport's logical clocks are represented by integers having a
total order but the causality relationship is only a partial order. This non
isomorphism is ok for the mutex algorithm given by Lamport in
[[paper-1978-cacm-logical-clocks]], but the paper says that this non-isomorphism
loses information about causality which can be a problem. One example is
of debugging: programmers may like to know which two events really were
concurrent and which two events were causally related just by looking at their
clocks.

The paper proposes vector clocks that ensure the clock condition in both
directions, i.e, $C(e_1) < C(e_2)$ iff $e_1 \rightarrow e_2$. A vector clocks
has a counter, one for each process: $<c_0, c_1, ..., c_p>$. When a process
clock ticks, i.e, it executes the next instruction, it increments its own clock 
component. While sending a message, processes send their current vector
timestamp. Let us say that a process receives a message with
timestamp $<c_0', c_1', ..., c_p'>$. Then, the receiving process updates its
clock as $max(c_i, c_i') \forall i$. Since, the process is the only one that is
incrementing its own clock, it is guaranteed to have the $max$ value of its
clock component at any moment. Time propagation is illustrated in Figure 11.

### Global vector time

The time propagation scheme described above *uniquely* assigns a vector
timestamp to each event. But there are *many* sequences of **global vector
times** that *could have been* observed for a given trace by an external
observer. For instance, the following shows three different global vector time
sequences for a trace with 4 events `A`, `B`, `C`, and `D` where `A` sends a
message which is received at `C`, `B` and `D` are internal events.

```                         
       01   02  12  22             01      11  12 22           01   11  21  22    
+------A----B-----------+   +------A-----------B----+   +------A------------B---+ 
       |                           |                           |                  
       ----------                  ----------                  ------         
                |                           |                       |         
+---------------C---D---+   +---------------C-----D-+   +-----------C---D-------+ 
```

Global vector time is maintained by incrementing the clock component of a
process that sees an event. For instance, in the first diagram we first see `A` so
we increment 00 to 01, then see `B` increment 01 to 02, next we see `C` incrementing
02 to 12 and finally see `D` to increment 12 to 22.

The reason we are able to see different global vector time sequences is that we
can think of each process timeline as an ideal rubberband. Any portion of the
rubberband can be elongated and compressed. The second sequence shown above
stretches the timeline for the first process pushing both `B` after `C`.

However, when stretching and compressing this rubberband we need to ensure the
forward direction of messages. For instance, the following cannot be observed.
```                         
    10 11 12 22
+------A--B-------------+
       |                 
    ----
    |                    
+---C--------D----------+
```

Formally, for a given trace of events, all possible global vector timestamps can
not be observed.  Message send and receive events disallow some timestamps. In
particular, let us say that the process $i$ send a message to the process $j$.
If the message is sent at $c_i = s$ and is received at $c_j = r$ then we shall
never see any event in the system where $c_i < s \land c_j > r$. This is because
any event that has $c_j > r$ is causally affected by the receipt of the message
sent at $c_i = s$. Therefore, if $c_j > r$, $c_i$ must atleast be $s$.
Therefore, the permissible vector timestamps for a given trace is only a subset
of $N^p$.

Now, clearly $(N^p, \leq)$ forms a lattice. The paper claims that the set of
possible observed global times form a sublattice of $(N^p, \leq)$. For instance,
the following shows the lattice $(\{0, 1, 2\}^2, \leq)$ and the set of possible
observed global vector times for the above trace. Essentially, 10 and 20 cannot
be observed.

```
     22              22     
    /  \            /  \  
  12    21        12    21  
 /  \  /  \      /  \  /
02   11   20    02   11
 \  /  \  /      \  /
  01    10        01
    \  /          |
     00           00     
```

This sublattice of observable global vector times follows from the trace. For
instance, the left figure shows the poset diagram of events. Each path in the
sublattice above comes from different order of events that can be observed over
the poset of events. 

```
    C          22    
    |         /  \   
 B  D        C    B
 | /         |    |   
 A          12    21 
           / \    |  
          D   B   C
          |    \ /   
          02   11    
          \    /
           B  D
            \/ 
            01
            |
            A
            |
            00     
```

### Cuts

Now that our virtual time is a vector, we can think about the semantic meaning
of this vector.  Essentially, a vector $v \in N^p$ can be seen as a **cut**
through the timeline of events. This cut separates the execution timeline into
*past* and *future*. Past contains a total of $v_i$ events that happened on
process $i$.

A cut is defined **consistent** if messages are never sent from future to past.
Intuitively, it is quite clear that only the global vector times, i.e, the
sublattice above will give us a consistent cut. This is because the global
vector time sublattice comes from the event poset which takes care of the
message sending order (`A` must happen before `C` in our example).

Therefore, **global vector time uniquely defines a global state** where global
state is the state of the system (state of all the processes and the messages
in-flight on all the channels) *after* seeing all the *past* events defined by
the cut corresponding to the global vector time and *before* seeing any of the
*future* events.

Global state is a state that the system **could have reached**. It is
*not* a state that the system reached for sure during the execution. Global
states are all the states according to the sublattice of global times which is
isomorphic to the sublattice of consistent cuts. An execution is just one path
through the sublattice.

### Consistent checkpoint

This paper beautifully defines the problem solved by
[[paper-1986-tocs-chandy-lamport]]. The current execution is a path through the
sublattice but a consistent checkpoint algorithm captures another global state
in the sublattice that *could* have occurred.  Figure 8/Theorem 4 illustrates
this. Process timelines can be stretched/elongated such that all the cut events
in a consistent cut occur simultaneously in real time.

Why is it guaranteed that we get a consistent checkpoint (a consitent cut or an
observal global time) via the algorithm?

The algorithm assumes that the messages are delivered in-order. Once a process 
gets the marker on a channel, it stops processing more messages from that
channel. When it receives a marker on all the channels, it checkpoints its local
state and sends markers on all the outgoing channels. Therefore, the point at
which a process checkpoints are the cut events. Process timelines can be
compressed to bring cut events to the same real time so that messages do not go 
backwards.

The paper also gives an algorithm to create a consistent checkpoint even when
messages are not guaranteed to be delievered in-order. The algorithm critically
uses the vector timestamps. 

The idea is that the vector timestamp has $P+1$ components for $P$ processes.
The last component is incremented only to create the next checkpoint. The
algorithm also uses the property that the process $i$ which owns the $v_i$
component of the vector timestamp always has the max value for that component
(i.e, it is most caught up).

The algorithm proceeds as follows: a co-ordinator process decides to checkpoint 
spontaneously. It increases the $v_{p+1}$ by 1. Since, the co-ordinator process 
is the only process that can initiate a checkpoint, it is guaranteed to have the
maximum value of $v_{p+1}$. The process checkpoints itself and starts sending
messages timestamped with the new vector timestamp.

Let us say that the old value of $v_{p+1} = d$ and new value is $d+1$.  Now it
is straightforward to distinguish messages sent before the checkpoint and the
messages sent after the checkpoint.

When a process receives a message with $v_{p+1} = d+1$ for the first time it
checkpoints itself and updates its clock according to the usual time propagation
mechanism. Therefore, all the messages sent by it will have $v_{p+1} = d$. 

But since messages may be delivered out-of-order, a process that had already
checkpointed may receive $v_{p+1} = d$ messages. These messages need to be
checkpointed separately as "in-flight" messages of the channel.

The only remaining difficulty is how to detect whether the checkpoint is
complete. Since, messages may be arbitrarily delayed, we may not know if we
might still receive a message with $v_{p+1} = d$. The paper proposes a simple 
solution: each process remembers the count of $v_{p+1} = d$ that it has sent
minus the count of $v_{p+1} = d$ messages it has received. 

When the process creates a checkpoint, it sends this count to the coordinator
process. This count can be summed across all the processes to know the pending
in-flight messages. As messages are received, this count is decremented. When 
the count hits zero, we can be sure that the checkpoint is complete!

## Random thoughts:

This paper is phenomenal, full of very interesting ideas. 

I love this kind of writing. Where the author is really in a
conversation with the reader. Such as this paragraph:

> However, these statements are rather vague and give rise to a few questions:
What exactly is virtual time, and what should b e considered to be a "best
possible approximation" of global (i.e., real) time? And given an algorithm
which was written assuming the existence of real time, is it still correct when
virtual time is used? What is the essential structure of real time? These
questions will be considered in the sequel.

Unfortunately, this style is not observed nowadays in systems paper. Another
example is from [[paper-1986-tocs-chandy-lamport]]:

> The state-detection algorithm plays the role of a group of photographers
observing a panoramic, dynamic scene, such as a sky filled with migrating birdsa
scene so vast that it cannot be captured by a single photograph. The
photographers must take several snapshots and piece the snapshots together to
form a picture of the overall scene. The snapshots cannot all be taken at
precisely the same instant because of synchronization problems. Furthermore, the
photographers should not disturb the process that is being photographed; for
instance, they cannot get all the birds in the heavens to remain motionless
while the photographs are taken. Yet, the composite picture should be
meaningful. The problem before us is to define “meaningful” and then to
determine how the photographs should be taken.
