---
title: MySQL调优
date: 2021-09-11 13:47:39
tags: mysql
categories: database
---
## 介绍

这篇文章，我们将谈到SQL优化的相关知识。

- 我们会讲到如何合理定义数据结构
- 如何设计高效的索引
- 如果写出高效的SQL语句
- 如果发生了慢查询，我们如何分析
- 如何选择其他的组件来替换MySQL

现在，我们开始吧！！

<!--more-->

## 数据结构的优化

### 数据类型的选择

#### 使用最佳的数据类型

数据类型越短越好，
越小的数据类型，往往意味着更好的效率。

整型的效率会比字符串的性能高
使用Date类型来存储时间，比字符串类型的效率来的高。

#### 尽量使用非空

null会影响到索引效率，查询的返回结果。
可会可能引发业务层的空指针。

我们可以给字段设为not null，且设定一个默认值。

#### 可以使用UNSIGNED来修饰数字类型

无符号数字的存储效率是有符号数组的将近一倍。

比如ID，金额等一些确定是正数的数字类型，
我们可以使用UNSIGNED来修饰。

### 表的设计

#### 避免宽表

宽表意味着一条记录有很多字段。
我们尽量避免一张表有多于100个字段。

他会在查询的时候，增加性能的负担。
尤其是使用alter操作来改变表结构的时候，
会非常消耗性能。

我们可以给一张宽表拆分成不同的小表。
比如：一张订单表，我们可以拆分成
订单主表；订单金额表；订单商品表；订单物流表 等等。

#### 范式 and 反范式

满足范式的设计，往往会更加精简，
但是如果需要查询更多的信息，需要连表查询。

比如说，订单表里面只有买家ID这个字段，
在页面上展示的时候，往往是需要展示买家名称，
这个时候，就需要连表查询，关联订单表，和客户表，
而在我们分布式系统中，更需要单独调用别的中台的接口，来填充信息。

而反范式，就是把一些数据给冗余下来，以提高查询效率。

在高并发，高性能，高数据量的时候，
往往都是单表查询，
反范式，冗余会使用的更多。
冗余可以大大简化我们的SQL，提高我们索引的命中率。

## 索引优化

> 好的索引，是查询效率提升的关键

### 什么时候添加索引

如果我们表的数据只有十几行甚至几行的时候，
我们不要添加索引。
因为这个时候，全表扫描的效率往往更好，
而索引的维护，还会影响到数据的修改效率。

如果我们的表数据量很多，
我们推荐设计合适的索引来提高查询效率。

如果我们的表数据非常非常多，
我们考虑使用分区，分表

### 添加合适的索引

在查询条件上，
在关联字段上，
在排序字段上，
在分组字段上，
可以添加索引。

添加索引的字段值区分度越高越好，比如ID
像枚举类型的值不适合设置成索引。

索引不是越多越好，索引会占用空间，索引会影响修改数据的效率。

### 组合索引

根据业务中的查询条件，添加合适的组合索引
遵循最左匹配原则

### 使用覆盖索引

```sql
select id from order where buyer_id = '';

select * from order where buyer_id = '';
```

如果我们在订单表（order）中的买家ID（buyer_id）上建立索引，

上面第一条的SQL会比第二条的执行速度快很多。

因为第一条索引直接走了覆盖索引，而不用回表查询，大大提高效率。

### 精简索引字段

如果索引字段太长，且有很多重复的字符，我们可以截取一部分来设成索引。

可以减少索引体积。

比如说email，或者网站地址，或者非常长的ID

把区分度高的部分，截取出来，设成索引。

## SQL语句编写的优化

### 基础SQL原则

- 只返回必要的行，避免`select *`这样的语句。
- where条件里面控制范围
- limit 控制返回条数
- 缓存热点数据，使用服务端缓存（redis），避免使用MySQL缓存
- 查询条件 和 索引相匹配

### count函数优化

一般来说，我们尽量使用`count(*)`，来统计行数，
但是还有一种用法，我们可以使用`count(column_1)`来统计，column_1不为空的记录数。

在column_1不为空的情况下：
count(*) == count(column_1)

在column_1有null值的情况下：
count(*) > column_1

这点需要大家注意区别。

在不需要准确数据的情况下，
我们可以使用 explain 来替代 count。

explain会给出一个总数目的估计值，
explain只会给出一个查询计划，并不会真正去存储引擎上执行，
它的执行效率是高的。

### 避免深分页

我们要避免这样的SQL。

```sql
select * from table limit 1000, 10;
```

这样会先把前面的1010条数据都查询出来，在截取后10条记录返回。
很营销效率。

在这种情况下，我会推荐使用游标的来查询。

```sql
select * from table where id > '' limit 10;
```

### 分解大批量

如果我们需要清楚历史日志数据，我们可能会这样：

```sql
delete from log where create_time < '';
```

如果历史数据非常大，这样会非常影响数据库的性能，
并会锁住很多记录，占用系统资源。

可能会让很多小而重要的查询发生中断。

我们需要这样做：

```sql
delete from log where create_time < '' limit 1000;
```

并在业务代码中，循环删除。

## 慢查询及其分析

### 开启慢查询监控

我们可以先查看一下配置，直接执行下面的SQL：

```sql
show variables like 'slow%';
```

得到效果如下：
![](https://cp-images.oss-cn-hangzhou.aliyuncs.com/O1oh8f.png)
很明显，我还没有开启慢查询监控。

我们继续再修改配置，开启监控：

```sql
set global slow_query_log=ON;
```

在设置一下触发时间：

```sql
show variables like 'long%';
set global long_query_time = 1
```

默认的慢查询的触发时间是10秒，太长了
我们配置成合适的触发时间， 我这里配置的是1秒。

Tip：配置好之后，需要重新打开新的session查看配置。

一切完成之后，如果我们再发生了慢查询，就会在对应的位置生成log啦。
我们再可以使用explain分析SQL

### 使用EXPLAIN来分析SQL执行计划

首先上[官方文档](https://dev.mysql.com/doc/refman/8.0/en/explain-output.html)
最权威

explain SQL 是一个非常复杂的过程，里面的细节很多，
在这里只做一个简单的介绍，和常用的排查慢查询的方法。

#### 执行和字段解释

我们拿到了慢查询的SQL，直接再语句的前面加上一个`explain` 就行。

```sql
explain select * from customer where id = 1
```

我们可以看到一下结果：
![](https://cp-images.oss-cn-hangzhou.aliyuncs.com/FZy098.png)

一共返回了12个字段，我来看一下每一个字段代表什么意思。


| Column                                                                                               | JSON Name       | Meaning                                                                                                                                                         |
| ------------------------------------------------------------------------------------------------------ | ----------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [`id`](https://dev.mysql.com/doc/refman/8.0/en/explain-output.html#explain_id)                       | `select_id`     | 表示查询中执行select子句或者操作表的顺序，id的值越大，代表优先级越高，越先执行                                                                                  |
| [`select_type`](https://dev.mysql.com/doc/refman/8.0/en/explain-output.html#explain_select_type)     | None            | 表示 select 查询的类型，主要是用于区分各种复杂的查询，例如：普通查询、联合查询、子查询等                                                                        |
| [`table`](https://dev.mysql.com/doc/refman/8.0/en/explain-output.html#explain_table)                 | `table_name`    | 查询的表名，并不一定是真实存在的表，有别名显示别名，也可能为临时表                                                                                              |
| [`partitions`](https://dev.mysql.com/doc/refman/8.0/en/explain-output.html#explain_partitions)       | `partitions`    | 查询时匹配到的分区信息，对于非分区表值为NULL，当查询的是分区表时，partitions显示分区表命中的分区情况                                                            |
| [`type`](https://dev.mysql.com/doc/refman/8.0/en/explain-output.html#explain_type)                   | `access_type`   | 查询使用了何种类型，它在 SQL优化中是一个非常重要的指标                                                                                                          |
| [`possible_keys`](https://dev.mysql.com/doc/refman/8.0/en/explain-output.html#explain_possible_keys) | `possible_keys` | 表示在MySQL中通过哪些索引，能让我们在表中找到想要的记录，一旦查询涉及到的某个字段上存在索引，则索引将被列出，但这个索引并不定一会是最终查询数据时所被用到的索引 |
| [`key`](https://dev.mysql.com/doc/refman/8.0/en/explain-output.html#explain_key)                     | `key`           | 区别于possible_keys，key是查询中实际使用到的索引，若没有使用索引，显示为NULL                                                                                    |
| [`key_len`](https://dev.mysql.com/doc/refman/8.0/en/explain-output.html#explain_key_len)             | `key_length`    | 表示查询用到的索引长度（字节数），原则上长度越短越好                                                                                                            |
| [`ref`](https://dev.mysql.com/doc/refman/8.0/en/explain-output.html#explain_ref)                     | `ref`           | 搜索关联字段，有可能是常数、多表关联字段、函数等                                                                                                                |
| [`rows`](https://dev.mysql.com/doc/refman/8.0/en/explain-output.html#explain_rows)                   | `rows`          | 以表的统计信息和索引使用情况，估算要找到我们所需的记录，需要读取的行数。                                                                                        |
| [`filtered`](https://dev.mysql.com/doc/refman/8.0/en/explain-output.html#explain_filtered)           | `filtered`      | 这个是一个百分比的值，表里符合条件的记录数的百分比。简单点说，这个字段表示存储引擎返回的数据在经过过滤后，剩下满足条件的记录数量的比例                          |
| [`Extra`](https://dev.mysql.com/doc/refman/8.0/en/explain-output.html#explain_extra)                 | None            | 不适合在其他列中显示的信息，Explain 中的很多额外的信息会在 Extra 字段显示                                                                                       |

#### 重点排查顺序
explain了一条SQL之后，我们的排查流程如下：

- type
  - system > const > eq_ref > ref > ref_or_null > index_merge > unique_subquery > index_subquery > range > index > ALL
  - 一般最少要达到range等级，最好达到ref等级
  
- key
  - 实际上用到的索引
  
- key_len
  - 使用到的索引长度，可以算出联合索引中走了几个索引
  - utf8中一个字符占3个字节， 允许null +1， not null +2
  
- Extra
  - 一些额外信息，表示使用了哪些操作
  - 像使用了覆盖索引，索引下推，这都是比较好的
  - 像使用了文件排序，需要优化下，尽量使用索引排序

## 使用其他的组件来替代MySQL

- 一些经常查询，或者修改频次高的数据，我们尽量放到Redis里面
- 如果需要使用到全文索引，首推Elastic Search
- 如果是类似日志的数据，数据量大，不需要连表，不需要复杂的索引，可以使用MongoDB

但是MySQL是一款非常成熟，非常稳定的数据库，在一般的条件下，我们还是会首选MySQL。

## 总结
至此，我们学到了MySQL优化的一些相关知识。
我们从建表开始，选择合适的数据结构，索引的建立，
到生产中，遇到的慢查询的处理。

希望对大家有所帮助！！Peace～
