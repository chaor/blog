---
layout: post
title:  "[系统设计] 复制和分片 Replication and Sharding"
date: 2022-05-13 23:21:16-07:00
categories: systemdesign
---
复制(Replication)将同一份数据拷贝到多个节点，或者是一个standby数据库。主数据库不断往standby数据库同步数据，如果主数据库挂，备用数据库就上。复制的数据叫replica。为了保持主数据库和备用数据库同步，写速度会变慢。如果不关心latency，比如发帖存到了主数据库，备用数据库过一会显示帖子也行，就可以用async方式同步，比如几分钟同步一次。

分片(Sharding)将不同数据存放到不同节点。当访问量对系统throughput要求越来越高时，考虑水平扩展增加机器并分割数据，即分片，每个片叫shard或data partition。可以按照顾客名字的first letter分片，或者顾客的区域分片。注意分片时哈希函数要robust，要避免hot spot，即有些shard拿到太多traffic。实际情况中的数据库分片可能会连接一个反向代理，反向代理连到application server。

想增加系统读性能，复制/增加slave节点即可。想提升写性能，对数据分片。
