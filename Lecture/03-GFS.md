# 6.5840 2023 Lecture 3: GFS

The Google File System
Sanjay Ghemawat, Howard Gobioff, and Shun-Tak Leung
SOSP 2003

## Why are we reading this paper?

  GFS paper touches on many themes of 6.824
    parallel Performance -> Sharding
    Faults -> Tolerance -> Replication -> Consistency -> Low Performance

  good systems paper -- details from apps all the way to network
  successful real-world design

## GFS context

  Many Google services needed a big fast unified storage system
    Mapreduce, crawler, indexer, log storage/analysis
  Shared among multiple applications e.g. crawl, index, analyze
  Huge capacity
  Huge performance
  Fault tolerant
  But:
    just for internal Google use
    aimed at batch big-data performance, not interactive

## GFS overview

  100s/1000s of clients (e.g. MapReduce worker machines)
  100s of chunkservers, each with its own disk
  one coordinator

Capacity story?
  big files split into 64 MB chunks
  each file's chunks striped/sharded over chunkservers
    so a file can be much larger than any one disk
  each chunk in a Linux file

Throughput story?
  clients talk directly to chunkservers to read/write data
  if lots of clients access different chunks, huge parallel throughput
  read or write

Fault tolerance story?
  each 64 MB chunk stored (replicated) on three chunkservers
  client writes are sent to all of a chunk's copies
  a read just needs to consult one copy

### What are the steps when client C wants to read a file?

1. C sends filename and offset to coordinator (CO) (if not cached)
   CO has a filename -> array-of-chunkhandle table
   and a chunkhandle -> list-of-chunkservers table
2. CO finds chunk handle for that offset (cached)
3. CO replies with chunkhandle + list of chunkservers
4. C caches handle + chunkserver list
5. C sends request to nearest chunkserver
   chunk handle, offset
6. chunk server reads from chunk file on disk, returns to client

Clients only ask coordinator where to find a file's chunks
  clients cache name -> chunkhandle info
  coordinator does not handle data, so (hopefully) not heavily loaded

### What about writes?

  Client knows which chunkservers hold replicas that must be updated.
  How should we manage updating of replicas of a chunk?

A BAD replication scheme
  (This is *not* what GFS does)
  [diagram: C, S1, S2, S3]
  Client sends update to each replica chunkserver
  Each chunkserver applies the update to its copy

What can go wrong?
  *Two* clients write the same data at the same time
    i.e. "concurrent writes"
    Chunkservers may see the updates in different orders!
    Again, the risk is that, later, two clients may read different content

Idea: primary/secondary replication
  (or primary/backup)
  For each chunk, designate one server as "primary".
  Clients send write requests just to the primary.
    The primary **alone** manages interactions with secondary servers and chooses the order for all client writes. (Some designs send reads just to primary, some also to secondaries)
    Tells the secondaries -- with sequence numbers -- so all replicas apply writes**in the same order**, even for concurrent client writes.
  There are still many details to fill in, and we'll
 see a number of variants in upcoming papers.

What are the steps when C wants to write a file at some offset?
  paper's Figure 2

1. C asks CO about file's chunk @ offset
2. CO tells C the primary and secondaries
3. C sends data to all (just temporary...), waits for all replies (?)
4. C asks P to write
5. P checks that lease hasn't expired
6. P writes its own chunk file (a Linux file)
7. P tells each secondary to write (copy temporary into chunk file)
8. P waits for all secondaries to reply, or timeout
   secondary can reply "error" e.g. out of disk space
9. P tells C "ok" or "error"
10. C retries from start if error

GFS guarantees to applications -- consistency (Table 1)
  somewhat complex!
  If primary tells client that a write succeeded,
    and no other client is writing the same part of the file,
    all readers will see the write.
    "defined"
  If successful concurrent writes to the same part of a file,
    and they all succeed,
    all readers will see the same content,
    but maybe it will be a mix of the writes.
    "consistent"
    E.g. C1 writes "ab", C2 writes "xy", everyone might see "axyb".
  If primary doesn't tell the client that the write succeeded,
    different readers may see different content, or none.
    "inconsistent"

How can inconsistent content arise?
  Primary P updated its own state.
  But secondary S1 did not update (failed? slow? network problem?).
  Client C1 reads from P; Client C2 reads from S1.
    They will see different results!
  Such a departure from ideal behavior is an "anomaly".
  But note that in this case the primary would have returned
    an error to the writing client.

How can consistent but undefined arise?
  Clients break big writes into multiple small writes,
  e.g. at chunk boundaries, and GFS may interleave them if concurrent client writes.

How can duplicated data arise?
  Clients re-try record appends.

Why are these anomalies OK?
  They only intended to support a certain subset of their own applications.
    Written with knowledge of GFS's behavior.
  Probably mostly single-writer and Record Append.
  Writers could include checksums and record IDs.
    Readers could use them to filter out junk and duplicates.
  Later commentary by Google engineers suggests that it
    might have been better to make GFS more consistent.
    http://queue.acm.org/detail.cfm?id=1594206

What might better consistency look like?
  There are many possible answers.
  Trade-off between easy-to-use for client application programmers,
    and easy-to-implement for storage system designers.
  Maybe try to mimic local disk file behavior.
  Perhaps:
    * atomic writes: either all replicas are updated, or none,
      even if failures.
    * read sees latest write.
    * all readers see the same content (assuming no writes).
  We'll see more precision later.

Let's think about how GFS handles crashes of various entities.

A client crashes while writing?
  Either it got as far as asking primary to write, or not.

A secondary crashes just as the primary asks it to write?

1. Primary may retry a few times, if secondary revives quickly
   with disk intact, it may execute the primary's request
   and all is well.
2. Primary gives up, and returns an error to the client.
   Client can retry -- but why would the write work the second time around?
3. Coordinator notices that a chunkserver is down.
   Periodically pings all chunk servers.
   Removes the failed chunkserver from all chunkhandle lists.
   Perhaps re-replicates, to maintain 3 replicas.
   Tells primary the new secondary list.

Re-replication after a chunkserver failure may take a Long Time.
  Since a chunkserver failure requires re-replication of all its chunks.
  80 GB disk, 10 MB/s network -> an hour or two for full copy.
  So the primary probably re-tries for a while,
    and the coordinator lets the system operate with a missing
    chunk replica,
    before declaring the chunkserver permanently dead.
  How long to wait before re-replicating?
    Too short: wasted copying work if chunkserver comes back to life.
    Too long: more failures might destroy all copies of data.

What if a primary crashes?
  Remove that chunkserver from all chunkhandle lists.
  For each chunk for which it was primary,
    wait for lease to expire,
    grant lease to another chunkserver holding that chunk.

What is a lease?
  Permission to act as primary for a given time (60 seconds).
  Primary promises to stop acting as primary before lease expires.
  Coordinator promises not to change primaries until after expiration.
  Separate lease per actively written chunk.

Why are leases helpful?
  The coordinator must be able to designate a new primary if the present primary fails.
  But the coordinator cannot distinguish "primary has failed" from
    "primary is still alive but the network has a problem."
  What if the coordinator designates a new primary while old one is active?
    Two active primaries!
    C1 writes to P1, C2 reads from P2, doesn't seen C1's write!
    called "split brain" -- a disaster
  Leases help prevent split brain:
    Coordinator won't designate new primary until the current one is
    guaranteed to have stopped acting as primary.

What if the coordinator crashes?
  Two strategies.

1. Coordinator writes critical state to its disk.
   If it crashes and reboots with disk intact,
   re-reads state, resumes operations.
2. Coordinator sends each state update to a "backup coordinator",
   which also records it to disk; backup coordinator can take
   over if main coordinator cannot be restarted.

What information must the coordinator save to disk to recover from crashes?
  Table mapping file name -> array of chunk handles.
  Table mapping chunk handle -> current version #.
  What about the list of chunkservers for each chunk?
    A rebooted coordinator asks all the chunkservers what they store.
  A rebooted coordinator must also wait one lease time before
    designating any new primaries.

* Who/what decides the coordinator is dead, and chooses a replacement?
  Paper does not say.
  Could the coordinator replicas ping the coordinator,
  and automatically take over if no response?
* Suppose the coordinator reboots, and polls chunkservers.
  What if a chunkserver has a chunk, but it wasn't a secondary?
  I.e. the current primary wasn't keeping it up to date?
  Coordinator remembers version number per chunk, on disk.
  Increments each time it designates a new primary for the chunk.
  Chunkserver also remembers its version number per chunk.
  When chunkserver reports to coordinator, coordinator compares
  version number, only accepts if current version.
* What if a client has cached a stale (wrong) primary for a chunk?
* What if the reading client has cached a stale server list for a chunk?
* What if the primary crashes before sending append to all secondaries?
  Could a secondary that *didn't* see the append be chosen as the new primary?
  Is it a problem that the other secondary *did* see the append?

What would it take to have no anomalies -- strict consistency?
  I.e. all clients see the same file content.
  Too hard to give a real answer, but here are some issues.

* All replicas should complete each write, or none -- "atomic write".
  Perhaps tentative writes until all promise to complete it?
  Don't expose writes until all have agreed to perform them!
* Primary should detect duplicate client write requests.
* If primary crashes, some replicas may be missing the last few ops.
  They must sync up.
* Clients must be prevented from reading from stale ex-secondaries.
  You'll see solutions in Labs 2 and 3!
* Are there circumstances in which GFS will break its guarantees?
  e.g. write succeeds, but subsequent readers don't see the data.
  All coordinator replicas permanently lose state (permanent disk failure).
  Read will fail.
  All chunkservers holding the chunk permanently lose disk content.
  Read will fail.
  CPU, RAM, network, or disk yields an incorrect value.
  checksum catches some cases, but not all
  Read may say "success" but yield the wrong data!
  Above errors were "fail-stop", but this is a "byzantine" failure.
  Time is not properly synchronized, so leases don't work out.
  So multiple primaries, maybe write goes to one, read to the other.
  Again, read may yield "success" but wrong data -- byzantine failure.

Performance (Figure 3)
  large aggregate throughput for read
    94 MB/sec total for 16 clients + 16 chunkservers
      or 6 MB/second per client
      is that good?
      one disk sequential throughput was about 30 MB/s
      one NIC was about 10 MB/s
    Close to saturating inter-switch link's 125 MB/sec (1 Gbit/sec)
    So: multi-client scalability is good
    Table 3 reports 500 MB/sec for production GFS, which was a lot
  writes to different files lower than possible maximum
    authors blame their network stack (but no detail)
  concurrent appends to single file
    limited by the server that stores last chunk
  hard to interpret after 15 years, e.g. how fast were the disks?

Retrospective interview with GFS engineer:
  http://queue.acm.org/detail.cfm?id=1594206
  file count was the biggest problem
    eventual numbers grew to 1000x those in Table 2 !
    hard to fit in coordinator RAM
    coordinator scanning of all files/chunks for GC is slow
  1000s of clients -> too much CPU load on coordinator
  coordinator fail-over initially manual, 10s of minutes, too long.
  applications had to be designed to cope with GFS semantics
    and limitations.
    more painful than expected.
  BigTable is one answer to many-small-files problem
  and Colossus apparently shards coordinator data over many coordinators

## Summary

  case study of performance, fault-tolerance, consistency
    specialized for MapReduce applications
  good ideas:
    global cluster file system as universal infrastructure
    separation of naming (coordinator) from storage (chunkserver)
    sharding for parallel throughput
    huge files/chunks to reduce overheads
    primary to choose order for concurrent writes
    leases to prevent split-brain
  not so great:
    single coordinator performance
      ran out of RAM and CPU
    chunkservers not very efficient for small files
    lack of automatic fail-over to coordinator replica
    maybe consistency was too relaxed
