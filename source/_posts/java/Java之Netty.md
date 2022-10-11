---
title: Java之Netty.md
date: 2021-12-25 10:18:43
tags: 
- netty
- java
categories: java
---
## 介绍
Netty是一套支持NIO的客户端-服务器框架。
高性能、高并发。
支持异步通信。
他是使用Java编写的。
在Java领域中，他是IO界的老大，特别是网络IO。
很多项目都用它，比如Dubbo，RocketMQ，Cassandra等等。

在这片文章中，我们会学习到：
1. 常见的IO模型
2. Netty项目
3. Netty实战
4. Netty在我司生产中的应用

那我们开始吧！

<!--more-->

## 常见的IO模型

### BIO，阻塞IO
> 排队

![](https://cp-images.oss-cn-hangzhou.aliyuncs.com/PGtSwc.png)
当一个IO请求过来，客户端进入等待，服务端处理数据，并保持阻塞（其他的客户端访问不了）。

等到服务端处理完数据，返回给等待中的客户端。

很像我们排队买鸭子的过程（没错，我在南京）。一个一个来。

他的优点是，模型很简单，容易实现，且不容易出问题。
确定也很明显，在客户多了的时候，体验会很不好。

### NIO，非阻塞IO
> 轮询

![](https://cp-images.oss-cn-hangzhou.aliyuncs.com/F7rzu9.png)
客户端在执行了IO请求之后，会立刻返回结果，要不返回结果，要不还需等待。
如果还需要等待的话，客户会发起轮询，不断的请求服务端，直到返回结果。

比如我去买一个手抓饼，我下单了之后，需要等待，并且过一段时间，我会问一下老板，
我的手抓饼好了没，如果没有，再过一段时间，再问下老板，我的手抓饼好了没。
直到老板把手抓饼给我。

他的优势是它不会再阻塞了，不用排队等在那里。我可以干一些别的事情。

缺点也是有的，我需要不断的询问，这样也很消耗性能。
而且如果我需要的结果（手抓饼）已经准备好，但是我没有及时询问，
这种情况，客户端拿到数据的时间会晚于真实的数据时间。

### IO多路复用
> 事件处理

![](https://cp-images.oss-cn-hangzhou.aliyuncs.com/BqbARO.png)
IO多路复用，它是接受各种事件（比如连接、读取或写入、错误发生等)。
接受到各种事件消息后，记录下来，并返回一个描述符。

等到数据准备完成，再根据描述符找到是哪个客户端，并通知客户端来取数据。

打个比方，就像我们排队买奶茶，下单之后，拿到一个号码。
等到奶茶做好来之后，奶茶店会叫号，让我们去取奶茶。

它的优点是可以释放客户端，不需要阻塞。
因为仅仅是接受事件，一个线程就够了，不要很多线程来处理请求。

像Redis，Nginx都是使用的IO多路复用模型。

### AIO、异步IO模型
> 回调

![](https://cp-images.oss-cn-hangzhou.aliyuncs.com/FZRyq2.png)

这种IO模型很像JS中的AJAX，发起一个请求之后，继续往下之后后续逻辑。
等到数据返回之后，走到回调方法，并发数据结果带过来。

这种就像点外卖，下单之后，等到外卖准备好，直接把外卖送到家。

优点就是，非常高效，客户端有着最佳的性能。
缺点就是，模型结构复杂，不是所有操作系统都能支持。

## Netty项目结构
![](https://cp-images.oss-cn-hangzhou.aliyuncs.com/7qmm3O.png)

### 三大核心

#### 1、字节缓冲区

You can define your own buffer type if necessary.

Transparent zero copy is achieved by a built-in composite buffer type.

A dynamic buffer type is provided out-of-the-box, whose capacity is expanded on demand, just like StringBuffer.

There's no need to call flip() anymore.

It is often faster than ByteBuffer.

![](https://cp-images.oss-cn-hangzhou.aliyuncs.com/lkHtRv.png)


#### 2、通用API
对BIO和NIO的通用抽象。

#### 3、事件模型 
pipeline

### 传输服务

### 协议支持

## Netty实战

## Netty在开放平台中的应用

## 总结

## 参考
- https://netty.io/
- http://www.kegel.com/c10k.html
- https://github.com/netty/netty
- https://github.com/netty/netty/wiki/User-guide-for-4.x
- https://netty.io/3.8/guide/#architecture
- https://www.youtube.com/watch?v=I8yy2Cy7dDI
- https://alibaba-cloud.medium.com/essential-technologies-for-java-developers-i-o-and-netty-ec765676fd21
- https://liakh-aliaksandr.medium.com/java-sockets-i-o-blocking-non-blocking-and-asynchronous-fb7f066e4ede