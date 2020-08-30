# Operating Systems Structure

## Table of Contents

* [Introduction](#introduction)
* [Goals of OS structure](#goals-of-os-structure)

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

Microsoft's first OS, DOS (Disk Operating System), let applications have direct access to system services as procedure calls which gave the OS better performance. The problem with this design is that there wasn't much protection against errant applications corrupting the OS!

<img src="resources/2_os_structure_resources/dos.png">


Technically, this meant that the OS and the application code lived in the same address space. 


