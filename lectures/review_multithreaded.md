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
