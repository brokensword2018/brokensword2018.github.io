---
title: 拥抱UDP之QUIC
date: 2021-08-25 22:53:09
tags:
    - WEBRTC
categories: 音视频
---

### 前言

 QUIC(quick udp internet connection)；是由google提出，基于UDP在应用层实现数据可靠有序传输的协议。相比于目前普遍使用的http2+tls+tcp主要拥有如下优势：

<!-- more -->

- 更少的连接建立时间
- 改进的拥塞控制
- 无队头阻塞现象
- 连接迁移

本文就这几点进行简单的介绍。

### 更少的连接建立时间

{% asset_img https握手及QUIC通信.bmp https握手及QUIC通信 %}

https握手及QUIC握手如上图所示。

- https握手需要经过tcp三次握手及TLS4次握手，共需要3.5个RTT，对于短连接来说消耗巨大。
- QUIC采用DH秘钥协商算法，除了第一次建立连接请求公钥需要一个RTT，之后建立连接都是0RTT。

### 改进的拥塞控制

QUIC默认采用的拥塞控制算法与TCP一样。QUIC改进之处在于其拥塞控制算法是可插拔的。

- 支持多种拥塞控制算法（BBR, PCC），可自由选择。
- 在同一个程序中甚至可以对不同的连接采用不同的拥塞控制算法，针对用户的网络情况采取合适的算法。

### 无队头阻塞现象 

{% asset_img http2队头阻塞.bmp http2队头阻塞 %}

多路复用时http2队头阻塞现象如上所示。形成原因：由于内核层的TCP需要保证数据的有序性，当某些TCP包丢失时，应用层无法读取到丢失包之后的数据，即使数据包已经到达了内核层。

{% asset_img QUIC多路复用.bmp QUIC多路复用 %}

QUIC多路复用如上图 所示，当UDP包发生丢失时，应用层可以读取后面的UDP数据包不会发生阻塞现象

### 连接迁移

- TCP的连接基于[源ip, 源port, 目标ip, 目标port]。当四元组里某个元素发生变化（例如客户端从4G网络切换成wift）连接就需要重新建立。
- QUIC使用一个随机生成的64位数字作为连接ID，就算ip和端口发生变化，只要ID保持不变，连接就一直维持着。

### 参考资料

[让互联网更快的“快”---QUIC协议原理分析](https://zhuanlan.zhihu.com/p/32630510)
[QUIC 0-RTT实现简析及一种分布式的0-RTT实现方案](https://cloud.tencent.com/developer/article/1594468)