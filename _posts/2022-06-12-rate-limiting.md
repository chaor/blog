---
layout: post
title:  "[系统设计] 限速 Rate Limiting"
date: 2022-06-05 22:10:23-07:00
categories: systemdesign
---
Client变多或某个client的requests变多，为什么不能通过autoscale来解决？因为autoscale也需要时间，等server加完，说不定service就已经挂了。
为什么通过Load Balance的max connection来限制？因为Load Balance并不理解每种操作花费的时间，当我们只想限速某一种操作时，只能在service这边做。

Rate Limiting(限速)又名throttling(节流)可以针对用户，IP地址或某个region或server全局(server每秒总共能服务多少request)。取决于我们要对什么进行限速，就用什么来做client id。Service可以返回HTTP code 429表示Too Many Requests，也可以返回503 service unavailable。限速是可以分层的，比如对网络请求限制每秒1次，每分钟10次。限速可以防止拒绝服务攻击(Denial-of-Service, DOS attack)，但不容易放DDoS(Distributed Denial-of-Service)。推荐使用token来当client ID，而且系统要让client尽可能容易地产生token。用IP地址当client id的缺点是多个用户可能通过NAT共用一个IP地址。Note that users can be humans or services.

对于一个很快的API，经验规律是rate limiting花费的时间不应超过整个request time的10%。

最简单的Rate Limiting就是只有一个server且只有一层(比如每5秒服务一次)。在server端维护一个dict，记录每个用户的最近一次访问时间，检查当前用户的访问时间和上次时间是否超过5秒即可。在分布式系统中，如果没有固定某个server服务某个用户，那同一个用户的请求可能会被发到不同的server。此时就无法依靠单个server内存来判断了，可以用Redis来做Rate Limiting。Server收到请求时，先问Redis是否限速。

对于一个大型分布式系统，High Availability和Fault Tolerance对于Rate Limiting来说并不需要。如果Rate Limiter挂了或无法快速给出是否限速的决定，默认是不throttle。
如果service非常简单，那可以hardcode限速的rule到Rate Limiter(即Decision Maker)。如果有多条rule，比如给每个client都建立一个rule，那需要把rules存到Rules Database，比如“Client A每秒可以发500个requests”。然后建立一个Rules Service，对于Service Host，里面要有Throttle Rules Retriever来周期性从Rules Service里拿rule，还可以有个Throttle Rules Cache来给Rate Limiter用。

最简单的限速算法是Token Bucket Algorithm。Bucket有最大容量，里面放了一些token代表服务能力，token以恒定速率产生，每过来一个request就消耗一个或多个token(取决于服务这个请求有多耗时)，没有token时就拒绝服务。如果Client长时间没有再次发送请求，可以把bucket从内存里删掉，下次这个client有请求了再创建bucket。
不同host之间需要互相talk才能知道其他host的token消耗情况，可以通过redis，也可以互相直接talk，也可以选leader然后都talk to leader。

Redis里最简单的限速算法是Simple Fixed Window Counter，即在整数分钟或小时来进行限速
- 定义每个time interval能服务的request number，比如每分钟10次。
- 每个request用string表示，比如: user:ip-address:start-timestamp, user:127.0.0.1:1573767000
- 让Redis来处理expiration，常用的redis命令有`SET mykey 10 EX 40`(设置mykey的值为10, 40秒后expire删掉这个key), INCR mykey(把mykey的值加1), TTL mykey(查询mykey还剩多少秒)
- 算法是拿到request后，检查user key是否存在，不存在就创建这个key，expire设为当前time window还剩下的秒数，比如00:01:05收到两个request，就SET user:127.0.0.1:1655683265 2 EX 55。然后根据后续reqeust来INCR这个key，比如收到了额外3个request，INCRBY user:127.0.0.1:1655683265 3。检查value是否超过limit(即10次)，超过了就拒绝。
- 这个算法的问题是只考虑了整数分钟。如果在第一分钟的最后几秒和第二分钟的开头几秒分别处理了10个request，其实就在一分钟time window里处理了20个requests了。这样就造成了bursty requests。

更好的Redis限速算法是Sliding Window，可以用Redis的Sorted Set数据结构。优点是非常精确，缺点是内存消耗大。
- Sorted Set里面的member也是不重复的，每个member有score，根据score来排序。增加/删除/更新member的时间复杂度都是log(N)。可以根据score或rank(前几名)来查询range。
ZADD user1 score1 member1(往user1这个sorted set里加入score=score1的member1)。
- 每个用户有自己的sorted set，把timestamp当作score和member加进来。即来了新的request，ZADD user1 1510000000 1510000000。
- 用ZREMRANGEBYSCORE user1 min max来删掉score在min和max(inclusive)之间的member/timestamp，也就是删掉expired timestamp。
- 用ZCARD user1来拿member个数(即cardinality)，也就是当前time window里的请求个数，如果超过limit就deny。

Redis限速算法也可以做Token Bucket，可以用Redis的Hash数据结构，即key value。
- 对每个用户，存last request timestamp和available token count, user1 -> {ts:1600000000, tokens:10}
- 对于新请求，用HGETALL user1拿到这个hash的所有key value，用HSET来根据last timestamp来refill token，
更新当前timestamp，减少token count(HMSET)
- 没有token时deny request。
- 缺点是Redis operation不是atomic的，在分布式环境里有race condition，因为HGETALL和HSET之间有一段时间。可以用乐观锁。注意分布式系统中的这种GET-then-SET的操作。
