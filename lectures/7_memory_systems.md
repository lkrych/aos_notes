# Memory Systems

## Table of Contents

* [Global Memory Systems](#global-memory-systems)
    * [Introduction](#introduction)
    * [Big Picture](#big-picture)

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

