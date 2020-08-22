# Review - Multithreaded Programming

## Table of Contents
* [Introduction](#introduction)

## Introduction

<img src="resources/multithreaded_resources/parallel.png"

Some tasks can be split up into independent pieces. This allows us to divide work amongst CPUs and save time! This is the beauty of parallel processing. 

## Asynchronous Computation

Even if you know that your application will only run on a single-core, single CPU, it can still be useful to have your program express parallelism. 

<img src="resources/multithreaded_resources/async.png"

The above model demonstrates the benefits of using a multithreaded programming paradigm, even in a single CPU system. This is because **we always gave the CPU something to do while we were waiting for asynchronous requests** to complete. This ensures that the page is loaded faster in a website. **Threads** make this possible.

## Process-Thread Relationship

In a multi-threaded environment, **each thread has its own stack, but it shares everything else in a process address space**. 

So from an execution perspective, it acts like a process unto itself. The difference comes in that part of the memory is shared. This makes it easier for the threads to communicate, but also to step on eachother's toes. 

<img src="resources/multithreaded_resources/process_thread.png"

It is also possible to do parallel programming without multiple threads through shared memory and message passing.

## Posix Threads (pthread)

```c
void* f1(void* arg) {
    int whos_better;
    whos_better = 1;
    while(1) {
        printf("Thread 1: thread %d is better.\n", whos_better);
    }
    return NULL;
}

void* f2(void* arg) {
    int whos_better;
    whos_better = 2;
    while(1) {
        printf("Thread 2: thread %d is better.\n", whos_better);
    }
    return NULL;
}


int main(int argc, char **argv) {
    pthread_t th1, th2;

    pthread_create(&th1, NULL, f1, NULL);
    pthread_create(&th2, NULL, f2, NULL);

    // main will wait until th1 and th2 are finished
    pthread_join(th1, NULL);
    pthread_join(th2, NULL);

    pthread_exit(NULL);
}

```

What will happen with the print statements? They are undeterministic, the scheduling will be handled by the threading library and the OS.

What happens if we save the `whos_better` variable as a global variable?

The threads will eventually agree on who is better because the threads share the global variable. Whichever thread runs second will become the better thread. Adding truth to the age-old adage that "first is the worst, second is the best". 

## Joinable and Detached threads

How does the memory for a thread get cleaned up after it finishes? Where does it's return value go? How can I make sure a thread finishes before the main thread finishes executing.

There are two cases for termination of "our" thread:
1. Any thread makes an exit call or main reaches the end of its code.
2. Our thread returns or calls `pthread_exit()`.

<img src="resources/multithreaded_resources/pthread_exit1.png">

<img src="resources/multithreaded_resources/pthread_exit2.png">

The distinction between joinable and detached threads is important for the second case where our thread returns or calls `pthread_exit()`. 

A thread can either be created in a detached state or detached with a procedure call, `pthread_detach()`. 

**When a detached thread reaches the end of its execution its memory is cleaned up and its return value disappears**. Moreover, we need to make sure that main doesn't finish first because our thread might not be done with its work!

If we want to keep the detached thread going, we have to exit main with `pthread_exit()`.

Alternatively, joinable threads don't get immediately obliterated after finishing execution. They stick around until another thread joins them. The joining thread makes a call to `pthread_join()`, specifying the thread it wants to join and an address to save the value of the threads work. `pthread_join()` blocks until the other thread finishes.

Let's take a look at joinable threads:

```c
void* thread_proc(void* arg) {
    printf("Hello from the created thread\n");

    return NULL;
}

int main(int argc, char **argv) {
    pthread_t thread;
    void* retval;

    pthread_create(&thread, NULL, thread_proc, NULL);

    printf("Hello from the main thread.\n");

    pthread_join(thread, &retval);
    return 0;
}
```

## Thread Patterns

