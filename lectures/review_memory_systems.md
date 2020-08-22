# Review - Memory Systems

## Table of Contents
* [Introduction]()
* []()

## Introduction

The most important resources on a computer are **space and time**, space and memory for storage and time on the CPU for processing. 

## Naive Memory Model

The CPU is where instructions are executed, the instructions may be read or written from main memory or disk.

<img src="resources/memory_systems_resources/naive_mem.png">

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

<img src="resources/memory_systems_resources/cache_motivation.png">

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

<img src="resources/memory_systems_resources/cache_heirarchy.png">

## Locality and Cache Blocks

In order for a cache to be effective, the hit rate needs to be high. That is, we need to anticipate what data the process will need next.

Ideally we would be able to plan ahead, instead we use the **heuristic of locality**. If an application needs memory from one memory address, it will likely need memory from the surrounding blocks. 

Experts like to distinguish between **two types of locality**: **temporal** and **spatial**. 

1. **temporal locality** - refers to the tendency of programs to refer to the same memory close in time. 
2. **spatial locality** - refers to the tendency of programs to access memory that is close together in terms of address.

Typical cache policies exploit both kinds of locality. It exploits temporal locality by putting your data in the cache after you use it, thinking you might use it again soon. It exploits spatial locality, by not just putting the address you needed into the cache, but the whole block that the address is part of in the cache. 

<img src="resources/memory_systems_resources/spatial_locality.png">

**Blocks** are a division of memory in relatively small amounts. Often 64 bytes.

## Direct Mapping

Given an address, how can I find out if that data is in the cache and how do I retrieve it? We will start with a direct-mapped cache and a little notation.

Let's suppose we have 2^m addresses, 2^n block size, and we have 2^k cache entries.

The lowest n-bits in an address are the **offset**. They **specify where within a given cache block the memory is**. The rest of the bits tell us the block number. 

How do we know where to look for this block within the cache? We need some kind of hashing function. The next k bits of the address are called the **index**. They tell us where to look for our data within the cache. The higher order bits are called the **tag**, and they specify within an index, which block is being used.

<img src="resources/memory_systems_resources/direct_mapping.png">

### Check your Understanding

<img src="resources/memory_systems_resources/memory_quiz.png">

In the example above, the offset is 1 bit, and the index and tag are both 2 bits.

<img src="resources/memory_systems_resources/memory_quiz.png">

Let's do the index first.

The cache can be broken into 8 blocks. 512/64. This means that only 2^3 blocks are needed for representing the index. `3`.

Next, for the tag we can work backwards. There are 22 total bits, minus 3 which are being used for the index, minus six (2^6), which is used for the cache offset to describe all of the addresses within the cache block. The answer is thus `13`.

## Associative Mapping

It's possible to get unlucky with direct map caching in that **blocks that are important to your application are mapped to the same index in the cache and are constantly evicting each other**. This is kind of silly given that a bunch of other cache indexes are probably vacant. 

The **fundamental problem is that the address is only associated with one location in the cache**. You can mitigate these effects by **associating an address with more than one block in the cache**.

<img src="resources/memory_systems_resources/associative_mapping.png">

The downside of this strategy is that we now have to check the tags of multiple cache lines. The upside is that we won't run into the problem of cache lines constantly evicting each other. 

### Checking for understanding

<img src="resources/memory_systems_resources/cache_quiz.png">

It's a miss. The index is 1. It is the 4th bit. The tag is `101`, and it doesn't match the tag in the cache ~ `010`. 


## Fully Associative Mapping

If we take associativity to the extreme, we let any cache block be put anywhere in the cache. All cache blocks are treated alike. With full associativity we don't treat any part of the address as an index anymore. It's all tag. This means that the hardware has to check tags for every entry into the cache. This fact generally limits fully-associative caches to being pretty small.

## Write Policies

For write policies, we need to consider what happens when there is a cache hit, and what happens on a miss.

### Cache hit policies

<img src="resources/memory_systems_resources/write_through.png">

One strategy is the **write-through policy**. That is we will write to the cache, and also to main memory to copy the data to keep the cache and memory consistent.

<img src="resources/memory_systems_resources/write_back.png">

Another strategy is **write-back** where only the cache is written to. As long as no other processors need access to the data that is being written, it is find just to write to the cache (this is where the same processor will look first) and then write to main memory later when it is more convenient.

### Cache miss policies

<img src="resources/memory_systems_resources/write_allocate.png">

<img src="resources/memory_systems_resources/no_write_allocate.png">