---
title: MySQL事物原理
date: 2021-08-22 09:12:24
tags: mysql
categories: 知识
---
## 介绍

什么是事物？！

一句话来说，事物是一次操作的`逻辑单位`。要不全部成功，要不全部失败。

它主要是用来保证`数据一致性`的问题。

<!--more-->

比如去银行转账的操作，原来账户的扣款，和目标账户的加款。
这两个操作要放在一个事物里面，要不一起成功，要不一起失败。

如果在事物中间，发生问题，需要把已经执行了的操作`回滚`，以保证数据准确。

## 事物的常见命令

- START TRANSACTION：开始一个事物
- COMMIT：事物顺利完成时，提交事物
- ROLLBACK：事物发生了异常，回滚

比如我们有一张customer表，执行一下语句：

```sql
start transaction;
insert into customer set name = 'zhangsan';
insert into customer set name = 'lisi';
commit ;
```

再查询表，会得到：
![](https://cp-images.oss-cn-hangzhou.aliyuncs.com/Prom8Z.png)
表示一个事物完成、提交。

我们再执行下面操作：

```sql
start transaction;
insert into customer set name = 'wangwu';
rollback ;
```

再查询customer表，会发现wangwu并没有被添加到表中，表中还只有两条记录。

- 建立保存点：SAVEPOINT 保存点名称
- 删除保存点：RELEASE SAVEPOINT 保存点名称
- 回滚到特定保存点：ROLLBACK TO SAVEPOINT 保存点名称

我们接着再执行下面点语句：

```sql
start transaction;
insert into customer set name = 'zhao liu';
savepoint my_point;
insert into customer set name = 'qian qi';
rollback to savepoint my_point;
```

我们查询到到结果是：
![](https://cp-images.oss-cn-hangzhou.aliyuncs.com/gIvWTx.png)
`zhao liu` 成功入库了，而 `qian qi` 被回滚调了。

`savepoint` 保存点，可以让我们实现部分回滚。

## Transaction 四大特性： ACID
![](https://cp-images.oss-cn-hangzhou.aliyuncs.com/NfuAJQ.png)

一个事物有下列四个重要到特性，简称ACID
- A：Atomicity 原子性。 事物里面到操作，不可分割，要不一起成功，要不一起失败。
- C：Consistency 一致性。在事物的前后，系统的整体数据保持一致。比如说银行转账。
- I：Isolation 隔离性。 事物之间，相互隔离，互不干扰。
- D：Durability 永久性。 一旦数据修改完成，永久有效。

![](https://cp-images.oss-cn-hangzhou.aliyuncs.com/9qUodw.png)
![](https://cp-images.oss-cn-hangzhou.aliyuncs.com/xJ1VeA.png)
![](https://cp-images.oss-cn-hangzhou.aliyuncs.com/jez6I1.png)
![](https://cp-images.oss-cn-hangzhou.aliyuncs.com/vXbP4p.png)


## 事物隔离级别

## InnoDB的事物实现（MVCC）

## 实操

## 总结
