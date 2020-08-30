# Operating Systems Structure

## Table of Contents

* [Introduction](#introduction)
* [Goals of OS structure](#goals-of-os-structure)
    * [Monolithic Architecture]()
    * [DOS vs Monolithic]()

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

