---
layout: post
title:  "[系统设计] 点对点网络 Peer-to-peer networks"
date: 2022-05-24 22:35:19-07:00
categories: systemdesign
---
如何把一个大文件从一台机器传到多台机器上？都从一台机器上拷太慢了，即使把文件分到几台机器上拷也是慢，sharding也慢。点对点网络是个不错的解决方案。

在点对点网络中，每台机器叫peer，我们把大文件打散成很多带序号的小文件。一开始source node传出这些小文件，之后很多机器可以同时互传这些小文件，这些小文件是以指数级扩散的。有两种方式可以让peer互相发现:
- 用一个相对中心化的server，叫tracker
- 用gossip protocol，peers talk to each other，会用到Distributed Hash Table，提供key-value lookup service。Peers共同来维护DHT。

https://github.com/uber/kraken是个P2P网络的例子。At its peak production load, Kraken distributes 20K 100MB-1G blobs in under 30 sec.

点对点协议已经不怎么流行了，除了文件共享，没什么新应用，围绕点对点协议建立可持续的商业模式也很困难，因为需要中心化系统来管理付款。点对点协议从定义上就无法实现中心化追踪，也无法保证质量，因为用户不断上线下线，恶意参与者都可以自由加入网络。在大公司的数据中心用点对点协议共享大文件还是有意义的。
