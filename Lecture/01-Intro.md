# 6.5840 2023 Lecture 1: Introduction

## 6.5840: Distributed Systems Engineering

What I mean by "distributed system":
  a group of computers cooperating to provide a service
  this class is mostly about infrastructure services
    e.g. storage for big web sites, MapReduce, peer-to-peer sharing
  lots of important infrastructure is distributed

Why do people build distributed systems?
  to increase capacity via parallel processing
  to tolerate faults via replication
  to match distribution of physical devices e.g. sensors
  to increase security via isolation

But it's not easy:
  concurrency
  complex interactions
  partial failure
  hard to get high performance

Why study this topic?
  interesting -- hard problems, powerful solutions
  widely used -- driven by the rise of big Web sites
  active research area -- important unsolved problems
  challenging to build -- you'll do it in the labs

## COURSE STRUCTURE

http://pdos.csail.mit.edu/6.5840

Course staff:
  Frans Kaashoek and Robert Morris, lecturers
  Upamanyu Sharma, TA
  Yun-Sheng Chang, TA
  Daniel Wisdom, TA

Course components:
  lectures
  papers
  two exams
  labs
  final project (optional)

Lectures:
  big ideas, paper discussion, lab guidance
  will be recorded, available online

Papers:
  there's a paper assigned for almost every lecture
  research papers, some classic, some new problems, ideas, implementation details, evaluation
  please read papers before class!
  web site has a short question for you to answer about each paper and we ask you to send us a question you have about the paper
submit answer and question before start of lecture

Exams:
  Mid-term exam in class
  Final exam during finals week
  Mostly about papers and labs

Labs:
  goal: deeper understanding of some important ideas
  goal: experience with distributed programming
  first lab is due a week from Friday
  one per week after that for a while

Lab 1: distributed big-data framework (like MapReduce)
Lab 2: fault tolerance using replication (Raft)
Lab 3: a simple fault-tolerant database
Lab 4: scalable database performance via sharding

Optional final project at the end, in groups of 2 or 3.
  The final project substitutes for Lab 4.
  You think of a project and clear it with us.
  Code, short write-up, demo on last day.

Warning: debugging the labs can be time-consuming
  start early
  ask questions on Piazza
  use the TA office hours

We grade the labs using a set of tests
  we give you all the tests; none are secret

## MAIN TOPICS

This is a course about infrastructure for applications.

* Storage.
* Communication.
* Computation.

A big goal: **hide the complexity** of distribution from applications.

### Topic: Fault Tolerance

  1000s of servers, big network -> always something broken
    We'd like to hide these failures from the application.
    "High availability": service continues despite failures
  Big idea: replicated servers.
    If one server crashes, can proceed using the other(s).
    Labs 2 and 3

  Recoverability
  e.g.   recover by human

### Topic: Consistency

  General-purpose infrastructure needs well-defined behavior.
    E.g. "Get(key) yields the value from the most recent Put(key,value)."

    Put(key,value)

    Get(key) -------> value

- Strong / Weak Consistency

Achieving good behavior is hard!
e.g. "Replica" servers are hard to keep identical.

| Server 1 | Server 2 |
| -------- | -------- |
| k = 1    | k = 1    |
| v=20     | v=30     |

### Topic: Performance

  The goal: scalable throughput
    Nx servers -> Nx total throughput via parallel CPU, RAM, disk, net.
  Scaling gets harder as N grows:
    Load imbalance.
    Slowest-of-N latency.
    Some things don't speed up with N: initialization, interaction.

WEB server    ------   DB (bottleneck)

WEB server    ------    |

WEB server    ------    |

  Labs 1, 4

### Topic: Tradeoffs

  Fault-tolerance, consistency, and performance are enemies.
  Fault tolerance and consistency require communication
    e.g., send data to backup server
    e.g., check if cached data is up-to-date
    communication is often slow and non-scalable
  Many designs provide only weak consistency, to gain speed.
    e.g. Get() might *not* yield the latest Put()!
    Painful for application programmers but may be a good trade-off.
  We'll see many design points in the consistency/performance spectrum.

### Topic: Implementation

  RPC, Threads, Concurrency control, Configuration.
  The labs...

This material comes up a lot in the real world.
  All big web sites and cloud providers are expert at distributed systems.
  Many big open source projects are built around these ideas.
  We'll read multiple papers from industry.
  And industry has adopted many ideas from academia.

