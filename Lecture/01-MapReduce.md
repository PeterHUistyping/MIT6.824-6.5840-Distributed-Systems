## CASE STUDY: MapReduce

Let's talk about MapReduce (MR)
  a good illustration of 6.5840's main topics
  hugely influential
  the focus of Lab 1

[MapReduce: Simplified Data Processing on Large Clusters Google](../Paper/01-mapreduce.pdf)

### MapReduce Overview

  context: multi-hour computations on multi-terabyte data-sets
    e.g. build search index, or sort, or analyze structure of web
    only practical with 1000s of computers
    applications not written by distributed systems experts
  big goal: easy for non-specialist programmers
    programmer just defines Map and Reduce functions
    often fairly simple sequential code
  MR manages, and hides, all aspects of distribution!

Abstract view of a MapReduce job -- word count
  Input1 -> Map -> a,1 b,1
  Input2 -> Map ->     b,1
  Input3 -> Map -> a,1     c,1
                    |   |   |
                    |   |   -> Reduce -> c,1
                    |   -----> Reduce -> b,2
                    ---------> Reduce -> a,2

1) input is (already) split into M files
2) MR calls Map() for each input file, produces list of k,v pairs
   "intermediate" data
   each Map() call is a "task"
3) when Maps are done,
   MR gathers all intermediate v's for each k,
   and passes each key + values to a Reduce call
4) final output is set of <k,v> pairs from Reduce()s

Word-count-specific code
  Map(k, v)
    split v into words
    for each word w
      emit(w, "1")
  Reduce(k, v_list)
    emit(len(v_list))

MapReduce scales well:
  N "worker" computers (might) get you Nx throughput.
    Maps()s can run in parallel, since they don't interact.
    Same for Reduce()s.
  Thus more computers -> more throughput -- very nice!

MapReduce hides many details:
  sending app code to servers
  tracking which tasks have finished
  "shuffling" intermediate data from Maps to Reduces
  balancing load over servers
  recovering from failures

However, MapReduce limits what apps can do:
  No interaction or state (other than via intermediate output).
  No iteration
  No real-time or streaming processing.

Some details (paper's Figure 1)

Input and output are stored on the GFS cluster file system
  MR needs huge parallel input and output throughput.
  GFS splits files over many servers, in 64 MB chunks
    Maps read in parallel
    Reduces write in parallel
  GFS also replicates each file on 2 or 3 servers
  GFS is a big win for MapReduce

The "Coordinator" manages all the steps in a job.

1. coordinator gives Map tasks to workers until all Maps complete
   Maps write output (intermediate data) to local disk
   Maps split output, by hash(key) mod R, into one file per Reduce task
2. after all Maps have finished, coordinator hands out Reduce tasks
   each Reduce task corresponds to one hash bucket of intermediate output
   each Reduce fetches its intermediate output from (all) Map workers
   each Reduce task writes a separate output file on GFS

What will likely limit the performance?
  We care since that's the thing to optimize.
  CPU? memory? disk? network?
  In 2004 authors were limited by network capacity.
    What does MR send over the network?
      Maps read input from GFS.
      Reduces read Map intermediate output.
        Often as large as input, e.g. for sorting.
      Reduces write output files to GFS.
    [diagram: servers, tree of network switches]
    In MR's all-to-all shuffle, half of traffic goes through root switch.
    Paper's root switch: 100 to 200 gigabits/second, total
      1800 machines, so 55 megabits/second/machine.
      55 is small:  much less than disk or RAM speed.

How does MR minimize network use?
  Coordinator tries to run each Map task on GFS server that stores its input.
    All computers run both GFS and MR workers
    So input is read from local disk (via GFS), not over network.
  Intermediate data goes over network just once.
    Map worker writes to local disk.
    Reduce workers read from Map worker disks over the network.
    Storing it in GFS would require at least two trips over the network.
  Intermediate data partitioned into files holding many keys.
    R is much smaller than the number of keys.
    Big network transfers are more efficient.

How does MR get good load balance?
  Wasteful and slow if N-1 servers have to wait for 1 slow server to finish.
  But some tasks likely take longer than others.
  Solution: many more tasks than worker machines.
    Coordinator hands out new tasks to workers who finish previous tasks.
    No task is so big it dominates completion time (hopefully).
    So faster servers do more tasks than slower ones,
    And all finish at about the same time.

What about fault tolerance?
  What if a worker crashes during a MR job?
  We want to hide failures from the application programmer!
  Does MR have to re-run the whole job from the beginning?
    Why not?
  MR re-runs just the failed Map()s and Reduce()s.

Suppose MR runs a Map twice, one Reduce sees first run's output,
    but another Reduce sees the second run's output?
  Could the two Reduces produce inconsistent results?
  No: Map() must produce exactly the same result if run twice with same input.
    And Reduce() too.
  So Map and Reduce must be pure deterministic functions:
    they are only allowed to look at their arguments/input.
    no state, no file I/O, no interaction, no external communication.

Details of worker crash recovery:

* a Map worker crashes:
  coordinator notices worker no longer responds to pings
  coordinator knows which Map tasks ran on that worker
  those tasks' intermediate output is now lost, must be re-created
  coordinator tells other workers to run those tasks
  can omit re-running if all Reduces have fetched the intermediate data
* a Reduce worker crashes:
  finished tasks are OK -- stored in GFS, with replicas.
  coordinator re-starts worker's unfinished tasks on other workers.

Other failures/problems:

* What if the coordinator gives two workers the same Map() task?
  perhaps the coordinator incorrectly thinks one worker died.
  it will tell Reduce workers about only one of them.
* What if the coordinator gives two workers the same Reduce() task?
  they will both try to write the same output file on GFS!
  atomic GFS rename prevents mixing; one complete file will be visible.
* What if a single worker is very slow -- a "straggler"?
  perhaps due to flakey hardware.
  coordinator starts a second copy of last few tasks.
* What if a worker computes incorrect output, due to broken h/w or s/w?
  too bad! MR assumes "fail-stop" CPUs and software.
* What if the coordinator crashes?

Current status?
  Hugely influential (Hadoop, Spark, &c).
  Probably no longer in use at Google.
    Replaced by Flume / FlumeJava (see paper by Chambers et al).
    GFS replaced by Colossus (no good description), and BigTable.

Conclusion
  MapReduce made big cluster computation popular.

- Not the most efficient or flexible.

+ Scales well.
+ Easy to program -- failures and data movement are hidden.
  These were good trade-offs in practice.
  We'll see some more advanced successors later in the course.
  Have fun with Lab 1!
