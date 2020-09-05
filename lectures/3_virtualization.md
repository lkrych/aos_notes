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