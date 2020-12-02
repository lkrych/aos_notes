# Memory Systems

## Table of Contents

* [Global Memory Systems](#global-memory-systems)
    * [Introduction](#introduction)
    * [Big Picture](#big-picture)
    * [Terminology](#terminology)
    * [Page Faults](#handling-page-faults)
    * [Page Fault Algorithm](#page-fault-algorithm)
    * [Implementation in Unix](#implementation-in-unix)
        * [Data Structures](#data-structures)
* [Distributed Shared Memory](#distributed-shared-memory)

## Global Memory Systems

### Introduction

In this module we will discuss the design and implementation of select distributed system services.

*Wisdom from Kishore - The byproducts of a thought experiment may have more lasting impact than the original vision behind the thought experiment.* 

The common theme we are going to discuss is **how we can leverage memory in a distributed system**. 

### Big Picture

Typically, the virtual address space of a process running on a processor is much larger than the physical memory that is allocated for a particular process. Thus, **the role of the virtual memory manager is to give the illusion to the process that all of its virtual address space is contained in physical memory**, when in fact only a portion of it (**the working set**) is actually in physical memory. The VMM supports this illusion by paging in-and-out of disk, the pages that the process needs for computation.

When a node is connected to other nodes in a LAN, it is likely that the memory pressure that a particular node feels compared to another node is different. One interesting question is to to think about how to take advantage of idle resources. 

**Is it possible that accessing remote physical memory on other nodes through network calls is faster than accessing a disk on the same machine?** At some point yes, networking technology is approaching 10GB/s throughput. 

<img src="resources/7_memory_systems/gms.png">

The **global memory system (gms)** uses cluster memory for paging across the network. This effectively **adds a new layer to the memory hierarchy**. 

GMS trades network communication for disk i/o. However, this **only applies to disk reads**. It does not get in the way of writing to the disk. The only pages that are paged out across the cluster are clean copies of pages. This means that if a node goes down, it's totally fine because the only thing that was a lost was a copy of what already exists on the original disk. 

When the VMM needs to replace the working set, instead of paging out the current memory to disk, it will place the pages out on the network (as long as they aren't dirty).

### Terminology

With GSM, "**cache**" **refers to physical memory** not the processor cache.

The word "community", is used to describe handling page faults at a particular node.

We are going to use peer memories as a supplement for the disk. **The physical memory for a node is split into two components**:
1. **Local memory** - working set of the currently executing processes.
2. **Global memory** - similar to a disk, out of my current physical memory, this is the part that is used as space for holding swapped-out pages. 

This split is dynamic, it is made in response to memory pressure. 

There are two states to memory pages in GMS, **private or shared**. A private page is local only to one node. A shared page is local amongst many nodes. If a page is in the global cache, it is still private. In the global cache, it is basically just a private copy, but stored somewhere else.

The whole idea of GMS is to **serve as a paging facility**. Coherence for shared pages is outside the purview of GMS, it considers this an application problem. Why? You can think about a uniprocessor example where many processes are sharing pages. It isn't really the VMM's responsibility to make sure that these pages are coherent, thus, in GMS, it is not it's responsibility.

Managing age information of pages in a distributed system of pages is one of the key technical contributions of the G<S> system.

### Handling Page Faults

Let's look at an example of a page fault in a GMS system with two hosts, P and Q. 

Let's say we are running a process on host P, and it page faults on same page X. When that happens, we have to find out if that page is in the global cache of the cluster.

<img src="resources/7_memory_systems/gms_page_fault1.png">

To handle the page fault, 
1. node P sends a search request for page X across the GMS. 
2. The GMS locates the page on host Q.
3. Host P makes a request to host Q for page X. 
4. Host Q sends page X to P across the network. 
5. Host P adds X to its current working set. This means that it needs to remove something from its local global memory (community service) and page it out to GMS. 

<img src="resources/7_memory_systems/gms_page_fault2.png">

The second case is similar to the common case except that the memory pressure is exacerbated on host P. There is so much memory pressure that all the physical memory that is available on P is taken up by the working set. There is no community service happening on host P. To handle the page fault,

1. The steps are roughly the same as above except the candidate/victim page for paging out comes from the local cache. 

The third page fault case happens when page X doesn't exist in the community service/global cache. 

<img src="resources/7_memory_systems/gms_page_fault3.png">

1. When a page fault on node P happens, it searches across the GMS.
2. It isn't found, so it makes a request for the page on its disk.
3. When the page comes in, it needs to page out into the GSM.
4. It uses LRU or some other scheme to page out a page in its working set. It's paged out to the node with the globally oldest page. 
5. If the node with the oldest page has memory pressure, it has to make some room for the new page. Because all paged out pages in GMS are clean, it can just discard the oldest page without worrying about writing it back to disk. 
6. However, if the oldest page has to be in local part of the node, then it is conceivable that the page is dirty and needs to be paged out to disk. 

So far the cases we've considered have assumed that the page we are faulting on is private to a process. Let's consider the case where the page is actively shared. In this scenario, two hosts, P and Q are both actively using a page to do some work. 

<img src="resources/7_memory_systems/gms_page_fault4.png">

When host P, needs page X
1. It makes a request for page X to the GMS.
2. The page is found in the local memory of host Q. 
3. Because it is in the current working set of host Q, we don't want to yank it out of memory. We want to **make a copy**.
4. Again, if there is pressure, the host P needs to page out a least-recently used page. 
5. The page is paged-out to the node with the globally oldest page. 

### Page Fault Algorithm

<img src="resources/7_memory_systems/page_fault_algorithm.png">

The behavior of the GMS algorithm is as follows.

If there is an idle node in the LAN, then the node's working set is going to decrease as it accommodates more and more of its peers pages that are swapped out to fit in the global cache. Eventually, a completely idle node because a memory server for peers on the network.

The **most important parameter in the working GMS system is age management**. We need to identify the globally oldest page whenever we want to do replacements.

One of the goals of distributed systems, or honestly, probably systems in general, is to **make sure that any management of the system doesn't bring down the productivity of any particular node**. This means that management work is distributed and is not centralized in one node. 

Because the **GMS system is dynamic, each node needs to understand how to communicate its needs with all the other nodes**.

<img src="resources/7_memory_systems/page_fault_algorithm2.png">

In GMS, management is broken up along two axes, space and time. Two parameters are used to describe an epoch (the granularity of management work done by a node), T, the maximum duration of an epoch. Or space-bound, M, maximum number of replacements.

This sounds complex but it is pretty simple. If a certain amount of time (T), passes, it's time to pass the management work of GMS to another node. If the node has done a certain amount of management work (M), then it is time to pass the management work to another node. 

At the start of each epoch, every node is going to send some information to the initiator (the node doing the work). That information will be the age of each page in its system (both local and global).

The initiator node will then do two things:
1. It will calculate the minimum age of all the M pages that are going to be replaced. Any page that is less than the minAge will be safe from eviction.
2. It calculates the weight parameter for each node, which is the fraction of pages to be evicted in the current epoch that exist on the node. 

Remember, the algorithm doesn't know the future. So, these are **expected calculations**.

<img src="resources/7_memory_systems/page_fault_algorithm3.png">

The **initiator node is new at each epoch**. The initiator is chosen to be the node with the highest weight. Why? Because **the weight parameter indicates that it is the least active node**. It is the node currently doing the least amount of work, that's why it is storing all the other nodes pages.

This election process happens because the initiator node sends back all the weights to each of the nodes in the system.

So **what is the action at a node when there is a page fault?**

If upon a page fault, a node needs to evict a page Y, if the age of the page is more than the minimum page, than we know it is going to be replaced, we can just drop it. 

However, if the age of the page that needs to be replaced is less than this minimum, we send it to a peer to store. **Which peer is it sent to? It depends, usually it is the node with a higher weight.** 

This algorithm approximates a global LRU algorithm.

### Implementation In Unix

The authors of GSM used a DEC OS as the base system.

<img src="resources/7_memory_systems/unix_imp.png">

There are two key components to the GSM implementation.
1. **The virtual memory system** - responsible for mapping process virtual address space to physical memory and worrying about page faults, etc.
2. **The unified buffer cache** - the cache used by the filesystem, exists in physical memory for faster access. It serves as the abstraction for the disk-resident files. Reads and writes to files actually go to this entity. 

<img src="resources/7_memory_systems/unix_imp2.png">

The GMS implementation modifies both of these systems to handle the needs of GMS.

When either system wanted a missing page, they would go to the disk. In GMS, they go to GMS first, :). They call `getPage`,  and GMS will query the network for that missing page. 

Notice that **writes to disk are unchanged**. We do not want to affect the reliability of the system. Only when there is a page fault do we consult GMS.

Remember that the most **important aspect of GMS is retrieving accurate age information** about the files in the system. This isn't too bad for pages housed in the unified buffer cache. We can just consult file handles to check for timestamps. The VMM manages anonymous pages, and these are much harder to determine any information about. To collect information about anonymous pages, a daemon is used to dump information from the TLB. The GMS uses this data dump to derive age information for the anonymous pages. 

#### Data structures

One of the things that GMS has to do is convert a local virtual address into a global identifier (universal ID). The universal ID is derived by concatenating a bunch of information (ip-addr, disk partition, i-node, offset) about the page.

There are three key data structures used in GSM work.

<img src="resources/7_memory_systems/unix_imp3.png">

1. PFD (Page Frame Directory) - Like a page table. **A page table typically associates a virtual address with a physical page frame**. The PFD **has a mapping between a UID an the page frame that backs that UID**. The page can be in one of four different states: local private, local shared, global private, on disk. 
2. GCD (Global Cache Directory) - A distributed hash table that given a UID, will tell you which node has the PFD. Because it is partitioned, every node may not be able to determine where the PFD lives. How does it now which GCD to consult? See below :)
3. POD (Page Ownership Directory) - Maps GCD for UID. This is replicated on all the nodes. 

The **path for page fault handling** is as follows:

1. Convert VA to UID
2. Go to POD and ask who has the PFD for the UID? (this is local to the node requesting it because it is replicated)
3. Receive the node to query the GCD from
4. Query the GCD at the node from above for the node that holds the PFD. 
5. Receive the node to query the PFD from
6. Query the node for the PFD and grab the page you need.

<img src="resources/7_memory_systems/unix_imp4.png">

One thing to notice about the schema above is that the most common case for a node is that a page is non-shared. The page exists locally on the node. This means that the GCD entry for the page probably exists on the node that is experiencing the fault.

There is another case we need to talk about. What happens when you consult the node that is supposed to have the page frame that you care about and it has dropped that page?

<img src="resources/7_memory_systems/unix_imp5.png">

In the case of eviction, the evicting node has to tell the owning node that it evicted the page. It is something that happens asynchronously, and it's possible that the data structures (GCD) haven't been updated yet to reflect this. 

Updating nodes about their evicted pages happens in a batch. This is because updating nodes individually for every eviction would be costly. **This is managed by the individual nodes paging daemon**. It is integrated with the GMS, and when the free list falls below a certain threshold, then it will start evicting pages and sending them to nodes that are storing fewer pages. This decision is made using the weight list.

Another case that is uncommon is that the POD information that the node has is stale. This happens when the POD information is being recomputed for the LAN. 

## Distributed Shared Memory

Let's look at another way to exploit remote memory. Let's look at a software implementation of distributed shared memory. We will create an OS abstraction that creates an interface for software that makes the system look like it is using shared memory.

### Implicit Parallelization

Let's start the section with a motivating question. Suppose we have a sequential program. How can we exploit the cluster of machines hooked up through the LAN?

<img src="resources/7_memory_systems/automatic_parallel2.png">

One possibility is to do **automatic parallelization**. In this scheme, instead of writing an explicitly parallel program, we write a sequential program and let someone else identify the opportunity for parallelism in the program and map it to the underlying cluster. 

Typically, this involves a compiler that knows how to sniff out parallel operations. High performance fortran is an example of this. This works pretty well for a certain class of programs - data parallel programs. Characteristics of these types of programs are that data accesses are fairly static and it is determinable at compile time. There is thus limited potential for exploiting parallelism if we result to implicitly parallel programming.

## Message Passing

<img src="resources/7_memory_systems/message_passing.png">

A different model is to write the program as a truly parallel program. 

There are two styles of doing this, we will talk about one of them, **message passing**. In this style, the **runtime system provides a message passing library which has primitives for an application thread to do sends and receives** to other nodes within the cluster. 

One way to think about this model is that it truly reflects the architecture of the system. It respects that each machine has its own CPU and memory. The second project of this class used MPI, to create barriers using message passing.

The only **downside to this style is that it is difficult to program** using this style. It would be easier if we had the notion of shared memory. 

## Distributed Shared Memory

<img src="resources/7_memory_systems/dsm.png">

The alternative to message passing is the abstraction of distributed shared memory. **We want to give the illusion to the application programmer writing an explicitly parallel program that all of the memory that is in the entire cluster is shared**. The DSM library gives the illusion to all of the threads running in each of the nodes.

Therefore programs have an easier transition path from running on an SMP to running on a cluster. 