# Shared Memory

![placeholder]()

## Table of Contents
* [Introduction](#introduction)
* [Synchronizaton]()
* [Communication]()
* [Lightweight RPC]()
* [Scheduling]()
* [Shared Memory Multiprocessor OS]()
* [Barrier Synchronization]()

## Introduction

Parallelism has become fundamental to computing systems, any general purpose CPU has multiple cores, any general purpose OS is designed to take advantage of these hardware features.

In this unit we will discuss the algorithms needed to synchronize, schedule and communicate in a shared memory multiprocessor.

Let's look at some possible models of a shared memory machine.

The first architecture we will look at is **Dancehall Architecture**. In this model, CPUs are on one side of an interconnection network, and memory on the other side.

<img src="resources/4_shared_memory/dancehall.png">

The commonalities between the three structures we are going to dicuss is that there are going to be **CPUs, memory, and an interconnection network**. 

A **shared memory system** means that the **entire address space is accessible by any of the CPUs**. 

<img src="resources/4_shared_memory/smp.png">

The next style is the **SMP(symmetric) architecture**. The interconnection network has been simplified to a system bus. This architecture is symmetric because the **access time of any of the CPUs to memory is the same**.

<img src="resources/4_shared_memory/dsm.png">

The third style is a **distributed shared memory (DSM) architecture**. What you have is **memory that is associated with each CPU**. Each CPU is able to access all of the memories through the interconnection network, it is just that the access to **memory close to the CPU is faster ti access**, than the memory accessed through the interconnection network.s

### Shared Memory Caching

A **cache in a shared memory** architecture shares the exact **same purpose as a cache in a uniprocessor system**: it **stores memory close to the CPU** for quick reference. 

If the memory doesn't exist in the cache, then the CPU fetches the data from main memory.

There is one **unique problem in multiprocessor systems**: the fact that there are private caches associated with each CPU, and the memory is shared amongst the processors. How does the **system handle cache invalidation**?

This is referred to as the **cache coherence** problem. Who ensures this consistency? There is a partnership between hardware and software, they agree on a **memory-consistency model**.

The memory-consistency model is a **contract between hardware and software as to what behavior a programmer should expect when a multi-threaded application is run on the architecture**. 

<img src="resources/4_shared_memory/sh_mem_quiz1.png">

* `c = d = 0` - Process p2 runs first. This is possible.
* `c = d = 1` - Process p2 run after both the instructions on p1 have executed. This is also possible.
* `c = 1, d = 0` - The first instruction of p2 is run, then both instructions in p1, then p2 finishes. This is also possible.
* `c = 0, d = 1` - While seemingly impossible, this is actually possible. How? Remember that the processes on each CPU are connected via an interconnection network, if the messages are received out-of-order, then this outcome is possible. This is non-intuitive, so memory-consistency models typically disallow this type of behavior.

### Memory Consistency Models

The memory-consistency model is a **contract between hardware and software as to what behavior a programmer should expect when a multi-threaded application is run on the architecture**. 

There are many possible different models, let's talk about one, the **sequential consistency model**.

<img src="resources/4_shared_memory/seq_consistency.png">

One expectation that a programmer has is that the **memory accesses** within an individual process are going to **happen in the order that the program is written**. 

Additionally, there is going to be **arbitrary interleaving** of memory accesses between the two processes.

A good physical analogy of this model is shuffling two halves of a card deck together. There is no guarantee over how the cards get interleaved, but you know they will be in the same order that they started out in when they were split.

Let's go back to our original example from the quiz, under the sequential consistency model the last outcome where the messages along the interconnection are received out-of-order would not be possible. This is comforting. 

### Cache Coherence

**How are memory consistency models implemented efficiently?** This brings us back to **cache coherence**, the implementation of this model in the presence of private caches. 

If the hardware only gives shared address space, but no way of determining whether the caches are consistent, then this system must have the software (the OS) guarantee the contract of the memory consistent model. This is a non-cache coherent shared address space multiprocessor (**ncc shared memory multiprocessor**).

Another option is that the hardware does everything for you, this is a cache coherent multiprocessor (**cc multiprocessor**).

Let's focus on the **hardware implementing cache coherence**. There are two possibilities: **write invalidate or write update**. 


