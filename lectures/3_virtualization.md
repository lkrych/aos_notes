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

## Hypervisors

In a shared environment, **how are operating systems protected from one another**, and who decides **who gets the resource and at what time**. What we need is **an operating system of operating systems**, something called a **virtual machine manager**, or more simply (cooly), a **hypervisor**.

<img src="resources/3_virtualization/hypervisors.png">

There are two types of hypervisors, a native hypervisor/bare metal (type-1). It is running atop the shared hardware. 

The second type is called a hosted hypervisor (type-2), these run on top of a host operating system.
