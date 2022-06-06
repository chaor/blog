---
layout: post
title:  "[系统设计] 选主 Leader Election"
date: 2022-05-22 18:30:04-07:00
categories: systemdesign
---

处理业务的server为了保证高可用，通常会有几个replica。选主就是从这些server里选出一个来处理业务，比如支付，其他的server就成了follower。Follower把收到的请求转发给leader，因为显然不可能让每个server都对同样的支付请求处理一次。难点在于让所有server达成谁是leader的共识。这里需要用到consensus algorithm共识算法，即在分布式系统中就某一数据值达成一致。共识算法满足3个特性:
- Agreement 每个正确的进程必须就某一数据值value X达成一致，即不存在两个正确的进程有不同的决定
- Validity 达成一致的value X必须是其他正确的进程提议的，即达成一致的value X不能是某个进程凭空发明的
- Termination 在某个时间点所有正确的进程要决定达成一致的value X

常见的共识算法有paxos(1998年)和raft(2014年)。我们一般不会直接接触到这两种算法，更多情况下是使用工具比如zookeeper和etcd。Zookeeper使用的共识算法是2007年提出的ZAB(ZooKeeper Atomic BroadCast)。Etcd使用的共识算法是Raft(Replicated and Fault Tolerant)。
- Etcd is a strongly consistent and highly available key-value store that's often used to implement leader election in a system. Etcd cannot be stored in memory/they can only be persisted in disk storage, whereas redis can be cached in ram and can also be persisted in disk.
- ZooKeeper is a strongly consistent, highly available key-value store. It's often used to store important configuration or to perform leader election.

Etcd提供了对象的TTL和原子性的compare-and-swap操作，所以实现选主比较容易。没有被选中的节点可以watch `/election`或某个key，如果发现为空就选自己为leader。这个值有TTL，leader必须周期性重写这个值保持选中状态。
