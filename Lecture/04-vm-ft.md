# 6.5840 2023 Lecture 4: Primary/Backup Replication

Today
  Primary/Backup Replication for Fault Tolerance
  Case study of VMware FT (2010), an extreme version of the idea

Why read this paper?
  A clean primary/backup design that brings out many issues that come up over and and over this semester

- state-machine replication
- output rule
- fail-over/primary election

  Impressive you can do this at the level of machine instructions
  you can take any application and replicate it VM FT
  all designs later in semester involve work by the application designer

Goal: **High availability**
  even if a machine fails, deliver service
  approach: replication

What kinds of failures can replication deal with?
  Replication is good for *"fail-stop" failure of a single replica*
    fan stops working, CPU overheats and shuts itself down
    someone trips over replica's power cord or network cable
    software notices it is out of disk space and stops
  Replication may not help with bugs or operator error
    Often not fail-stop
    May be correlated (i.e. some input causes all replicas to crash)
  How about earthquake or city-wide power failure?
    Only if replicas are *physically* separated

## Two main replication approaches

### State transfer

    Primary executes the service
    Primary**sends state snapshots** over network to a storage system
    On failure:
      Find a spare machine (or maybe there's a dedicated backup waiting)
      Load software, load saved state, execute

### Replicated State Machine

    Clients send operations to primary,
      primary sequences and sends to backups
    All replicas execute all operations
    If same start state,
      same operations,
      same order,
      deterministic,
      then same end state.

State transfer is conceptually simple
  But state may be large, slow to transfer over network

Replicated state machine often generates less network traffic
  Operations are often small compared to state
  But complex to get right
  VM-FT uses replicated state machine, as do Labs 2/3/4

## Big Questions

  What are the state and operations?
  Does primary have to wait for backup?
  How does backup decide to take over?
  Are anomalies visible at cut-over?
  How to bring a replacement backup up to speed and resume replication?

At what level do we want replicas to be identical?
  Application state, e.g. a database's tables?
    GFS works this way
    Efficient; primary only sends high-level operations to backup
    Application must understand fault tolerance
  Machine level, e.g. registers and RAM content?
    might allow us to replicate any existing application w/o modification!
    requires forwarding of machine events (interrupts, network packets, &c)
    requires "machine" modifications to send/recv event stream...

## Today's paper (VMware FT) replicates machine-level state

  Transparent: can run any existing O/S and server software!
  Appears like a single server to clients

### Overview

  [diagram: app, O/S, VM-FT underneath, disk server, network, clients]
  words:
    hypervisor == monitor == VMM (virtual machine monitor)
    O/S+app is the "guest" running inside a virtual machine
  two physical machines, primary and backup

The basic idea:
  Primary and backup initially start with identical memory and registers
    Including identical software (O/S and app)
  Most instructions execute identically on primary and backup
    e.g. an ADD instruction
  So most of the time, no work is required to cause them to remain identical!

When does the primary have to send information to the backup?
  Any time something happens that might cause their executions to diverge.
  Anything that's not a deterministic consequence of executing instructions.

What sources of divergence must FT eliminate?
  Instructions that aren't functions of state, such as reading current time.
  Inputs from external world -- network packets and disk reads.
    These appear as DMA'd data plus an interrupt.
  Timing of interrupts.
  But not multi-core races, since uniprocessor only.

Why would divergence be a disaster?
  b/c state on backup would differ from state on primary,
    and if primary then failed, clients would see inconsistency.
  Example: the 6.824 homework submission server
    Enforces midnight deadline for labs.
    A hardware timer goes off at midnight.
    Let's replicate submission server with a *broken* FT.
    On primary, my homework packet interrupt arrives just *before* timer goes off.
      Primary will tell me I get full credit for homework.
    On backup, my homework arrives just after, so backup thinks it is late.
      Primary and backup now have divergent state.
      For now, no-one notices, since the primary answers all requests.
    Then primary fails, backup takes over, and course staff see
      backup's state, which says I submitted late!
  So: backup must see same events,
    in same order,
    at same points in instruction stream.

The logging channel
  primary sends all events to backup over network
    "logging channel", carrying log entries
    interrupts, incoming network packets, data read from shared disk
  FT provides backup's input (interrupts &c) from log entries
  FT suppresses backup's network output
  if either stops being able to talk to the other over the network
    "goes live" and provides sole service
    if primary goes live, it stops sending log entries to the backup

Each log entry: instruction #, type, data.

FT's handling of timer interrupts
  Goal: primary and backup should see interrupt at exactly
        the same point in the instruction stream
  Primary:
    FT fields the timer interrupt
    FT reads instruction number from CPU
    FT sends "timer interrupt at instruction # X" on logging channel
    FT delivers interrupt to primary, and resumes it
    (relies on CPU support to direct interrupts to FT software)
  Backup:
    ignores its own timer hardware
    FT sees log entry *before* backup gets to instruction # X
    FT tells CPU to transfer control to FT at instruction # X
    FT mimics a timer interrupt that backup guest sees
    (relies on CPU support to jump to FT after the X'th instruction)

FT's handling of network packet arrival (input)
  Primary:
    FT configures NIC to write packet data into FT's private "bounce buffer"
    At some point a packet arrives, NIC does DMA, then interrupts
    FT gets the interrupt, reads instruction # from CPU
    FT pauses the primary
    FT copies the bounce buffer into the primary's memory
    FT simulates a NIC interrupt in primary
    FT sends the packet data and the instruction # to the backup
  Backup:
    FT gets data and instruction # from log stream
    FT tells CPU to interrupt (to FT) at instruction # X
    FT copies the data to guest memory, simulates NIC interrupt in backup

Why the bounce buffer?
  We want the data to appear in memory at exactly the same point in
    execution of the primary and backup.
  So they see the same thing if they read packet memory before interrupt.
  Otherwise they may diverge.

FT VMM emulates a local disk interface
  but actual storage is on a network server -- the "shared disk"
  all files/directories are in the shared storage; no local disks
  only primary talks to the shared disk
    primary forwards blocks it reads to the backup
    backup's FT ignores backup app's writes, serves reads from primary's data
  shared disk makes creating a new backup much faster
    don't have to copy primary's disk

The backup must lag by one log entry
  Suppose primary gets an interrupt at instruction # X
  If backup has already executed past X, it is too late!
  So backup FT can't execute unless at least one log entry is waiting
    Then it executes just to the instruction # in that log entry
    And waits for the next log entry before resuming

Example: non-deterministic instructions
  some instructions yield different results even if primary/backup have same state
  e.g. reading the current time or processor serial #
  Primary:
    FT sets up the CPU to interrupt if primary executes such an instruction
    FT executes the instruction and records the result
    sends result and instruction # to backup
  Backup:
    FT reads log entry, sets up for interrupt at instruction #
    FT then supplies value that the primary got, does not execute instruction

What about output (sending network packets, writing the shared disk)?
  Primary and backup both execute instructions for output
  Primary's FT actually does the output
  Backup's FT discards the output

Output example: DB server
  clients can send "increment" request
    DB increments stored value, replies with new value
  so:
    [diagram]
    suppose the server's value starts out at 10
    network delivers client request to FT on primary
    primary's FT sends on logging channel to backup
    FTs deliver request packet to primary and backup
    primary executes, sets value to 11, sends "11" reply, FT really sends reply
    backup executes, sets value to 11, sends "11" reply, and FT discards
    the client gets one "11" response, as expected

But wait:
  suppose primary sends reply and then crashes
    so client gets the "11" reply
  AND the logging channel discards the log entry w/ client request
    primary is dead, so it won't re-send
  backup goes live
    but it has value "10" in its memory!
  now a client sends another increment request
    it will get "11" again, not "12"
  oops

Solution: the Output Rule (Section 2.2)
  before primary sends output (e.g. to a client, or shared disk),
  must wait for backup to acknowledge all previous log entries

Again, with output rule:
  [diagram]
  primary:
    receives client "increment" request
    sends client request on logging channel
    about to send "11" reply to client
    first waits for backup to acknowledge previous log entry
    then sends "11" reply to client
  suppose the primary crashes at some point in this sequence
  if before primary receives acknowledgement from backup
    maybe backup didn't see client's request, and didn't increment
    but also primary won't have replied
  if after primary receives acknowledgement from backup
    then client may see "11" reply
    but backup guaranteed to have received log entry w/ client's request
    so backup will increment to 11

The Output Rule is a big deal
  Occurs in some form in most strongly consistent replication systems
    Often called "synchronous replication" b/c primary must wait
  A serious constraint on performance
  An area for application-specific cleverness
    Eg. maybe no need for primary to wait before replying to read-only operation
  FT has no application-level knowledge, must be conservative

Q: What if the primary crashes just after getting acknowledgement from backup,
   but before the primary emits the output?
   Does this mean that the output won't ever be generated?

A: The backup goes live either before or after the instruction that
     sends the reply packet to the client.
   If before, it will send the reply packet.
   If after, FT will have discarded the packet. But the backup's
     TCP will think it sent it, and will expect a TCP ACK packet, and
     will re-send if it doesn't get the ACK.

Q: But what if the primary crashed *after* emitting the output?
   Will the backup emit the output a *second* time?

A: It might!
   OK for TCP, since receivers ignore duplicate sequence numbers.
   OK for writes to shared disk,
     since backup will write same data to same block #.

Duplicate output at cut-over is pretty common in replication systems
  Clients need to keep enough state to ignore duplicates
  Or be designed so that duplicates are harmless
  VM FT gets duplicate detection "for free",
    TCP state is duplicated on backup by VM FT

Q: Does FT cope with network partition -- could it suffer from split brain?
   E.g. if primary and backup both think the other is down.
   Will they both go live?

A: The shared disk server breaks the tie.
   Disk server supports atomic test-and-set.
   If primary or backup thinks other is dead, attempts test-and-set.
   If only one is alive, it will win test-and-set and go live.
   If both try, one will lose, and halt.

The shared disk server needs to be reliable!
  If disk server is down, service is down
  They have in mind an expensive fault-tolerant disk server

Q: Why don't they support multi-core?

Performance (table 1)
  FT/Non-FT: impressive!
    little slow down
  Logging bandwidth
    Directly reflects disk read rate + network input rate
    18 Mbit/s is the max
  The logging channel traffic numbers seem low to me
    Applications can read a disk at a few 100 megabits/second
    So their applications may not be very disk-intensive

When might FT be attractive?
  Critical but low-intensity services, e.g. name server.
  Services whose software is not convenient to modify.

What about replication for high-throughput services?
  People use application-level replicated state machines for e.g. databases.
    The state is just the DB, not all of memory+disk.
    The events are DB commands (put or get), not packets and interrupts.
    Can have short-cuts for e.g. read-only operations.
  Result: less logging traffic, fewer Output Rule pauses.
  GFS use application-level replication, as do Lab 2 &c

## Summary

  Primary-backup replication
    VM-FT: clean example
  How to cope with partition without single point of failure?
    Next lecture
  How to get better performance?
    Application-level replicated state machines

---

VMware KB (#1013428) talks about multi-CPU support.  VM-FT may have switched
from a replicated state machine approach to the state transfer approach, but
unclear whether that is true or not.

http://www.wooditwork.com/2014/08/26/whats-new-vsphere-6-0-fault-tolerance/

http://www-mount.ece.umn.edu/~jjyi/MoBS/2007/program/01C-Xu.pdf

http://web.eecs.umich.edu/~nsatish/abstracts/ASPLOS-10-Respec.html
