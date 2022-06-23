---
title: MySQL事物原理
date: 2021-08-25 09:12:24
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
`zhao liu` 成功入库了，而 `qian qi` 被回滚掉了。

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

介绍完了ACID，我们再来重点说说ACID其中的I(Isolation)。

我们都知道，事物之间是隔离的
但是如何处理多个事物之间操作和读取同一个数据的结果，
我们需要仔细考虑。

事物隔离级别是在多个事务同时进行更改和执行查询时，
调整`性能`和结果的`可靠性`、`一致性`和`可再现性`之间的平衡的设置。

MySQL（InnoDB）提供四种事物隔离级别：

- read uncommitted 读未提交
- read committed   读已提交
- repeatable read  可重复读（MySQL默认隔离级别）
- serializable     串行

我们可以参考[官方文档](https://dev.mysql.com/doc/refman/8.0/en/innodb-transaction-isolation-levels.html)

### 事物隔离级别的设置

```sql
# 设置全局的事物隔离级别
SET GLOBAL TRANSACTION LEVEL [REPEATABLE READ | READ COMMITTED | READ UNCOMMITTED | SERIALIZABLE];
# 设置当前Session的隔离级别
SET SESSION TRANSACTION ISOLATION LEVEL [REPEATABLE READ | READ COMMITTED | READ UNCOMMITTED | SERIALIZABLE];
# 查看当前事物隔离级别
SELECT @@transaction_ISOLATION;
```

我们先做一下前置的准备，
开启两个Session，
![](https://cp-images.oss-cn-hangzhou.aliyuncs.com/Lf8Nzf.png)

并查看一下当前的事物隔离级别
![](https://cp-images.oss-cn-hangzhou.aliyuncs.com/3zaxpJ.png)

接下来，我们一个一个看：

### 未提交读（READ UNCOMMITTED）

未提交读，指的就是一个事物读到了另外一个事物未提交的数据。

这样会导致脏读。

我们来实验一下：

我们先设置两个Session的隔离级别：

```sql
SET SESSION TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
```

查看下：
![](https://cp-images.oss-cn-hangzhou.aliyuncs.com/WNJBVz.png)

我们现在数据库当中有两条数据

```sql
select * from customer;
```

![](https://cp-images.oss-cn-hangzhou.aliyuncs.com/trxTdG.png)

我们在Session A中开启一个事物，并修改一条记录，但是不提交。

再在Session B 中查询，看会得到怎样的结果。

我们现在Session A 中执行以下操作：

```sql
# session A
# 1、开启事物
start transaction ;

# 2、修改数据
update customer set name = 'zhangsan2' where id = 1;
```

我们在Session B 中看下结果：

```sql
# session B
# 3、查看数据
select * from customer where id = 1;
```

此时Session A中的事物还没有提交，也就是说有回滚的可能。
但在Session B中查看数据， 客户名称已经从 `zhangsan` 变成了 `zhangsan2`。

如果Session A再进行了回滚，rollback。 这样Session B就读出了一个不存在的数据，就是脏读。

这是非常不安全的一种隔离级别。

### 已提交读（READ COMMITTED）

已提交读，就是读到的数据都是另外的事物已经提交过了的数据。
可以解决脏读的问题。

我们先修改下事物隔离级别：

```sql
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;
SELECT @@transaction_ISOLATION;
```

![](https://cp-images.oss-cn-hangzhou.aliyuncs.com/5fAmh8.png)

我们在SessionA中修改数据：

```sql
# session A
# 1、开启事物
start transaction ;

# 2、修改数据
update customer set name = '张飞' where id = 1;
```

在Session B 中查看数据：

```sql
# session B
# 3、查看数据
select * from customer where id = 1;
```

![](https://cp-images.oss-cn-hangzhou.aliyuncs.com/YMiybO.png)
我们可以看到，客户名称还是`zhangsan`，没有变成`张飞`。

我们这个时候再把Session A中的事物提交

```sql
# session A
# 1、开启事物
# start transaction ;

# 2、修改数据
# update customer set name = '张飞' where id = 1;

# 4、提交事物
commit ;
```

再次在Session B 中查看数据：

```sql
# session B
# 3、查看数据
select * from customer where id = 1;

# 5、再次查看数据
select * from customer where id = 1;
```

![](https://cp-images.oss-cn-hangzhou.aliyuncs.com/wY9JUG.png)
我们可以看到，客户名称已经变成了 `张飞`。

如果session A的事物回滚的话，我们在Session B中看到的结果还会是 `zhangsan`
这样，我们就成功解决了脏读的问题了。[](https://)

但是如果我们在同一个事物中，读取数据，可能会查询到不同的结果。
我们按照数字的顺序，执行以下SQL：

```sql
# session B

# 1、开启事物，并查看数据
start transaction ;
select * from customer where id = 1; -- 张飞

# 3、再次查看数据
select * from customer where id = 1; -- 关羽

# 5、再次查看数据
select * from customer where id = 1; -- 刘备
```

```sql
# session A

# 2、修改数据（自动提交事物）
update customer set name = '关羽' where id = 1;

# 4、修改数据（自动提交事物）
update customer set name = '刘备' where id = 1;
```

我们可以看到，Session B 在同一个事物中，同样的查询语句，得出的结果是不一样的。
这种现象我们称之为`不可重复读`。

### 可重复读（REPEATABLE READ）

### 串行化（SERIALIZABLE）

## InnoDB的事物实现（MVCC）

## 总结
