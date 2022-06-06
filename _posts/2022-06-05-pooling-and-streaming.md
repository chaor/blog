---
layout: post
title:  "[系统设计] 轮询和流式传输 polling and streaming"
date: 2022-06-05 22:10:23-07:00
categories: systemdesign
---
如果server端的数据频繁更新，比如股票价格或聊天，client如何持续拿到呢?
- 轮询(polling)，client每过几秒请求一次server，逻辑简单。缺点是无法及时拿到server的最新信息。减少interval，比如每0.1秒查询一次，对server的压力就太大了。
- 流式传输(streaming)，client开一个long-lived connection(即socket)和server通信，只要client或server没有关闭连接且网络健康，则socket一直保持。Client并不发请求，只是listen，server有新数据就发给client。Streaming的方向是从server到client，这种server主动发数据给client的方式也叫pushing。适用于server数据频繁更新的情况。
