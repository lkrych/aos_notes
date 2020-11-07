# Giant Scale Services

![](https://media.giphy.com/media/QBtzAnMFO5i9O/giphy.gif)

## Table of Contents

* [Introduction]

## Introduction

In this lecture we are going to talk about systems issues in large data centers, how to program big data applications, how to store and disseminate content on the web. 

## Systems issues in giant scale services

<img src="resources/9_giant_scale/giant_service_model.png">

Let's talk about the architecture of giant scale services. The architecture of a site will probably look like the image above. 

A **load manager** is the first point of contact with the outside world. It **balances the traffic** between the horizontally scaled systems behind it. Any of the servers may be able to handle any of the client requests. 

Client requests are independent of one another, they can be **handled in parallel**.

Another role of the load manager is to hide partial failures. There are so many nodes behind this manager. **It's not a question of if something will fail, but when**. So the load manager needs to ensure that the incoming client requests are shielded from these inevitable failures. 

**Computation clusters are the workhorses of giant scale services.** Any modern data centers employ thousands and thousands of these nodes connected by high-performance networks. Each node in a cluster might be an SMP. 

The advantages of managing identical nodes in a data center in a horizontally scaled manner is that it is much easier to manage, and keep costs down.

### Load Management at the Network Level

<img src="resources/9_giant_scale/round_robin_dns.png">

The load manager may be operating at the **network level**. In this case, the **load manager is a round robin DNS server**.

 When a client request comes in, they all have the same domain name to come to this particular service. The DNS server will direct the request to any of the nodes behind it by **returning a different IP (corresponding to a server) for the same domain name**. 

 The pro to this strategy is that there is good balancing because the scheduler can pick the least-loaded server. 

 The con is that it cannot hide downed server nodes from the external world. 

 ### Load Management in the Transport or Application level

 <img src="resources/9_giant_scale/transport_level_load.png">

 The load manager could be **layer 4 switches**, or even higher-level switches. This **gives the system the opportunity to dynamically isolate downed server nodes**. 

 It is also possible with this architecture to have service-specific front-end nodes. 

 ### DQ Principle

<img src="resources/9_giant_scale/dq_principle.png">

A service provides an offered load, in reality it can only provide a portion of that capacity because of random failures. This is known as the yield Q.

Each query that comes into the server might require processing all of the data in the scope of that server. This corresponds to the entire data corpus, however, in reality it can only process a portion of the data because of random failures. This value is known as the harvest of D. 

The **product of these two values**, the harvest D, and the yield of Q, **represents a system limit**. Said differently, we can **increase the number of client's served, if we reduce the amount of data that needs to be referenced** in a query.

As a system administrator, we have some choices to play with. The important thing to note is that in giant scale services, **the performance is limited by the network, not I/O capacity**.

A traditional measure that has been used in servers is IOPs (I/O operations per second), these are particularly meaningful in database applications. However, this isn't that meaningful in giant scale services which are bound by the network. DQ is a better metric.

Another metric is **uptime**, the ratio between the difference between mean time to failure and mean time to repair, and mean time to failure.

### Replication vs Partitioning

<img src="resources/9_giant_scale/replication_partition.png">

A system administrator has a choice to either replicate the data behind a service or to partition it.

In a **replicated system**, **every data store has the full corpus of data needed to serve a particular request**. Failures are inevitable in a giant scale service. What happens when we have replicated failure in a replicated system?

* The harvest is unchanged, but the capacity/yield goes down. A client request can be redirected to one of the live servers.

In a **partitioned system, every data store contains a portion of the data**. If there is a failure, then some of the data will be unavailable.

* The harvest will come down, but the capacity/yield will stay the same. 

One important thing to note is that **DQ is independent of whether we are replicating or partitioning the data**. This is because these services are network-bound. The only case where this is not the case is when a service requires a high amount of writes. In this case, replication may require a higher DQ.

Partitioned data should definitely be replicated.

### Graceful Degradation

The DQ principle is often helpful for managing graceful degradation of a service. DQ defines the system capacity. If a server is saturated, meaning we've reached the limit. Then we have the choice.

We can either keep the harvest the same, meaning every client request will have complete fidelity, but the amount of clients served will go down.

The other option is to keep the volume of the client steady, but to decrease the data harvest. The fidelity of the results will be less than 100%. We are keeping more of the community happy.

### Online Evolution and Growth

<img src="resources/9_giant_scale/server_swaparoos.png">

Services are continually evolving. This means the **servers at the datacenters need to be constantly updated**. Nodes need to be replaced or updated.

There are various strategies for doing this.

One is the **fast reboot** which means taking down all the services at once, update them and then turn them back on. This has to be done at offpeak times to not be incredibly disruptive. 

Another strategy is **rolling upgrade**. Bring down 1 server at a time. This means there is no loss in service, however, there will be disparities between individual services and there will be a loss in total yield.

A third alternative is the **big flip**. Bring down half the nodes at once and swap 'em. The service is always available but at 50% capacity.

No matter what strategy is taken, the DQ loss will be the same. 

## MapReduce

How do we program services on top of datacenters? How do we exploit hardware parallelism in these clusters?

The MapReduce programming paradigm helps solve some of these problems. Specifically, they help solve **embarrassingly parallel computations**, where no coordination or synchronization is needed amongst the parallel threads that comprise the applications.

MapReduce input is a set of records identified by **key-value pairs**. The big data application developer will supply the runtime environment with **two functions, map and reduce**. These are the **only two functions that the developer needs** to supply.

Both **map and reduce take user-defined key-value pairs as inputs and produce user-defined key-value pairs as outputs**.

Let's look at an example. Imagine we have a corpus of documents, and we want to get a count of how many times specific words are found in those documents.

The input key space will be the document filename, the value will be the contents of the file.

<img src="resources/9_giant_scale/map_reduce.png">

In this example, the **user-defined map function is looking for each of the unique words that we are interested in**. We will take as an input a specific file, and emit a value that corresponds to the number of times that word exists in the input file. This process is embarrassingly parallel, we can spawn as many map functions as we need. 

The output of the mappers are the input to the reducers. The **reducers will aggregate all of the emitted input** from the mappers. 

### Why MapReduce?

Several processing steps in giant scale services are expressible in the MapReduce framework. What are the common properties of these examples?

* embarrassingly parallel
* common in giant scale services
* all on big data sets

All the heavylifting that needs to be done for this data crunching are taken care of by the programming framework.