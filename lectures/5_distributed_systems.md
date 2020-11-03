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