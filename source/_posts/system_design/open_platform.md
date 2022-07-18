---
title: SystemDesign:开放平台
date: 2021-10-18 17:38:34
tags: 
- system_design
- open_platform
categories: 系统设计
---

> - 客户端注册管理
> - 客户端权限验证
> - 客户端调用内部接口
> - 消息接收

<!--more-->

## 使用场景
MVP
- 用户注册app使用权限
- 后台管理系统
- 用户调用接口
- 消息回调
- 用户Oauth2权限验证
- 接口文档展示

目标：
- 系统高可用
- 消息不丢失
- 数据持久化

加分项：
- Error Trace
- 统计分析
- mock invoke in document
- 沙箱

## 约束
- 用户：10万 不需要partition
- RPS：5k /user/day = 5K
- Message 1K/user/day = 1K

## 架构
- apply appKey/appSecret
  console server
- getToken
    - OAuth2 server
- invoke interface
    - api server
- message server
- doc server
  
![](https://cp-images.oss-cn-hangzhou.aliyuncs.com/openplatformsystem.png)


### 后台管理服务（console server）
内部管理员操作
- 用户管理
- appId、appSecret管理
- token管理
- 内部系统用户与client绑定管理
### 权限认证服务（OAuth2 server）
- 客户端获取token
- token校验
### 接口服务（API server）
- 类似一个网关
- 客户端参照文档，调用接口（同步）
- 校验token，验证签名
- 调用内部系统的接口
- 消息订阅
- 请求返回信息给分析服务（异步）
### 分析服务（analysis server）
- 接受所有的请求、消息
- 记录分析、生成仪表板
- 大数据
### 消息服务（message server）
- 监听内部系统消息
- 消息记录（状态：Queue｜Sent｜Fail）
- 客户端消息订阅查看
- 异步发送已订阅的消息（Rabbit Queue）
- 消息给分析服务（异步）

## 数据库选择
### 后台管理服务
变化少 - 关系型数据库MySQL

### 认证服务
调用频繁，过期，读多写少 - redis

### 消息服务
消息metaData保存，消息状态，事务 - MySQL
异步发送消息，消息队列 - Rabbit/Kafka

### 统计分析服务
数据量大，实时分析 - Mongo/KafKa/Flink


## API 设计
### 管理员
createUser(innerId)  appId,appSecret
dashboard()
CRUD(userInfo)
triggerGenerateDocWebSite()

### 客户端
token = generateToken(appId, appKey)
token = refreshToken(token, appId, appKey)
result = invokeInterface(data, appId, token, sign)

listenMessage(netty/socket)

### 内部接口
#### API server
innerId = verityTokenAndSign(appId, token, sign)
invokeInnerInterface(data, innerId)
sendRequestToAnalysis(requestInfo, metaData)

#### Message server
listenInnerSystemMessage

filterAndSaveAndQueueMessage(message)

sendMessageAndChangeState(message)

sendMessageToAnalysis(messageInfo, metaData)

## 数据模型
Client:
- id
- innerId
- appKey
- appSecret
- token
- token overtime
- client info
- subscribe topic list

Message:
- id
- type
- status(Queue｜Sent｜Fail)
- create_time
- finish_time

Analysis Model
- id
- appId (partition key)
- clientInfo
- type (request/message)
- interfaceName
- messageTopic
- cost
- requestResult(success/fail)
- requestBody(json)
- createTime
- invokeTime



