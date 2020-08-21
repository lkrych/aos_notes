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