# Operating Systems Structure

## Table of Contents

* [Introduction](#introduction)
* [Goals of OS structure](#goals-of-os-structure)
    * [Monolithic Architecture](#monolithic-architecture)
    * [DOS vs Monolithic](#comparing-DOS-to-a-monolithic-structure)
    * [Microkernel Architecture](#microkernel-based-architecture)
* [SPIN](#the-spin-approach)
    * [Approaches to Extensibility](#approaches-to-extensibility)
    * [SPIN's Approach to Extensibility](#spins-approach-to-extensibility)
    * [Logical Protection Domains](#logical-protection-domains)
    * [Domain Protection](#spin-mechanisms-for-domain-protection)
    * [Events](#spin-mechanisms-for-events)
    * [Default Services](#default-core-services-in-spin)
* [Exokernel](#the-exokernel-approach)
    * [Implementing Secure Bindings](#implementing-secure-bindings-with-exokernel)
    * [Default Services](#default-core-services-in-the-exokernel)
    * [Secure Binding](#secure-binding)
    * [Software Caching](#software-caching)
    * [CPU Scheduling](#cpu-scheduling)
    * [Revocation](#revocation-of-resources)
    * [Data Structures](#exokernel-data-structures)
* [L3 Microkernel](#l3-microkernel-approach)
    * [Performance loss in a microkernel?](#potentials-for-performance-loss-in-a-microkernel)
    * [Introduction to L3](#l3-microkernel)
    * [In defense of a microkernel](#strikes-against-a-microkernel)
    * [Memory Effects](#memory-effects)
    * [L3 Summary](#l3-summary)


## Introduction

An OS needs to protect the integrity of the hardware resources it manages while providing the services to applications.

Some of the components of an OS need to run in a privileged mode of the processor architecture that allows them access to hardware, but most the entire OS have this privilege?

**OS structure** is a term used to reference **how the OS software is organized** with respect to the applications it serves and the underlying hardware it manages.

## Goals of OS structure

* **Protection** - protecting the user from the system and vice-versa. 
* **Performance** - Time taken to perform services on behalf of the applications. A good OS is quick and gets out of the way.
* **Flexibility** - Extensibility, a service is not one-size-fits-all, it is adaptable.
* **Scalability** - The performance increases as the hardware resources increases. 
* **Agility** - The OS is able to adapt to changing application needs or resource availability.
* **Responsiveness** - The OS can react to external events. Especially important for interactive applications.

All of these goals are not simultaneously achievable. There are always trade-offs with design decisions. 

### Monolithic Structure

<img src="resources/2_os_structure_resources/monolithic.png">

The most common organization of an OS is the **monolithic** approach, where the entire OS runs as **a single program in kernel mode**. The OS is written as a collection of procedures, linked together into **a single large executable binary program**. 

When this technique is used, each procedure is free to call any other procedure. This is efficient, but having thousands of procedures that can call each other without restriction can also make the system difficult to understand (also **a crash in any of these procedures will take down the entire OS**!).

**System calls** are exported by the OS  to applications to provide a well-defined **interface for interacting with hardware**. 

### Comparing DOS to a Monolithic structure

Microsoft's first OS, DOS (Disk Operating System), let applications have direct access to system services as procedure calls which gave the OS better performance. The problem with this design is that there wasn't much protection against errant applications corrupting the OS!

<img src="resources/2_os_structure_resources/dos.png">


Technically, this meant that the OS and the application code lived in the same address space. 

The loss of protection to the OS in a DOS-like architecture is unacceptable in today's world. On the otherhand, monolithic architecture ensures this safety.

One of the ways that the monolithic design aids in increasing performance is to consolidate all of its code into a single binary. This means that once a system call has transitioned application code from user space to kernel space, all of the code needed for the OS will be inside the same address space.

What is lost in the monolithic structure is customization and flexibility. Everything is built into a single binary, there is no mix-and-match.

### Microkernel-based Architecture

The need for customization and the opportunity for customization is what spurred OS designers to create an OS structure that would allow for the customization of services offered by the OS.

<img src="resources/2_os_structure_resources/microkernel.png">

The basic idea behind the microkernel design is to a**chieve high reliability by splitting the OS into small well-defined modules**, **only one** which, the **microkernel, runs in kernel mode**. The others run as relatively powerless user processes.

**Only mechanisms for accessing hardware (no policies) are situated in the microkernel**. These simple mechanisms include threads, address space and IPC.

The OS services such as VM management, CPU scheduling, and the filesystem are implemented as servers on top of the microkernel. These services execute with the same privilege as the applications themselves. Each service is in it's own address space. 

In a microkernel, in principle, there is no distinction between regular applications and system services that are executing as server processes on top of the microkernel.

This gives us strong protection between all of the different layers of the microkernel. We have also gained extensibility. It's easy to mix and match new OS services by building the on top of the microkernel.

One **downside of the microkernel approach is that there is a potential for performance loss** because we've introduced so many new boundaries (address spaces) between services. There is the potential for many more context switches in a microkernel architecture per system call than in a monolithic architecture.

<img src="resources/2_os_structure_resources/microkernel_vs_monolith.png">

Border crossings change the locality of the execution of the processor. Also, when are going across boundaries, there may be a need to copy from user space to system space. This can be slow as well.

<img src="resources/2_os_structure_resources/what_do_we_want_of_os.png">

## The SPIN Approach

Both the SPIN and the Exokernel approaches start with two basic premises:

1. The microkernel design compromises on performance due to frequent border crossings.
2. The monolithic design does not lend itself to extensibility because everything is built in one binary.

Let's revisit what we are trying to achieve in an OS structure.
* We want the OS to be thin, **there should only be mechanisms in the kernel, not policies** (like microkernel). 
* The structure **should allow fine-grained access to system resources without border-crossing** too much (like DOS).
* The OS should be flexible, resource management should be flexible (like microkernel) without sacrificing protection and performance (like monolithic). 

### Approaches to Extensibility

Historically, there has been interest in extensibility in OS's since 1981, when a team of researches from CMU introduced the Hydra OS. Hydra OS used kernel mechanisms for resource allocation. It provided access via **capabilities**, an entity that can be passed from one to another, it cannot be forged, and it can be verified. Capabilities are a heavy-weight mechanism. Resources managers were introduced as course-grained objects to reduce border crossing overheads.

While in principle, Hydra had a thin design and used resource managers to reduce border crossings, the design did not lend itself to extensibility.

One of the most well-known extensible OS's in the early days was the Mach operating system from CMU. It was microkernel based, providing very limited mechanisms in the kernel. It provided all the services above the kernel. It achieved its goal of extensibility and portability, but performance took a back seat. 

### SPINs approach to extensibility

SPIN's idea was to **co-locate the kernel and the extensions** (same hardware address space) so as **to avoid border crossing**. 

SPIN relied on the characteristics of a **strongly-typed programming language to give guarantees** about how protected each process was. This is called **compiler-enforced modularity**. 

SPIN uses **logical protection domains**, not hardware address spaces to protect the OS from services and the kernel.

The flexibility of SPIN comes from **dynamic call binding**, different functions can be attached to the OS interface for a service. This makes extensions as cheap as a procedure call.

### Logical Protection Domains

Modula-3 is a strongly-typed language with built-in safety and encapsulation methods. It does automatic management of memory. It supports objects, threads, exceptions, and generic interfaces. 

OS services are packaged up in Modula-3 objects which gives some guarantees of a protected interface. This allows for both protection and performance. 

Objects that implement the specific services can be fine-grained, or they can be collections of interfaces. All of these are accessible via capabilities. **Capabilities are implemented as pointers in Modula-3**, which means they are much lighter-weight. Modula-3 pointes are type-specific which makes them easier to guard.

### SPIN Mechanisms for Domain Protection

There are three mechanisms in SPIN to create protection domains. Write your code in Modula-3 and use these three mechanisms.

1. `create()` - initiates an object with contents and exports names that are contained as entry point methods to be visible outside the object. This creates logical protection domains.
2. `resolve()` - if one protection domain wants to use the name of another protection domain, it can accomplish this using the resolve primitive, which is **very similar to linking two separately compiled files together**(compiler process). **Once resolved, resource sharing is done at memory speeds**.
3. `combine()` - to reduce the profileration of small logical protection domains, you can combine domains and aggregate them using combine.

This is the secret sauce of SPIN, everything hinges on the strongly-typed nature of Modula-3.

The upshot of the logical protection domain, is the **ability to extend SPIN to include OS services and make them all part of the same hardware address space**. No border-crossings!

<img src="resources/2_os_structure_resources/spin.png">

This image demonstrates the flexibility of the extensions, here we see two separate extensions (individual OS's) that can share components. 

### SPIN mechanisms for events

An OS has to field external events. For example, external interrupts that come in as a process is executing, or the process itself may incur some exceptions that cause an interrupt in its own execution. All of these are events that an OS has to support.

<img src="resources/2_os_structure_resources/spin_event_handler.png">

SPIN supports these events using an **event-based communication model**. Services register event handlers with the SPIN event dispatcher. SPIN supports several types of mapping:
* 1:1 mapping between an event and handler
* 1:many mapping between an event and handlers
* many:1 mapping to the same handler. 

An example of this is a protocol stack. There might be several interfaces for network connections, say ethernet and ATM. These packets arrive on one of these ports and this arrival is considered an event.

Both of those packets might be IP packets, so the event is handled by the IP handler(many:1), which looks at the events and decides which client (1:many) to hand it off to: ICMP, UDP, or TCP.

### Default Core Services in SPIN

Physical memory is a precious resource. SPIN provides interfaces for extending physical memory management.

1. Physical address - allocation, deallocation, and reclamation.
2. Virtual address - allocation and deallocation.
3. Translation -  Create/destroy address spaces, add or remove mapping between virtual/physical pages.
4. Event Handlers - page fault, access fault.

**SPIN provides interface functions in header files** for adding your own custom policies to these core services of an OS. 


**SPIN also arbitrates the CPU** It only arbitrates at a macro-level the amount of time that is given to a particular extension. That is done through the SPIN global scheduler.

1. SPIN abstraction - Strand (thread abstraction defined by SPIN as a unit of scheduling). The semantics of the Strand is defined by the extension.
2. Event Handlers - block, unblock, checkpoint, resume
3. SPIN global scheduler - interacts with application threads package to allocate the time slice. 

Core services are trusted services since they provide access to hardware mechanisms. This means they might have to step outside the language-enforced protection model to control the hardware-resources. In other words, the applications that run on top of an extension have to trust that extension.

## The Exokernel Approach

The **exokernel** approach to extensibility is that the **kernel exposes hardware explicitly to the OS extensions** living above it. 

**Decouple authorization of the hardware with its actual use.**

<img src="resources/2_os_structure_resources/exokernel.png">

1. The library OS asks for a resource from the exokernel. 
2. Validates the request from the library and binds the request to the specific hardware resource.
3. This exposes the hardware through a secure binding
4. The exokernel then creates an encrypted key for the resource and gives it to the requesting service. 
5. Finally, the library uses the keys to access the hardware.

In this system, the creation of the keys is the computationally expensive procedure, but the use of the keys is much cheaper.

Let's look at an example of a resource: a TLB entry. Imagine you want to add a TLB entry to the hardware TLB. 

1. Virtual to physical mapping is done by the library that wants to add the entry.
2. Once that mapping has been done, it presents this mapping to the exokernel, along with the key that it has for the TLB.
3. The exokernel validates this request and then on behalf of the user process, adds the entry to the hardware.
4. The process that is going to use this page can now use this entry without exokernel intervention. (putting the entry in required intervention, but use of the entry doesn't require exokernel intervention)

### Implementing secure bindings with exokernel

There are three methods:
1. **Hardware Mechanisms** - specific hardware resources that can be requested by the the Library OS and can be bound to that Library OS by the exokernel and exported to the Library OS as an encrypted key. 
2. **Software Caching** - At the point of context switch, the exokernel will dump the hardware TLB into a software TLB associated with the Library OS. This preserves access that has been granted to services/processes within that OS if the exokernel needs to switch to other processes.
3. **Downloading code into Kernel** - Avoid border processing by importing specific code into the kernel address space. This is functionally-equivalent to the SPIN strategy, but is less safe because SPIN enforces compile-time checks.  

### Default Core Services in the Exokernel

<img src="resources/2_os_structure_resources/exokernel_pagefault.png">

A process running on behalf of an application will not have access to update a TLB, so a page fault will percolate up through the exokernel to fetch the page that needs to be mapped in the TLB.

### Secure Binding

The Library OS is given the opportunity to drop arbitrary code into the kernel. The rational for this ability is performance. This is dangerous! So how do we make it as safe as possible?

### Software Caching

<img src="resources/2_os_structure_resources/soft_tlb.png">

When we have a context switch, one of the **biggest sources of performance loss is that we lose locality**. Since the address spaces of different processes are necessarily different, we have to flush out the entire TLB. 

One strategy for reducing overhead is maintaining **in-memory data structures that operate as snapshots of the TLB for different processes**. 

When a context switch happens, the exokernel preloads the TLB with the software TLB that is associated with the new process.

### CPU scheduling

In order to facilitate the core service of CPU scheduling, the exokernel maintains a linear vector of "time slots". 

<img src="resources/2_os_structure_resources/exokernel_scheduling.png">

Time is divided into quantums/epochs. The time quantums represent the time allocated to the Library OS/process that is operating on top of the exokernel. Each process gets to mark its time quantum in the vector. 

Thus CPU scheduling is as simple as keeping track fo what time it is and looking up in the vector which process should be running.

If a process misbehaves and takes a little more time than it should, the scheduler remembers and adds a penalty in subsequent scheduled execution so that execution time remains fair between processes.

### Revocation of Resources

Notice that the exokernel doesn't support abstractions. It only has mechanisms for securely giving resources to the Library OS/process. 

Therefore the OS needs a way to revoke resources that have been allocated to a Library OS/process. To do this, the exokernel maintains a table of what resources are allocated to which OS/process.

The exokernel uses the `revoke()` mechanism to revoke access to specific resources in a Library OS. The exokernel gives the Library OS time to take corrective action when a `revoke()` call is made. 

### Exokernel Data Structures

<img src="resources/2_os_structure_resources/exokernel_problems.png">

In order to know how to deal with any kinds of interrupts in program execution, the exokernel must maintain data structures that link running processes with the proper Library OS/applications running a top of it.

The exokernel maintains a data structure called the **PE data structure**. It contains the **handler entrypoints to the Library OS for dealing with the different kinds of program discontinuities**, like page faults, exceptions, interrupts, etc. It additionally maintains software TLBs and other mappings that allow for fast execution.

<img src="resources/2_os_structure_resources/exokernel_ds.png">

## L3 Microkernel Approach

Both the SPIN and the exokernel approach were built with the assumption that the microkernel design was inherently poised for poor performance.

Why? They used a common microkernel approach, Mach that favored portability over performance. Let's look at a microkernel design that tries to keep performance in the forefront.

<img src="resources/2_os_structure_resources/microkernel_review.png">

A **microkernel** provides a **small number of simple abstractions** and **all the system services that you normally expect from an OS are implemented as processes above the microkernel**, they run at the same privilege-level as user-level applications. Only the microkernel runs at a different level of privilege. 

In order for services to communicate with each other, they need to utilize IPC.

### Potentials for Performance Loss in a Microkernel

The main source of performance loss is **border crossings** in the microkernel design. They have both an explicit cost (making system calls from application processes to the microkernel) and implicit costs (border crossings between services).

<img src="resources/2_os_structure_resources/protected_procedure.png">

**Protected procedure calls** are made between services. Because they go across address spaces, they are much less efficient than normal procedure calls. 

The implicit cost of border crossing is the fact that we are **losing locality** both in terms of **address translations in the TLB**, as well as the **contents of the cache** that the processor uses to access memory. 

### L3 Microkernel

The L3 microkernel was built to prove that a microkernel doesn't need to have bad performance. 

<img src="resources/2_os_structure_resources/l3_micro.png">

The key distinction of L3 is that **each OS service is in its own protection domain**. Not necessarily its own hardware address space. 

The point L3 establishes is that there are ways to construct a microkernel efficiently knowing the features of the hardware platform. 

### Strikes against a Microkernel

<img src="resources/2_os_structure_resources/microkernel_costs.png">

1. **Kernel-user switches** - border crossings are expensive, this happens every time a user level process makes a system call. 
2. **Address Spaces switches** - Each system service lives in its own address space, any system call that triggers action that needs to take place across OS services is going to incur address space context switch performance penalties.
3. **Thread switches and IPC** - Thread switches need to be mediated by the kernel. This will happen in service-to-service communication.
4. **Memory Effects** - an implicit cost, the loss of locality when doing many context switches.

L3 debunks each of these strikes. Let's look at how it does this. 

The first thing it does is debunking the border-crossing myth. The implementors of L3 found that they can accomplish border crossing in 123 process cycles, this includes TLB/cache misses. 

Let's now debunk the address switch costs of going across protection domains.

The appropriate question to ask here is **if we have to throw away/flush all entries in the TLB** when we want to move across protection domains. The answer is that it depends. 

If the TLB keeps track of address space, then we don't have to flush all of the TLB. This is called **address space tagged TLBs**. 

Virtual addresses are being generated on behalf of a particular process. When we make an entry into the TLB is not only the tag with a particular virtual address, but also the PID for the entry.

When a TLB entry is looked up, both the address and the PID are checked. The hardware disambiguates the entries per process using PID, this means the whole TLB doesn't have to be flushed in a context switch.   

<img src="resources/2_os_structure_resources/as_tagged_tlb.png">

Liedtke, the author of the L3 microkernel, had some suggestions for exploiting the hardware to avoid TLB flushing even if the TLB is not AS/PID tagged.

In particular, the architecture might support segment registers (which bound the legal virtual addresses of a running process). You can carve out individual protection domains in the linear address space. 

<img src="resources/2_os_structure_resources/segment_registers.png">

What if the protection domain is so large that it needs the entire hardware address space? If this is the situation, then the TLB needs to be flushed on a context switch. 

Switching between small protection domains can actually be pretty efficient. On the other hand if the switch is from a large protection domain to another, the switching cost is not that significant, BUT the loss of locality is significant (TLB misses and cache misses).


The third myth about microkernel performance is that thread **switching and IPC is expensive**. The explicit cost in this case is saving all the volatile state of a processor into the thread context block. L3 shows that this can be competitive to the SPIN and exokernel design.

### Memory Effects

The last **myth is around the assertion that microkernels will lose locality much more than a monolithic design**.

Because of demand paging design, the entire hardware address space is probably not all-in memory. What the criticism of the microkernel design above means, is that within a demand-paged OS design with a typical CPU hierarchy, **how warm are the caches when we context switch**?

<img src="resources/2_os_structure_resources/microkernel_memory_effects.png">

When we have small protection domains, it will probably be pretty good. This begs the question, why was border crossing so bad in Mach?

The reason is that it was focused on portability, the code was bloated which led to a **larger memory footprint**. This meant that the protection domains were larger and thus there were more cache misses.

### L3 Summary

The L3 paper's importance lies in its debunking of the belief that microkernel OS design is inefficient. The first principal advocated by L3 is that the OS should have minimal abstractions: address space, threads, IPC, UID. These are abstractions that are needed by any subsystem. 

If you want an efficient implementation of a microkernel, you need to be processor-specific. This means that microkernels are non-portable. 

The right set of kernel abstractions and processor-specific implementation allows microkernels to be efficient. Then **other services can be built in processor-independent ways** on top of the microkernel.

