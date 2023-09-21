# The Google File System
Keywords: #gfs #bigstorage

## Summary:
What is new?
- Hundreds of terrabytes of data spread over 1000s of disks. 
- Component failure is the "norm". Need a fault tolerant file system.
- Files are huge. Multi GB files are common. Example: output of a crawler.
Typical files in a "regular" file system are a few KBs.
- Random writes are uncommon. Most files are only appended.
- Reads are also sequential reads of whole files instead of seeks. Example:
Mapreduce computes incoming links from crawler's file to compute pagerank.

Goals:
- Fault tolerance using replication
- Scale: high read-write bandwidth
- Atomic appends

Effect of context on design:
- Large chunk sizes 64MB. 
  - Large chunks because we are optimizing for large files
    unlike the smaller 512B/4KB disk blocks. 
  - Increasing chunk size reduces master-client communications.
  - Reduces metadata on master.
  - Client can keep persistent TCP connection with chunkserver to read write 
    for a longer period. 
- Client-side caching is useless. Clients are going through huge files
sequentially instead of repeatedly reading different indexes of the same small
file.


Architecture:
- Clients are servers within Google. E.g. crawler, mapreduce worker.
- A GFS master and some chunkservers 
- Chunks have a unique chunk handle. 
- Chunkservers hold a chunk just as a Linux file.
- Again, be careful with the "control layer" and "data layer" separation. Master
only has metadata about the chunks; not the actual chunk data.

Read flow:
- Client has a "GFS library". Library turns read offset to chunk index (divide read offset by 64MB).
- Client asks master about chunk index, filename.
- Master replies with chunkserver replicas.
- Client can read from the closest chunkserver.

Master state:
- Directory tree, access permissions per file, chunk ids within a file, list of
chunkservers holdin the chunk.
- A change is considered "committed" only after master has put the change in its
"operation log" and replicated this log. Similar to write-ahead logging in
filesystems and databases.
- Operation log has important information like changes to directory tree, access
control, chunk handles, etc.
- Who owns a chunk need not be in operation log. Master does not maintain an
up-to-date view of who holds the chunks. When master restarts, it asks
chunkservers about their chunks. Somewhat, similar to [[paper-2011-nsdi-ciel]]
master asking workers about objects in their object store.
- When operation log becomes large, create a checkpoint of master state.
Pre-checkpoint logs can be thrown away.

Append flow:
- Client just wants to append some data. It will be appended to the offset
decided by GFS.
- Client asks master about the last chunk id for the filename, its chunkserver
replicas and the "primary chunkserver" who has the lease. Master might give a
lease to one of the replicas if it hasn't already assigned a primary.
- Client sends its append to all the chunkserver replicas who keep these appends
in a temporary buffer. Next, this client asks primary to do the write. Primary
picks the latest offset, does the write from the buffer, and asks all the
replicas to apply the write at the same offset. If everyone is able to write,
then the write is a success, otherwise primary will tell client that their write
has failed and that they should retry.

So, only for "successful appends" the chunkserver replicas have an identical
state. But there maybe intermediate records that were written only to a subset 
of replicas. It is left to the reader to ignore/de-duplicate these intermediate
records. There are two possibilities:
* Because of client-side retry, a record is seen twice. The suggestion is to 
keep a unique record id like webpage URL so that reader can de-duplicate
records.
* Another possibility is that a chunkserver actually did not see the record that
had "failed". For this, the approach is that records are "self verifying"
containing their checksums (similar to IP packets). If the reader reads from a
chunkserver that didn't see the record append, it will have garbage data instead
of a valid record. This garbage data is checked against its checksum. With this, 
reader knows that the record is not valid.
* So "successful appends" are consistent across replicas. "Failed appends" are
inconsistent (Table 1).

What if the primary chunkserver crashes? Master would notice that we are no 
longer receiving heartbeats for this chunkserver. It will slowly clone its
chunks to new replicas to come back up to three replicas per chunk. When the
master recieves an append request, it will assign a different chunkserver
replica as primary to keep the chunk available to clients for record appends.

But, the chunkserver replicas and the primary chunkserver information is cached
in clients to reduce client<>master chit-chat. Clients can use this cached
information to append several records.  What if this information is now stale?
The client `C1` thinks that `CS1` is primary, but master has already made `CS2`
as primary after receiving an append request from client `C2`. Worse yet, master
thought that `CS1` has crashed and was unable to reach to it, but `CS1` is
actually alive and only the network is broken. So, `CS1` does not even know that
the primary has changed.

This is called a "split-brain" situation. There are two active primaries in the 
system. Both of them think they are incharge of handling appends to the same
chunk. Now let us say that `C2` wants to append a record. It puts the record at 
`CS2`, `CS3`, and `CS4`; `CS2` asks to commit the record at offset 23. `CS2`
promises `C2` that commit is successful. Now `C1` tries to append a record.
`CS1` asks `CS2` and `CS3` to append the record to the same offset 23. If we let
this append happen, we permanently lost the record appended by `C2`!

So, we would like `CS2` and `CS3` to reject this append request from `CS1`. The
way GFS does this is to assign a monotonically increasing version number to each
chunk. Every time the master assigns new primary for a chunk, it bumps up the
version number. All the replicas that master is able to talk to (`CS2` and
`CS3`) know the latest version number. If a replica receives an append request
with an earlier version number, then it rejects the request.

For further safety, master makes primaries for only a lease period (default 1
minute). Primaries have to renew their leases with the master in heartbeat
messages. Leases are generally granted. When selecting a new primary, master
waits for the previous primary's lease to expire. After the lease expires, 
master can be sure that there are no outstanding writes. This is because primary
will return error to client if it is no longer the primary (its lease was not
renewed).

Master must persist the new version number in its own log and replicate this log
entry before replying to the client with the new version number. If master
restarts, we do not want to reply with old version numbers for a chunk. In this 
case, the client might successfully append with the old version number on stale 
replicas.

It is also possible that the version number in master's log is older than the
version number known to the chunkservers. This can happen if master sent out
messages to give lease and then it crashed before writing to the log and before
replying to the client.

> If the master sees a version number greater than the one in its records, the
master assumes that it failed when granting the lease and so takes the higher
version to be up-to-date.

Is this dangerous? Could we have lost writes?

No. Because we had not given this new version number to any client (otherwise
the version number must have been in master's log). Therefore, no client could
have written any record with this new version number. Master can hand leases
again and start admitting writes.

So master keeps the following data. nv means non-volatile data. Changes to it 
are put in the operation log. v means volatile. It's ok if we lose this data.

* filename -> array of chunk handles (nv)
* chunk handle -> 
  * list of chunkservers that hold the data (v). Master will talk to chunkservers at reboot to gather chunk information.
  * version # (nv). After a data center power outage, if the chunkservers with
  latest chunk versions come up later, then master could incorrectly admit
  writes to chunkservers with stale chunks.
  * primary (all writes must be co-ordinated by the primary) (v) if master
  forgets who primary is, we can just wait for default lease expiration of 60
  seconds.
  * lease expiration until there is primary (v) same reason, just wait for 60
  seconds 