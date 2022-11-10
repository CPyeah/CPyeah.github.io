---
title: 计算机网络
date: 2022-01-18 16:30:52
tags: 
- network
categories: basic
---
## 历史与简介
- 电话，电话网络
- 邮电局，接线员与单点故障
- 冷战背景
- 核弹炸不坏的通信
- IP internet protocol (网络层)
- IPv4 vs IPv6

<!--more-->

## 网络分层模型
分层模型
分层的意义
以快递配送做比喻

![](https://cp-images.oss-cn-hangzhou.aliyuncs.com/DoyVUQ.png)


## TCP与UDP （传输层）
### TCP

三次握手、四次挥手
建立链接，路线确定
发送消息，基于流
避免装包、拆包
速率控制
失败重传

端口级别

很多协议的基石
HTTP(s)、FTP、Dubbo

![](https://cp-images.oss-cn-hangzhou.aliyuncs.com/SxZ9xf.png)

特点：
面向链接
点对点 P2P
可靠交付
面向流

### UDP
无连接
尽量交付
面向报文

把数据封装成一个一个的数据包，然后就不管了

视频、语音聊天  可以容忍丢包

## Socket、HTTP与HTTPS

Socket
一个Socket连接：四元组：src ip, src port, dest ip, dest port
建立一个Socket连接，就是建立了一个传输通道


对TCP协议对又一层封装

基于包对概念

一个请求 或 响应 就是一个数据包

粘包，拆包

Head and body

### 手动实现一个简单的Http服务

