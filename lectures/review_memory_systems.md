# Review - Memory Systems

## Table of Contents
* [Introduction]()
* []()

## Introduction

The most important resources on a computer are **space and time**, space and memory for storage and time on the CPU for processing. 

## Naive Memory Model

The CPU is where instructions are executed, the instructions may be read or written from main memory or disk.

<img src="resources/review_memory_systems/naive_mem.png">

1. **Main memory** is fast, you can access any element in the same amount of time (random access), lastly it is temporary. 
2. **Hard Disk** is slow, it uses sequential access because the disk has to be spun to the right place, lastly the memory stored on a disk is permanent. 

In this lecture we are going to refine this picture by looking at **cache systems** which **allow the CPU to access the data in main memory faster**.  Second, we will talk about the **virtual memory system**, the abstraction provided by the OS that **makes handling memory easy**. 

## Cache Motivation

Let's gaze at the humble for-loop:
```c
int sum = 0;
int N = 5;
int a[5];
for(int i = 0; i < N; i++) {
    sum += a[i];
}
```

From an application programmer's perspective, the addition call would appear to happen in one step. Even in assembly, it would only take a few instructions. In terms of time on the clock however, **not all of these instructions are the same**.

<img src="resources/review_memory_systems/cache_motivation.png">

`sum` being a local variable, is likely stored on a register in the CPU, therefore there is almost no cost to retrieving it. `a[i]` however resides in main memory and retrieving it can take 100 CPU cycles. 

Trips to main memory are expensive. The solution to this problem is to place **smaller faster memory closer to the CPU**, this is called a **cache**. The latency to pull from the cache is closer to 10 CPU cycles. 

Keeping cache misses infrequent is a goal of computer system designers. 

### Check for understanding

Suppose retrieving data costs 4 cycles on a cache hit, and 100 cycles on a cache miss. 

How many cycles does a process with a 99% hit rate spend on average?

`((1 * 100) + (99 * 4))/100 -> 4.96 cycles`

How about one with a 90% hit rate?

`((10 * 100) + (90 * 4))/100 -> 13.6 cycles`

## Memory Hierarchy

One of the most important considerations in cache design is the **tradeoff between speed and capacity**. We would like the retrieval to be fast and for the cache hit rate to be high. These goals however are somewhat contradictory.

To deal with this we trade-off size vs speed multiple times by creating a cache hierarchy. 

<img src="resources/review_memory_systems/cache_heirarchy.png">

## Locality and Cache Blocks

In order for a cache to be effective, the hit rate needs to be high. That is, we need to anticipate what data the process will need next.

Ideally we would be able to plan ahead, instead we use the **heuristic of locality**. If an application needs memory from one memory address, it will likely need memory from the surrounding blocks. 

Experts like to distinguish between **two types of locality**: **temporal** and **spatial**. 

1. **temporal locality** - refers to the tendency of programs to refer to the same memory close in time. 
2. **spatial locality** - refers to the tendency of programs to access memory that is close together in terms of address.

Typical cache policies exploit both kinds of locality. It exploits temporal locality by putting your data in the cache after you use it, thinking you might use it again soon. It exploits spatial locality, by not just putting the address you needed into the cache, but the whole block that the address is part of in the cache. 

<img src="resources/review_memory_systems/spatial_locality.png">

**Blocks** are a division of memory in relatively small amounts. Often 64 bytes.

## Direct Mapping

Given an address, how can I find out if that data is in the cache and how do I retrieve it? We will start with a direct-mapped cache and a little notation.

Let's suppose we have 2^m addresses, 2^n block size, and we have 2^k cache entries.

The lowest n-bits in an address are the **offset**. They **specify where within a given cache block the memory is**. The rest of the bits tell us the block number. 

How do we know where to look for this block within the cache? We need some kind of hashing function. The next k bits of the address are called the **index**. They tell us where to look for our data within the cache. The higher order bits are called the **tag**, and they specify within an index, which block is being used.

<img src="resources/review_memory_systems/direct_mapping.png">

<img src="resources/review_memory_systems/memory_quiz.png">

## Associative Mapping