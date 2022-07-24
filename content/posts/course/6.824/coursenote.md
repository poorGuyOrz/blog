---
title: "Coursenote"
date: 2022-05-17T13:17:04+08:00
draft: false
---

## 随堂笔记

Why do people build distributed systems?
  to increase capacity via parallel processing
  to tolerate faults via replication
  to match distribution of physical devices e.g. sensors
  to achieve security via isolation


分布式的困难点：
  1. 大量的并发操作
  2. 具有容错性
  3. 难于实现

Lab 1: distributed big-data framework (like MapReduce)
Lab 2: fault tolerance library using replication (Raft)
Lab 3: a simple fault-tolerant database
Lab 4: scalable database performance via sharding

A big goal: hide the complexity of distribution from applications.

Topic: fault tolerance
  1000s of servers, big network -> always something broken
    We'd like to hide these failures from the application.  
    "High availability": service continues despite failures  
  Big idea: replicated servers.
    If one server crashes, can proceed using the other(s).
    Labs 2 and 3

Topic: consistency
  General-purpose infrastructure needs well-defined behavior.
    E.g. "Get(k) yields the value from the most recent Put(k,v)."
  Achieving good behavior is hard!
    "Replica" servers are hard to keep identical.

Topic: performance
  The goal: scalable throughput
    Nx servers -> Nx total throughput via parallel CPU, disk, net.
  Scaling gets harder as N grows:
    Load imbalance.
    Slowest-of-N latency.
    Some things don't speed up with N: initialization, interaction.
  Labs 1, 4

Topic: tradeoffs
  Fault-tolerance, consistency, and performance are enemies.
  Fault tolerance and consistency require communication
    e.g., send data to backup
    e.g., check if my data is up-to-date
    communication is often slow and non-scalable
  Many designs provide only weak consistency, to gain speed.
    e.g. Get() does *not* yield the latest Put()!
    Painful for application programmers but may be a good trade-off.
  We'll see many design points in the consistency/performance spectrum.

Topic: implementation
  RPC, threads, concurrency control, configuration.
  The labs...


## GFS


Why is distributed storage hard?  
  high performance -> shard data over many servers  
  many servers -> constant faults  
  fault tolerance -> replication  
  replication -> potential inconsistencies  
  better consistency -> low performance    

是一个闭环问题，高性能需要更多的机器，更多的机器导致更高的故障率，解决这个问题一般使用多副本解决，但是多副本又带来不一致的问题，而为了更高的一致性，又会导致性能下降

 
### consistency

在单机上，多个用户同时写一个数据，最终的结果可能是其中一个，但是后续所有的用户都会得到同一个数据，这是强一致性  
但是在分布式集群上，用户写数据，之后的用户在不同的机器上得到的数据需要看此时集群的一致性协议

* 分布式的核心问题是一致性以及其他限制之间的博弈


master记录文件块的元数据信息，例如文件所在的服务器，文件分片信息等  
server保存文件，文件使用3副本

What are the steps when client C wants to read a file?
  1. C sends filename and offset to coordinator (CO) (if not cached)
  2. CO finds chunk handle for that offset
  3. CO replies with list of chunkhandles + chunkservers
     only those with latest version
  4. C caches handle + chunkserver list
  5. C sends request to nearest chunkserver
     chunk handle, offset
  6. chunk server reads from chunk file on disk, returns to client