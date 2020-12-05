# System Recovery

## Table of Contents
* [Lightweight Recoverable Virtual Memory]

## Lightweight Recoverable Virtual Memory

### Introduction

System crashes can happen during power failures, hardware and software failures. As OS designers, we need to know how to build systems that can survive crashes. 

Many OS subsystems need persistence, for example, file systems have i-nodes that describe how files are persisted on disk. I-nodes are a persistent data structure. While the OS might manipulate that i-node, the OS needs to put it back on disk.

All of these subsystems need to have an in-memory cached copy of the persistent data that lives on the disk somewhere. If and when changes are made to these in-memory structures, those changes to need to be written back to the disk.

One possibility for achieving this is to make virtual memory persistent. The upshot of this is that the subsystems don't have to worry about flushing the persistent objects to the disk. Some other layer will handle this process. This also makes crash recovery really easy because all data at the time of the crash will be available. 

How do we make this abstraction efficient? 

The way the system works now is that the virtual address space of a process may have persistent data structures strewn all over the address space. If these structures are written to or manipulated and you want them to be reflected in storage, it requires that the system makes many I/O operations. Second, it could also mean that we may be writing to different portions of the disk. This can be very slow because there are latencies associated with seeking.

<img src="resources/8_system_recovery/log_based.png">

Instead of doing this, we would like to have a log segment, this is very similar to the [log structured system we saw in our discussion of xFS](https://github.com/lkrych/aos_notes/blob/master/lectures/7_memory_systems.md#log-structured-file-systems). 

The idea is that when we are makinng changes to the virtual memory and we know which portions are persistent, every time we write to a persistent in-memory data structure, then we write a log record that corresponds to the change we made to the data structure. The log segment is a data structure that supports this subsystem. It is a collection of these log records written contiguously. Then we commit the changes to the disk and store it contiguously on the disk. 

This design eliminates random writes/invidual I/O operations to the disk because we are recording them as changes. We also eliminate the disk write inefficiencies. 

### Server Design

<img src="resources/8_system_recovery/lrvm_server_design.png">

Let's discuss how we would design a server that requires persistent support using a persistent virtual memory.

Notice that the entire virtual address space doesn't need to be persistent. Because we are designing this system, we know which data structures need to be persistent. 

We allow an application to bundle these data structures together into an external data segment. This is a data segment that lives in the virtual address space but is directly mapped to persistent data on the disk.

The application establishes this mapping at startup but can easily unmap it during the runtime if it needs to. 

### RVM primitives

We do not want the in-memory data structures to immediately start writing to disk when they are modified. That would result in a lot of random writes. 

The better model is to **use a log-segment to aggregate the changes we are making to the virtual address space so that the log-segment can be committed to the disk** to record the changes that have been made. 

RVM is a runtime library that can be used by application developers.

RVM provides three initialization primitives:
* `initialize(options)` - identifies the log segment to be used by the server process for recording changes to the persistent data structures of this process. Every process can declare its own log segment for use in managing its persistence.
* `map(region, options)` - Allows the application to say what is the region of the virtual address space that they want to map to an external data segment. There is a 1-to-1 correspondence between an address range in the VA space and the external data segments. 
* `unmap(region)` - Unmap does the reverse, it decouples the address range from the external data segment.

In the body of the server code, the following actions are defined. 
* `begin_xact(tid, restore_mode)` - alerts the RVM runtime that the application is about to make changes to persistent data structures after this call.
* `set_range(tid, addr, size)` - the very first thing that an app developer would do between a begin and end. This establishes the portion of the address range that is going to be modified in the critical section.
* `end_xact(tid, commit_mode)` - alerts the RVM runtime that the application is not going to make any more changes to persistent data structures after this call. **After this call, all of the changes need to be flushed to the disk.**
* `abort_xact(tid)` - signals to the RVM runtime that all the changes that the application made bound between the begin and end should be aborted and not be committed to the disk.


As we said earlier, RVM needs to be efficient or no one would use it! The runtime does not actually write the persistent data directly to the data segments. Instead it writes the changes that it made to the block of address specified by the `set_range` call as a redo-log in a log segment that was named in the `initialize` call.

The log segment is an in-memory data structure specified by the RVM runtime. Once a transaction commits, the log segments that contains the changes that have been made to the in-memory version of the persistent data structures will be committed to the disk. 

There are two parts to managing the log: **flushing and truncation**. 
**Flush** - At the point of commitment the log has to be forced to the disk because you want to persist it.  
**Truncate** - You need to apply the changes from the log to the external data segments and  delete the log-segment. 

They are provided as primitives to the application, but **both of these are done automatically by the runtime**.

One optimization features provided by the runtime is an option to the `end_xact` call that informs the runtime that the developer does not want the system to flush the changes to the disk yet, but that she will take care of the flushing.

The main thing to take away is that the RVM API is simple :).

### How the Server uses primitives

<img src="resources/8_system_recovery/lrvm_primitive.png">

The first thing the runtime does inside a transaction is to create a copy of the chunk of the address space specified in the `set_range()` call. This copy is known as the **undo record**. It is set aside because it is possible that the developer will call `abort_xact()`, and it needs to make sure that none of the changes made to the persistent data structures are actually made.

LRVM creates the undo record only if it is needed by the transaction semantic, in the `begin_xact()` call there is a mode specifier that the user can specify to the runtime whether that transaction could ever abort. 

In any event if the transaction eventually commits, at that point it will throw away the undo record. 

Finally, if the transaction commits by calling `end_xact()`, then all the changes made to the persistent data structures need to be written to the log segment that records the redo logs for this thread. At this point the runtime creates a **redo log of the changes made to the region** specified by `set_range()`. The redo log is a data structure managed by the runtime in memory that corresponds to the changes made to the external data segments. It should not be confused with the external data segments themselves. 

The redo log is not available as a log entry in the log segment created at the beginning of the transaction. The runtime then needs to flush the redo log to disk synchronously, this means the `end_xact()` call waits for the redo log to be written to disk, then it returns. 

<img src="resources/8_system_recovery/lrvm_primitive2.png">

Once the transaction is committed, the redo log has been written to disk. Then the undo record is no longer needed, and can be thrown away. 