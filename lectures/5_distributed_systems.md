# Distributed Systems

## Table of Contents
* [Introduction](#introduction)
    * [Defining Distributed Systems](#defining-distributed-systems)
    * [Happened Before Relationship](#happened-before-relationship)
* [Lamport's Clock](#lamports-clock)

## Introduction

What fundamentally distinguishes a distributed system from a parallel system is the **individual autonomy of the nodes** of a distributed system. The interconnection network that connects all the nodes is wide-open to th world, as opposed to just being connected inside a box or through a rack.

Many of the issues that are considered to be just within the domain of distributed systems are now surfacing even within the problem space of a single chip. This makes the useful to study.

### Defining Distributed Systems

There are a couple of definitions of distributed systems. Let's touch on a couple of useful ideas.

<img src="resources/5_distributed/definition.png">

A distributed system is **a collection of nodes connected by a local area network or wide area network**. The network can be implemented through twisted pair coaxial cable, optical fibers, or even satellite links. The media access protocols used by the notes may be ATM, ethernet, etc. 

There is **no physical memory that are shared between nodes in a distributed system**. This means that the only way nodes can communicate is by **sending messages** to each other. 

Lastly, the third property is that the **event computation time**, the time taken on a single node to do some meaningful unit of work, is that the **time for communication between nodes is much larger than event computation time**. 

Leslie Lamport gives us a definition of distributed systems: **a system is distributed if the message transmission time is not negligible to the time between events in a single process.**

What's interesting about this definition is that clusters can even be considered distributed systems because processing time is much faster than event communication in a rack.

There are some core tenets of distributed systems:
* Processes are sequential, the events are totally ordered and that a message has to be sent before it is received. 

### Happened Before Relationship

<img src="resources/5_distributed/happened_before.png">

**A -> B**, "A" happened before "B". What this notation implies is that either A and B are events on the same process and that A happened before B on the same process. Or that they are on different processes, then there must be a communication event that connects A and B.  

There is also **transitivity** between happened-before relationships. If A happened before B, and B happened before C, then A happened before C. 

There is another concept that needs to be explored, **concurrent events**. Concurrent events are **events where there is no apparent relationship between events**. This means it is impossible to get a total ordering of events in a distributed system. This means it is important to think about how events are structured/related when designing a distributed algorithm.

Let's look at an example of a real-world distributed system and the associated events.

<img src="resources/5_distributed/distributed_example.png">

## Lamport Clocks

**What does each node in the distributed system know?**
* It's own events. 
* It's communication events with the rest of the nodes in the system.

<img src="resources/5_distributed/lamports_logical_clock.png">

**Lamport's logical clock** builds on this idea. We want to **associate a timestamp with every event** that happens in every process in the system. 

How is this accomplished? **A counter is associated with each node**, when an event happens, the event is associated with the current value of the counter. The counter is monotonically increased for each event. 

Communication events are more complicated. The receipt of a message is considered an event. Which timestamp does the receiving node pay attention to? The timestamp associated with the sending node, or the local counter timestamp?

A function is used to determine the timestamp. **The timestamp is set as the maximum value between the current timestamp and the received timestamp incremented by some monotonic quantity**.

So how do we give timestamps for concurrent events?

### Logical Clock Conditions

<img src="resources/5_distributed/lamport_conditions.png">

Concurrent events have arbitrary timestamps. Also, just because a timestamp for `x` is less than the timestamp for `y` , it does not mean that `x` -> `y` (x happened before y). 

**Is partial order good enough for constructing deterministic distributed algorithms?**

It turns out it is sufficient for certain situations. However, there are some situations where total order is needed. Let's look at a real world example. 

Say you live in a family and you have a car that you need to reserve. To reserve it, you send out a group text with a timestamp. The earliest timestamp wins, if there is a tie, the oldest member of the family wins.

### Lamport's Total Order

<img src="resources/5_distributed/lamports_total_order.png">

If there are two events, `A` and `B` in a distributed system, where `A` happens on node `I` and `B` happens on node `J`. 

If we want to assert that `A` is totally ordered ahead of `B`, then we have to prove that either the timestamp associated with `A` is less than the timestamp associated with `B`, or that if the timestamp happens to be the same, there is some other arbitrary function that can be used to break a tie between the two identical timestamps and that `A` is less than `B` when evaluated with this function.

### Distributed Mutual Exclusion Lock Algorithm

Let's put Lamport's clock to work by using it to implement an algorithm. 

In a distributed system we don't have shared memory to host a lock implementation. We will use Lamport's clock instead. 

Any process that needs to acquire a lock is going to send a message to all the other processes. The intent to acquire a lock may emanate simultaneously from different processes.

<img src="resources/5_distributed/distributed_lock.png">

The algorithm is as follows, every process has a **queue data structure that is ordered by the happened-before relationship**. To request a lock, a process is going to **send a message to all the other processes with its local timestamp**. All the peers are going **stick the request into their local queue** in the appropriate place. It then **acknowledges the request to its peers**. 

A tie is broken by the process that has a lower process ID. At some point, it is possible that the queues are not equivalent, this is because messages take time to travel to the other processes across the network. 

Two things have to be true for the process to know it has the lock.

1. The request for the lock is at the top of the process's queue. 
2. The process has received ACKs from the other processes. Alternatively, all the requests that the process has received are later than the current lock request.

The lock is **released by sending an unlock message**. This removes the entry from the current processes queue, and as soon as the other processes receive the unlock message, they will also remove the entry. 

Here's a quiz. 

<img src="resources/5_distributed/distributed_lock_quiz.png">

The **message complexity for this algorithm is 3(N-1)**. 

1. There are N-1 messages request messages sent.
2. There are N-1 ACK messages sent back to the requesting process.
3. There are N-1 unlock messages sent. 

One good question to ask is **can we do better**?

Yes, we can defer ACKs if the lock requests from the peers are greater than the current requested timestamp. In such a case, the requesting process can acquire the lock and send the ACK message with the unlock request. Under these conditions the complexity is 2(N-1).

### Real World Scenarios

In some instances, the logical/virtual clock is not appropriate or sufficient. These anomalies occur due to individual clock drift and also relative drift between two clocks in different processors.

<img src="resources/5_distributed/lamports_physical_clock.png">


This brings us to **Lamport's physical clock**. In physical/real time, event `A` happens before `B` then we need to provide two guarantees or conditions. 

1. The first condition is that there is a bound on individual clock drift. 
2. There is a bound on mutual drift. 

Let's now look at the relationship between interprocess communication time and invidual/mutual clock drift. 
