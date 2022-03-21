---
layout: post
title:  "[系统设计] 负载均衡 Load Balancer"
date: 2022-03-13 19:04:19-07:00
categories: systemdesign
---
Server只能处理有限的请求，我们可以垂直scale server，即用一个更强大的server，也可以水平scale server，就是增加server数量。`Load Balancer`(负载均衡器)是位于client和server之间的一个server，是一种反向代理，用于让server收到的请求更加均衡，避免单个server过载，提高系统latency。Load balancer可分为软件和硬件两种，软件的更灵活。

负载均衡选择server的方式可以是Round-robin或weighted round-robin，比如1,1,1,2,3,...N,1,1,1,2,3,...N，这里server 1更强大，可以处理更多请求。可以是随机选择server。可以是基于性能选择server，比如选响应时间最快的server或负载最小的server。还可以基于client的IP地址，求hash然后选server，好处是这个server可以缓存有关这个client的数据。还可以基于网址(path-based)来选择server，比如写代码的请求去server1，支付的请求去server2。

如果每个server收到的负载不均衡，可能是哈希函数又问题，或负载本来就不均衡。收到更多traffic的server叫热点(`hot spot`)。

负载均衡器可以接server，也可以接另一个负载均衡器。系统中可能在多处有LB。系统的同一个位置也可以放多个LB，这些LB互相交流，以避免某一个load balancer过载。

L4 LB工作在OSI模型第4层(transport传输层)，其实也包含了第3层(network网络层)，根据IP地址和端口来做负载均衡。Client跟LB建立TCP连接，LB复用这个TCP连接跟server相连，把source IP换成load balancer自己的IP。因为不知道请求数据内容，所以并不智能，也无法用于streaming/keep-alive connection。

L7 LB工作在第7层(application应用层)，client和load balancer之间有一个TCP连接，LB和server之间有个新的TCP连接。因为知道请求数据内容，所以可以做authentication，smart routing, TLS termination。TLS termination是指TLS加密数据被解密(decrypted or offloaded)。client和LB之间是HTTPS连接，LB和server之间是HTTP连接，好处是减少server负担。为了安全，有些LB可以在它和server之间提供一个self-signed TLS。注意HTTPS加密所有内容，包括HTTP headers, request/response data。

来看一个weighted round-robin的例子。首先起两个server，分别监听3000和3005端口:

{% highlight python %}
from flask import Flask, request

app = Flask(__name__)

@app.route('/')
def hello_world():
    print('request.headers =\n', request.headers)
    return 'Hello!\n'
{% endhighlight %}

保存为`hello.py`，用 `FLASK_APP=hello flask run -h localhost -p 3000` 和 `FLASK_APP=hello flask run -h localhost -p 3005`启动。

{% highlight nginx %}
events { }

http {
    upstream flask-backend {
        server localhost:3000 weight=3;
        server localhost:3005;
    }

    server  {
        listen 8000;

        location / {
            proxy_pass http://flask-backend;
        }
    }
}
{% endhighlight %}

保存为`nginx.conf`，然后用`nginx -c nginx.conf`启动nginx反向代理server。现在用`curl localhost:8000`去访问，会看到3:1的访问量，有四分之三的request去了3000端口。