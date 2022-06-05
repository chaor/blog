---
layout: post
title:  "[系统设计] 分布式锁 Distributed Lock"
date: 2022-05-24 22:35:19-07:00
categories: systemdesign
---
锁的目的是同步并发操作。Concurrent transaction need to be synchronized.

分布式锁的目的是保证多个节点做同样工作时，只有一个节点能成功，或在某一时间只有一个节点能成功。使用分布式锁好处有:
- Efficiency 同样的活避免干两次。如果同样的活两个节点分别干一次，浪费计算资源，用户可能收到两份一样的邮件等等。
- Correctness 两个节点干同样的活，可能破坏数据或一致性。显然correctness这一点更重要。在[how to do distributed locking](https://martin.kleppmann.com/2016/02/08/ how-to-do-distributed-locking.html)一文中提到Redis的分布式锁Redlock不能保证正确性，因为对GC，网络延迟，存储系统的写延迟，时钟等做了太理想的假设。作者Martin Kleppmann提出应该用一种提供单调递增token的锁服务，存储系统见到旧token就拒绝服务。这种token叫做fencing token。同时作者认为Redlock比较heavy，也无法满足efficiency的要求。对于不需保证正确性的锁，作者推荐单节点redis node来做lock。对于需要保证正确性的锁，避免Redis，可以用Zookeeper，At the very least, use a database with reasonable transactional guarantees. 而且需要使用fencing token(单调递增token)。

3种锁解决方案:
- 数据库锁 Database Lock
- Shard Application，应避免这种做法
- 中心化的分布式锁管理器 Centralized Distributed Lock Manager。这种东西很难开发，因为需要用到Pasox/Raft/ZAB协议，还要处理Master Failover和Leader election。

数据库锁比如MySQL的行级别锁Shared Lock共享锁和Exclusive Lock排他锁
- Shared lock允许拿到锁的transaction读这一行。其他transaction无法给这一行上排他锁，其他transaction可以给这一行上共享锁。`SELECT ... LOCK IN SHARE MODE`
- Exclusive lock允许拿到锁的transaction更新或删除这一行。其他transaction既无法给这一行上共享锁，也无法上排他锁。
`SELECT ... from ... where ... FOR UPDATE`

上面的锁属于悲观锁(pessimistic lock)，即我写数据时很悲观，认为其他人也在写数据，所以要先加锁。缺点是效率低，容易发生死锁，适用于并发写操作多的场景。要求保持与数据库的连接。

与之相对的是乐观锁(optimistic lock)，即我写数据时很乐观，认为其他人没有在写数据，对数据记一个版本号(数据库表中单独的一列version)或timestamp，先操作，写数据时如果版本号没变就写数据并对版本号加1，如果版本号变了就放弃(dirty record)，适用于读操作多的场景，不要求与数据库一直保持连接，需要时从数据库连接池里拿一个。缺点是用户代码要多写一点，维护这个version或timestamp。DB Lock is fine, but the Optimistic Lock is great.

Sharding数据的方式是把对同一份数据的多个访问固定到了一个shard上，这样在单机上就可以用mutex等锁了，会用到一致性哈希。缺点是容易造成热点，操作多个数据不方便(因为多个数据未必在同一节点上，比如从account A往account B转账)。注意单机上的锁的实现和硬件，指令集(Instruction Set Architecture)，操作系统，编程语言都相关。为什么我们自己用一个flag锁不住？因为不具备atomicity(flag++仍然是读内存，CPU加1，写内存)和memory reordering(为了提速)。x86提供了LOCK指令前缀，保证了原子性和没有乱序执行。
```
LOCK ADD 原子加，对应Go语言的atomic.Add
LOCK CMPXCHG 原子Compare-and-swap(CAS)，只有看到expected value时，才把变量值更新为desired value，对应Go语言的atomic.CompareAndSwap
```
以下结构叫spinlock，在Linux kernel里广泛使用。
```
for {
    // Try to atomically CAS flag from 0 -> 1
    if atomic.CompareAndSwap(&flag, 0, 1) {
        ...
        // Atomically set flag back to 0
        atomic.Store(&flag, 0)
        return
    }

    // CAS failed, try again
}
```

Distributed Lock Manager(DLM)提供锁和租约(lease)机制，而且不能是只在内存的，需要persist，需要高可用。好处是service就可以没有状态了，状态被存在了DLM。DLM比如有zookeeper。每个cient跟locknode保持心跳，locknode每收到一个client的心跳，就把client对应锁的序号加1，没收到心跳就删除这个client。client可以列出这个lock的所有序号，如果发现自己的序号最小，就拿到锁。Client B如果发现Client A的序号比自己小，也可以watch client A，等到client A没了以后再试图获取锁。
