# CIEL: a universal execution engine for distributed data-ﬂow computing
Keywords: 

## Background:

## Summary:
Main contribution is to allow data-dependent dynamic control flow unlike
[[paper-2012-nsdi-spark]], [[paper-2007-eurosys-dryad]], and
[[paper-2004-osdi-mapreduce]].  This approach is popular in the industry: used
in #celery, #dask, #ray, #airflow.

The following shows an iterative computation. Here, 

```js
function process_chunk(chunk, prev_result) { 
	// Execute native code for chunk processing. 
	// Returns a reference to a partial result. 
	return spawn_exec(...); 
} 
function is_converged(curr_result, prev_result) { 
	// Execute native code for convergence test. 
	// Returns a reference to a boolean. 
	return spawn_exec(...)[0]; 
} 

input_data = [ref("ciel://host137/chunk0"), ref("ciel://host223/chunk1"), ...]; 
curr = ...; // Initial guess at the result. 
do { 
	prev = curr; 
	curr = []; 
	for (chunk in input_data) { 
		curr += process_chunk(chunk, prev); 
	} 
} while (!*is_converged(curr, prev)); 
return curr;
```

The input is still immutable so we will have the same issue as
[[paper-2012-nsdi-spark]] while running ML training that updates weights. The
dynamic tasks were made stateful via actors by [[paper-2018-osdi-ray]].
Although, this is perhaps slightly better than spark because we can get more
control over what we want to do with each RDD partition. Here, each partition is
a separate object. With this flexibility, we can recover Spark like abstraction
as is done by [Google's Dask on panda arrays and
lists](https://docs.dask.org/en/stable/10-minutes-to-dask.html). 

Slowly, issues highlighted in this paper are being solved by Ray. The paper
mentions that serialization is done via Json. This was worked upon in Plasma to
share memory across worker processes. This paper clearly hightlights the issue 
of task scheduling overhead which was solved by Ray's distributed scheduler.
This paper is a good segue into talking about Ray.

### Evaluation
Takeaways:
* Grep: Small setup time for the job and for each task 
* K-means: Data locality across iterations. Spawning different MapReduce job for each 
iteration couldn't use cross-job locality. Job doesn't know that a GFS partition
may already by in a worker's page cache.
* Smith-waterman: Smaller tasks are great because we can have more tasks than
workers keeping the workers better utilized. But, with smaller tasks we start 
having increased scheduling overhead.

> At the time of writing, an m1.small instance has **1.7 GB** of RAM and 1
virtual core
  
The RAM is much smaller than what is used in [[paper-2012-nsdi-spark]]: Unless
otherwise noted, our tests used m1.xlarge EC2 nodes with 4 cores and **15 GB** of
RAM. We used HDFS for storage, with 256 MB blocks. Before each test, we cleared
OS buffer caches to measure IO costs accurately.

But, the input file still fits in page cache. In k-means, 5 iterations are run
on 20 workers. Each task takes 64MB and we run a maximum of 5 tasks on each
worker (maximum of 100 total tasks). So each worker is only working on
64*5=320MB. And in total we are only working on a 6GB file. In contrast,
[[paper-2004-osdi-mapreduce]] shows evaluation on multi terabyte file.

> Hadoop is not well-suited to short jobs, which is a result of its original
application (large-scale document indexing). However, anecdotal evidence
suggests that production Hadoop clusters mostly run jobs lasting less than 90
seconds [40].

> When computing an iterative job such as k-means, CIEL can use information
about previous iterations to improve the performance of subsequent iterations.
For example, CIEL preferentially schedules tasks on workers that consumed the
same inputs in previous iterations, in order to exploit data that might still be
stored in the page cache. When a task reads its input from a remote worker, CIEL
also updates the object table to record that another replica of that input now
exists. By contrast, each iteration on Hadoop is an independent job, and Hadoop
does not perform cross-job optimisations, so the scheduler is less able to
exploit data locality.

### Design 
So, the design is pretty straight-forward now that we have read it in multiple
formats. The main idea is that tasks can spawn new tasks. Each task returns a
future. We can do a `.get` on the future, (or `*fut` in this paper or `.compute`
in #dask) to block the execution while the task is running. 

When a task finishes, the future is *concretized* into the object store of the
worker. All the references are being tracked in a central object table at
master. In celery, the task outputs were written directly into Redis which is
going to be very slow.

These futures can be passed to new tasks as arguments. This essentially ensures
that the tasks form a DAG. Tasks are stored in the *task table* on master.
Master schedules tasks from task table to workers as soon as a task is ready.  A
task is ready after all its inputs are concretized. Master tries to keep data
locality when spawning tasks. 

If a task has to start at a different machine than where its inputs are, the
inputs will be serialized as JSON and get downloaded to run the task. 

The approach to #ft is similar as in other papers. Tasks are deterministic and 
idempotent. Master keeps heartbeating with the workers. If the worker is not 
responding to a heartbeat, the task is marked as runnable. The scheduler will 
then re-execute runnable task. Therefore, the task should be deterministic and 
idempotent. 

It is possible that the input to a task becomes inaccessible because the worker 
holding the input in its object table crashed. The worker will complain to the 
master, master will invalidate the locations in the object table for each
missing input, rerun the task that will re-generate the input, and mark the
original task as pending. 

The input to the rerun task may itself be missing. So it will again follow the
same process as above recursively. In essence, this is lineage based recovery.

### Runthrough with an example:

Let us say we have the following simple task DAG:
 
Z -> a -> a:Z -> b -> b:A -> c -> c:B

Z is the input to the program, like a GFS chunk, resident at both W0 and W1. 

For memoization, we keep deterministic naming of objects: <method name,
hash(arguments), output number>. (For simplicity we are assuming each method 
only generates a single output).  In case, the task `a` is asked to be
re-executed with input `Z`, we need not run it again if its output is already
there in the object table.

| Task table: 	| Object table
| --------------|-------------
| 		| Z: W0, W1
| a: runnable	| a:Z pending
| b: pending	| b:A pending
| c: pending	| c:B pending

Since `a` is runnable, we can schedule `a`. For locality, we run it on `W1`. Let
us say that `a` finishes:

| Task table: 		| Object table
|-----------------------|-------------
| 			| Z: W0, W1
| a: **done**		| a:Z **W1**
| b: **runnable**	| b:A pending
| c: pending		| c:B pending

Now task `b` got spawned on worker `W2` (because `W1` was busy) and it tries to
read `a:Z` from `W1`. By this time, `W1` has crashed and therefore we lost the
object `a:Z`. `W2` complains to the master. Master marks `a` as runnable again:

| Task table: 		| Object table
| ----------------------|-------------
| 			| Z: W0, W1
| a: **runnable**	| a:Z **W1: unreachable at time t**
| b: **pending**	| b:A pending
| c: pending		| c:B pending

We might also like to remember that `W1` had the output of `a`. It is possible
that `W1` just restarted or `W1`'s network was temporarily disconnected.
Now let us say that `b` is done at `W3`:

| Task table: 		| Object table
| ----------------------|-------------
| 			| Z: W0, W1
| a: **done**		| a:Z **W2, W1: unreachable at time t**
| b: **done**		| b:A **W3**
| c: runnable		| c:B pending

Now, in this situation if `W2` crashes holding the output of `a`, we need not 
run it again since the task that used `a` has already finished. 

### Recovering master from faults

Another interesting aspect is how to recover from master failure. The
paper talks about a few obvious things: 
* before giving acknowledgement to the client about the starting of the job,
  write the root node task description to distributed storage.
  * if master crashes, new master can read root node task description and rerun 
    the job from the beginning.
* In the beginning, seconday master is told about all the workers. Secondary
master keeps heartbeating with primary master; if it is unable to ping, it 
checks with workers that they are also not able to ping. After we establish that
primary is dead, secondary can take over as master. 

Primary master has been asynchronously telling secondary master about the task
table and the object table. We recover task table and object table from what we
know so far. Then we ask the workers to tell us about the objects they might be
holding and that we are unaware of. 

**This is the most interesting part**. Without this, we will need to run all the
tasks in the task DAG after the checkpoint. But by recovering the object table,
we reduce the number of tasks we need to run. We can skip tasks whose results
are already present with workers. 

If some objects are garbage collected, they may not be recovered into the
master's object table. Now master might reschedule functions computing these
objects. But these objects are already "dead" in the sense that the functions
using these objects have already done what they wanted to do! So, there is
actually no point in trying to recompute these objects.

For example, let us say our task DAG is 

a -> A -> b -> B -> c -> C

Here, a, b, and c are tasks and A, B, and C are objects. We are interested in
computing C. Let us say that before the master crash, we were done executing B,
we garbage collected A, and we were about to run c.  When master restarts, it
will learn that B is calculated but A is not calculated, therefore it might mark
b as pending. 

This is not that hard to solve. Master can take a lazy approach and say we are
interested in C. For that I need to execute c, for that I need B. I already have
B, let's execute c.

## Thoughts

In trying to learn CIEL, we tried to write the [0-1 knapsack
problem](https://www.geeksforgeeks.org/0-1-knapsack-problem-dp-10/) by
parallelizing it in a manner similar to in Figure 9a. Turns out we **could
not**!  The memory access pattern cannot be supported by CIEL!  CIEL
builds a graph of futures and tasks.  Each task exactly knows the futures that
it wants. In CIEL Figure 9a, we require exactly the block above and to the left
of the current block. 

However, in 0-1 knapsack we require *all the left blocks* since we have to do
`K[i][w] = max(val[i-1] + K[i-1][w-wt[i-1]], K[i-1][w])`. Here,
`K[i-1][w-wt[i-1]]` may not be in the adjacent left block!

One option is that we declare dependency on all the blocks to the left. But then
a system like CIEL/Ray will have to transfer all these blocks to the worker
*before* running the task.  However, in reality we may need only a few of them.
So, essentially CIEL and Ray are conservative in terms of which futures they
desire. 