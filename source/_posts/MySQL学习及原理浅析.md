---
title: MySQL学习及原理浅析
date: 2021-08-13 11:06:53
tags: mysql
categories: 知识
---
## 介绍
从今天开始，我们将学习目前最流行的一个数据库MySQL。

这篇文章，带大家走进Mysql的世界。

- MySQL的底层结构
- MySQL的查询过程
- 索引结构
- MySQL的三种Log
- MySQL的存储引擎

<!--more-->
资料：
https://github.com/dunwu/db-tutorial#mysql
https://app.yinxiang.com/Home.action?hpts=1655089226286&loginToken=S%3Ds30%3AU%3D1c6b1dc%3AE%3D1815b38508d%3AC%3D1815b01620f%3AP%3D5fd%3AA%3Den-web%3AV%3D3%3AH%3D073d44951df05e0ba4c664d44136f44b&wechatLogin=true&login=true&hptsh=DihA%2BKmd04hTxUWfh9C0eYdK4Eg%3D#n=30c25c81-a703-4934-8724-11d270bc6c2a&s=s30&ses=4&sh=2&sds=2&

## MySQL的底层结构
MySQL我们可以主要分成三层：

![](https://cp-images.oss-cn-hangzhou.aliyuncs.com/v3Tr6K.png)

1. 客户端连接层。主要负责处理客户端的连接，用户的账号密码校验，权限等功能。
2. 核心服务层。主要功能有缓存，对SQL语句的解析和优化，对底层API的调用。
3. 存储引擎。 对数据对管理，存储，查询，事物，索引等等功能。

## MySQL的查询过程

一个SQL的查询过程可以分成六步。 如图：
![](https://cp-images.oss-cn-hangzhou.aliyuncs.com/mkccLn.png)
1. 客户端和MySQL建立连接。
2. 查询缓存，如果缓存命中，直接返回。但是不建议使用缓存，命中率低，弊大于利。
3. 解析SQL，生成语法解析树。如果SQL写的有问题，这一步会报错。
4. 查询优化，通过语法树，生成执行计划，计划可能不止一个，优化器会找到成本最小的一个执行计划。
5. 执行计划，查询执行引擎拿到了执行计划，调用对应的存储引擎的接口，来执行查询，得到结果。
6. 返回结果，缓存结果。如果结果集很大（1W条），不会等到全部结果出来再返回，会在第一条结果出来的时候，就开始返回结果。

## MySQL的三种Log
MySQL有三种Log，各司其职。分别是binlog、redo log、undo log。
### binlog
binlog的主要作用是记录数据库的每一步变化。由MySQL核心服务层控制。

可以让别的服务来监听，实现数据同步。

或者再数据库损坏之后，恢复数据。

数据库的每一次变化，比如某一个表新增了一条数据，就会生成一个对应的binlog。
另外的服务监听到了之后，可以做出对应的操作，把数据同步一份。

可以做主备，可以做读写分离。

### redo log
首先我们先要了解到一点，磁盘的IO昂贵，数据库再做数据变更的时候，都不会一有变更，就立即更新磁盘数据。

做法都是先在内存中进行操作，等到合适的时候（配置决定），再一次性的刷到磁盘当中。

但是如果数据在内存当中变更了之后，数据库崩溃了，这个时候内存中的数据就会丢失。

MySQL就会有一个redo log来保证上述情况的数据不丢失。

redo log由存储引擎控制。

当MySQL修改数据的做法如下：
1. 把对应数据的页，查询出来，并保存到内存中
2. 修改内存中的数据，同时生成redo log（哪一页，修改了那些内容）。原子操作
3. 如果这时MySQL崩溃，可以根据redo log恢复内存中的数据。
4. 如果没有崩溃，会在合适的时候，把内存中的数据落到磁盘上，同时删除对应的redo log。

### undo log
undo log和事物有关。InnoDB存储引擎特有。

在一个事物中，每一次操作，都会生成一个log，所有的log会组成一个数据链。

可以用来控制回滚，和在不同的事物级别下，所展示的数据的版本。

这就是MVCC，多版本并发控制。我们在事物那一章的时候，还会详细说明。

## 索引结构


## MySQL的存储引擎

## 总结