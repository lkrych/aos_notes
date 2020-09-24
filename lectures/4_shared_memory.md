# Shared Memory

![cat eating corn](https://media.giphy.com/media/aQUGAeZ1fBWpy/giphy.gif)

## Table of Contents
* [Introduction](#introduction)
    * [Shared Memory Caching](#shared-memory-caching)
    * [Memory Consistency Models](#memory-consistency-models)
    * [Cache Coherence](#cache-coherence)
* [Synchronizaton](#synchronization)
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

<img src="resources/4_shared_memory/write_invalidate.png">

In **write invalidate**, if a particular memory location is contained in all the caches, and if a processor decides to write to said memory location in the cache, then the **hardware will ensure that all the other references in the other caches are invalidated**. It does this by broadcasting a signal on the system bus. The caches snoop on the system bus to watch out for these kinds of signals.

<img src="resources/4_shared_memory/write_update.png">

In **write update**, if a particular CPU writes to the memory location in its cache, then the **hardware sends an update signal on the bus with the new value for that memory location**. 

One thing that is important to remember is that no matter what scheme is used, there is work to be done by the hardware. Overhead grows with an increasing number of processors. So while is is reasonable to expect that increasing the number of processors will increase the performance of a system in general, it is not in a linear fashion because of these overheads.

<img src="resources/4_shared_memory/scalability.png">

Limit the amount of memory you share across machines in parallel system to have efficient code.

## Synchronization

Synchronization primitives are key for parallel programming. A **lock** is an **entity that allows a thread to make sure it is has exclusive access to shared data**. 

Locks come in two flavors:
1. **Exclusive Lock** - which means the lock can only be used by one thread at a time.
2. **Shared Lock** - simultaneous threads can access the data at the same time. This happens a lot with read access for data entries. 

Another kind of synchronization primitive that is very popular in multithreaded parallel programming is a **barrier**. The idea behind this primitive is that there are multiple threads and they are all doing some computation in parallel. At some point, they want to get back together and check up with each other to see what they've been up to. This means that threads need to wait at the barrier until the other threads have shown up.

<img src="resources/4_shared_memory/barrier.png">

A physical analogy to this idea is a restaurant that only seats you if all members of your party have arrived.

### Atomic Operations

Let's look at a simple implementation of an exclusive lock. 

<img src="resources/4_shared_memory/simple_lock.png">

**Is it possible to implement this with atomic reads and writes alone?**

Let's look at the lock acquisition code to answer this question.

```c
if (l == 0) {
    l = 1;
}
```
There are three instructions that need to be performed here:

1. Read `l` from memory.
2. Compare `l` to 0. 
3. Set `l` to 1.

We know that reads and writes are *individually* atomic. But, these three instructions **need to be executed together (atomically)** to ensure that different threads don't interfere with each other's lock acquisition.

Therefore **reads and writes are not sufficient to implement this lock algorithm**.

What is needed is a **read-modify-write atomic instruction**. There are several flavors of these instructions.

1. **Test-and-set** - Takes a memory location as an argument, it returns the current value at that memory location and sets the current value to 1. 
2. **Fetch-and-inc** - Takes a memory location as an argument, it returns the current value at that memory location and increments the current value by 1. 

**Generically**, these **read-modify-write instructions** are known as **fetch-and-phi instructions** because they fetch a value and modify it in some way. 

### Mutex Lock Algorithms

Let's discuss some scalability issues with synchronization primitives in a shared memory processor. The source of inefficiencies in both barrier and lock algorithms are
1. **latency** (time spent by a thread to acquire a lock)
2. **waiting time** (how long do I wait to obtain a busy lock?)
3. **contention** (when a lock is freed, how long does it take in the presence of contention for a winner to emerge with a lock?). 

### Naive Spinlock

Let's start our discussion with the simplest/naive lock. This lock is called the **spin lock** because the processor has to actively wait on the lock when it needs to acquire it. The lock will be implemented with a test-and-set instruction.

We have a shared memory location, `l`, that can have one of two possible values, locked or unlocked. 

<img src="resources/4_shared_memory/naive_spinlock.png">

To acquire a lock a thread spins on the test-and-set operation. To unlock it, the thread that has the lock sets the value to unlocked. 

**What are the problems with this naive spinlock?**
* **too much contention** - each thread is spinning on the test-and-set instruction on that shared memory value. 
* **does not exploit caches** - every CPU in a shared memory architecture has its own cache. It is often the case that these caches are kept coherent by the hardware. Caches are used to make the execution faster. However, there is an **issue with the test-and-set instruction.** Test-and-set cannot use the cached value, because by definition it has to make sure that the memory value it is operating on is modified atomically. It will bypass the cache and go to memory.
* **disrupts useful work** -  When a processor releases the lock, that processor is prevented from doing work by the contention of the other processors. 

### Caching Spinlock

Let's look at how we can exploit the CPU caches. The problem with the previous approach is that the **test-and-set instruction HAS to go to memory**. The implementation of the naive spinlock involved spinning on this instruction: when we wanted to acquire the lock, we had to execute this instruction that goes to memory.

<img src="resources/4_shared_memory/caching_spinlock.png">

Well what about if we modify this a bit. What if the threads that have to wait on the lock rely on the cache instead of going to memory every time? This strategy is called **spin-on-read**. The assumption about the architecture here is that it provides cache coherence (and thus that all the caches reflect what is in main memory). 

So, lets have the waiters spin locally on the cached value. Practically this means that instead of having a test-and-set operation be in the tight loop, we just have a normal read instruction be in the loop. So if the value doesn't exist in the cache, it is fetched from memory and placed in the local cache and now the instructions can just spin on this cached value. **The released lock will be detected via the cache coherence mechanisms of the hardware**. 

Let's talk about one of the inefficiencies of this schema.  When a lock is released, all of the processors are going to do a test-and-set at the same time. We know that this instruction bypasses the cache and goes into main memory. This is going to create A LOT of chatter on the system bus. In a write-invalidate cache mechanism, there will be an O(n^2) bus transactions. Because every CPU will invalidate the cache.

### Spinlocks with delay

In order to **limit the amount of contention** on the system bus when a lock is released we are going to use a **delay**.

Each process will delay asking for the lock immediately even though they know that it has been unlocked. You could liken this behavior to pulling your car off the road during rush hour and going for a walk. 

**Static delays** aren't a great strategy because there can be wasted cycles. 

<img src="resources/4_shared_memory/spinlock_delay.png">

Let's talk about two different delay alternatives:

1. **Delay after lock release**  - uses static delays
2. **Delay with exponential backoff** - uses dynamic delays, in this schema you delay immediately after checking the lock. This delay is some small number at first, but it increases exponentially.

One of the nice thing about the exponential delay algorithm is that we are not using the caching at all. This means the algorithm will work on a non-cache coherent multiprocessor architecture. 

Generally speaking, if there is a lot of contention, then static delay will be a better strategy than exponential backoff.

### Ticket Lock

Up until now we've discussed strategies for reducing latency in acquiring a lock, and the contention when the lock is released.

We haven't talked about **fairness**, **giving the lock out to the threads that asked for it first**. Not the line cutters. This is impossible in the spinlock strategy.

The basic idea of the ticket lock algorithm implements a system whereby a ticket the demarcates a threads place in the queue of other threads that want the lock. How does this work?

<img src="resources/4_shared_memory/ticket_lock.png">

The lock has a struct with two fields, `next_ticket` and `now_serving`. A thread that wants to acquire the lock issues a fetch-and-inc instruction on the lock's `next_ticket` field. It then loops on pausing and then checking the lock's `now_serving` field. When a thread releases the lock, the thread increments the `now_serving` field.

One downside of this algorithm is that the `now_serving` value in the local cache is being incremented which causes cache-coherent mechanisms to flood the system bus.

Ideally, we would want a system where one thread could be signaled instead of all of them. 

### Queueing Locks

<img src="resources/4_shared_memory/array_queue_lock.png">

Let's talk about an array-based queueing lock. Associated with each lock, `l`, is an array of flags. The size of this array is equal to the number of processors in the shared memory processor. The **flags array serves as a circular queue for enqueueing the requestors for a particular lock**. This means that **each lock has a flags array**.

Each element in the flags array can be in one of two states:
1. **has-lock** - the thread has the lock and can do what it wants to do with it!
2. **must-wait** - the thread does not have the lock and must wait.

There can be exactly one processor with the **has-lock** state. One thing to notice is that the slots in the array are not statically-associated to any particular processor. There is a spot designated for each processor, but it is not an assigned spot. So how does it work?

Well, there is an integer associated with the flags array called `queuelast`. This references the index in the array that is available for queueing in the array. This index is incremented each time a processor requests a lock.

Because the array size is N (the number of processors), no processor will be denied a place in the queue. 

When a lock request is made, a fetch-and-inc instruction is made on the `queuelast` variable. This ensures that a spot  is set aside. Then the processor waits for its turn, this means it checks the flags array for the `has-lock` state.

A variant on the array-based queueing lock is the **linked-list based queueing lock**. This strategy **avoids the space-complexity of the array**-based queueing lock.

<img src="resources/4_shared_memory/linked_list_queue.png">

The size of the queue is going to be as big as the dynamic sharing of the lock. The head of the queue is a dummy node that is associated with every lock. There are two fields for every queue node for a requestor:

1. **guarded** - a boolean that says if the requestor has the lock
2. **next** - points to the successor in the queue. 

To add yourself to the linked-list you need to increment the last-requestor field on the dummy node of the linked list. This ensures that you are referenced by the previous last requested node. Now the requestor can spin on the guarded boolean variable.

Let's now talk about the lock algorithm. The lock takes the name of the lock, and the queue node that wants to be enqueued. The first thing to do is join the queue. 
1. **Join the queue** - The following must be done atomically. Set the last pointer of the head node to the new node. Update the last element in the list to point to the new last node. 
2. **Await the lock** - spin on the guarded value.

To facilitate the atomic queue-joining actions, we propose that an instruction called **fetch-and-store** is added to the instruction set.

The unlock function removes the current node from the linked list and signals to the successor that it is the current node.

A special case occurs when there is no successor to the current node. When that occurs, we have to set the head node of the linked list to point to NIL to indicate that no successor exists.

Let's talk about one of the classic race conditions of parallel programs. Imagine that a new request is forming for the lock queue, but the current node that is unlocking hasn't fully released itself from the lock queue. The requestor will be trying to build itself into the queue based on the current node's position, and the current node will be trying to extricate itself! This is a classic race condition that happens in all sorts of places in distributed/parallel systems. 

<img src="resources/4_shared_memory/ll_lock.png">

The solution to this problem is to add some more logic into the unlock function. Before the linked list sets the value of NIL on the head node, double check that there are no requests forming. This means **we need an atomic instruction for setting the value to NIL if the head node is pointing to the currently escaping node and not a currently forming request**. The solution is a **conditional store instruction called a compare-and-swap**. This instruction will only store if a condition is satisfied. 

Compare-and-swap will return true if the head pointer is pointing to the node trying to unlock itself. In this case, the compare-and-swap instruction will set the pointer to NIL. On the other hand, if the comparison fails, it won't do the swap, it will return false. If this is the case, then the node that is trying to leave needs to spin until the joining node finishes the lock call. At this point, the next pointer of the leaving node will be not NIL, and it can proceed with the unlock call by signaling to the successor that it now has the lock.