# Virtualization

## Table of Contents

## Introduction

The drive for extensibility in OS services led to innovation in the internal structure of operating systems. In this lesson we will see how the concept of virtualization has taken the vision of extensibility to a whole new level: namely the simultaneous existence of multiple OS's on the same hardware platform.

## Platform Virtualization

The hope for **virtualization** is that we can give applications a platform to run on without having to dedicate that platform to JUST running that application. This **makes running services cheaper**! We as OS designers are interested in how this black box works. 

<img src="resources/3_virtualization/blackbox.png">

Sharing resources can make the operating cost of an organization cheaper. However this model introduces a lot of confounding problems for the companies running the virtualization technology. How do we keep these applications secure and prevent them from stepping on each other's toes?

<img src="resources/3_virtualization/bursty.png">

The fundamental intuition that makes sharing hardware resources across a diverse set of users is the fact that **resource usage is very bursty**. The VM typically has a memory that is much bigger than the needs of the individual applications. The cost of maintaining the platform is shared by all the tenants of the system. Virtualization is the application of extensibiilty at the granularity of an entire OS instead of with individual services like in SPIN and the exokernel. 

### Hypervisors

In a shared environment, **how are operating systems protected from one another**, and who decides **who gets the resource and at what time**. What we need is **an operating system of operating systems**, something called a **virtual machine manager**, or more simply (cooly), a **hypervisor**.

<img src="resources/3_virtualization/hypervisors.png">

There are two types of hypervisors, a native hypervisor/bare metal (type-1). It is running atop the shared hardware. 

The second type is called a hosted hypervisor (type-2), these run on top of a host operating system.

For the purpose of these lessons, we will focus on **bare metal hypervisors**. These hypervisors **interfere minimally with the normal operation of the guest OS's**. For this reason, bare metal hypervisors **offer the best performance for the guest OS** on a shared resource.

### Full Virtualization

<img src="resources/3_virtualization/full_virtualization.png">

In **full virtualization**, the **OS is left untouched**, so that you can run the unchanged binary on top of the hypervisor. 

We have to be clever to get this to work. The **guest OS's** running on top of the hypervisor are run as **user-level processes**. If the OS code isn't changed though, it doesn't know that it doesn't have the privilege for issuing privileged instructions.

When the OS executes privileged instructions (needs to be in kernel mode), those instructions create a trap that goes into the hypervisor and the hypervisor emulates those instructions. This is known as the **trap-and-emulate strategy**. 

There are some thorny issues with this strategy, in some architectures, some privileged instructions may fail silently. In order to get around this problem, the hypervisor will resort to **binary translation**. Meaning, the OS knows which things can fail silently in the architecture, look for those gotchas and through binary editing, ensure that those instructions are dealt with carefully so that if those instructions fail, the hypervisor can catch them and take the appropriate action. This is employed in the VMware system.

### Paravirtualization

Another approach to virtualization is to **modify the source code of the guest OS**. If we can do that, we can avoid problematic instructions and also include optimizations (like letting the guest OS see real hardware resources).

<img src="resources/3_virtualization/paravirtualization.png">

The percentage of the guest OS code that needs to be modified is miniscule, less than 2% of the original guest OS.


### Big Picture

In either virtualization circumstance we need to virtualize the hardware resources and make them safely available to the guest OS's. The hardware resources are the memory hierarchy, the CPU and the devices in the hardware platform.

## Memory Virtualization

Caches are physically tagged so you don't have to do too much about handling them in a virtualized space. The really thorny issue comes in handling virtual memory -- The virtual address to the physical address mapping.

Recall that in any modern OS, each process is in its own protection domain and usually its own hardware address space. The OS maintains a **page table** for each process, and it is the **OS data structure** that holds the **mapping between the virtual pages and the physical pages**.

<img src="resources/3_virtualization/memory_system.png">

The physical memory is contiguous starting from 0 to the max of the hardware. The virtual address space is not contiguous, it is scattered all over the physical memory. This is the advantage of page-based memory systems.

### Memory Virtualization in a Virtualized Environment

In the virtualized setup, the hypervisor sits between the guest OS and the hardware. Thus the picture gets more complicated. Inside each guest OS are user-level processes, and each process is in its own protection domain. That means within each guest OS there is a distinct PT for each process within the guest OS. 

Does the hypervisor know about these page tables? No, it doesn't! So how does it manage?

<img src="resources/3_virtualization/machine_memory.png">


The guest OS's think of physical memory as contiguous, but unfortunately the real physical memory (machine memory), is in control of the hypervisor, not the guest OS. Thus **the physical memory being managed by each guest OS is an illusion in a virtualized environment**. 

So what is going on within a given guest OS?

The process address space within an OS is an illusion, the physical memory that the OS thinks it has is actually being managed by the hypervisor. 

Remember, the page table is a data structure that is managed for each process within an OS that handles the mapping between a Virtual Page Number and the Physical Page Number. This is how a typical OS behaves.

<img src="resources/3_virtualization/shadow_pagetable.png">

In a virtualized setting we have **another level of indirection whereby each physical page number in an OS has to be mapped to machine memory**,machine page numbers.

The mapping between the **physical page number and the machine page number** is maintained in another page table called the **shadow page table**. 

Thus in a virtualized setting there is a two-step translation process to go from VPN to MPN. In the case of a fully-virtualized hypervisor, the shadow page table is kept in the hypervisor. In a paravirtualized setting, the guest OS knows it is not running on the hypervisor. It can thus house the mapping. 

### Shadow Page Table

<img src="resources/3_virtualization/pagetable.png">

In many architectures, the CPU uses the page table for address translation. What that means is that presented with a virtual address, the CPU first checks the TLB to see if there is a match for the VPN. If there is a match, there is a hit and it can translate to a physical address. If there is a miss the CPU goes to the page table in main memory and retrieves the entry for the virtual address. 

The hardware PT is really the shadow page table in the virtualized setting if the architecture is going to use the page table for address translation.

