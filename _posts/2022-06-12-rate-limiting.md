---
layout: post
title:  "[系统设计] 限速 Rate Limiting"
date: 2022-06-05 22:10:23-07:00
categories: systemdesign
---
Client变多或某个client的requests变多，为什么不能通过autoscale来解决？因为autoscale也需要时间，等server加完，说不定service就已经挂了。
为什么通过Load Balance的max connection来限制？因为Load Balance并不理解每种操作花费的时间，当我们只想限速某一种操作时，只能在service这边做。

Rate Limiting(限速)又名throttling(节流)可以针对用户，IP地址或某个region或server全局(server每秒总共能服务多少request)。取决于我们要对什么进行限速，就用什么来做client id。Service可以返回HTTP code 429表示Too Many Requests，也可以返回503 service unavailable。限速是可以分层的，比如对网络请求限制每秒1次，每分钟10次。限速可以防止拒绝服务攻击(Denial-of-Service, DOS attack)，但不容易放DDoS(Distributed Denial-of-Service)。

最简单的Rate Limiting就是只有一个server且只有一层(比如每5秒服务一次)。在server端维护一个dict，记录每个用户的最近一次访问时间，检查当前用户的访问时间和上次时间是否超过5秒即可。在分布式系统中，如果没有固定某个server服务某个用户，那同一个用户的请求可能会被发到不同的server。此时就无法依靠单个server内存来判断了，可以用Redis来做Rate Limiting。Server收到请求时，先问Redis是否限速。

对于一个大型分布式系统，High Availability和Fault Tolerance对于Rate Limiting来说并不需要。如果Rate Limiter挂了或无法快速给出是否限速的决定，默认是不throttle。
如果service非常简单，那可以hardcode限速的rule到Rate Limiter(即Decision Maker)。如果有多条rule，比如给每个client都建立一个rule，那需要把rules存到Rules Database，比如“Client A每秒可以发500个requests”。然后建立一个Rules Service，对于Service Host，里面要有Throttle Rules Retriever来周期性从Rules Service里拿rule，还可以有个Throttle Rules Cache来给Rate Limiter用。

最简单的限速算法是Token Bucket Algorithm。Bucket有最大容量，里面放了一些token代表服务能力，token以恒定速率产生，每过来一个request就消耗一个或多个token(取决于服务这个请求有多耗时)，没有token时就拒绝服务。如果Client长时间没有再次发送请求，可以把bucket从内存里删掉，下次这个client有请求了再创建bucket。
不同host之间需要互相talk才能知道其他host的token消耗情况，可以通过redis，也可以互相直接talk，也可以选leader然后都talk to leader。
