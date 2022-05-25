---
title: Java操作Redis
date: 2022-04-25 10:18:43
tags: 
- redis 
- redisson 
- java
categories: 技术
---
## 介绍

Redis是目前最流行的缓存数据库，完全基于内存操作，速度非常快。
它是我们互联网公司高性能，高并发的基石。
它的功能包括但不限于：提供中央缓存功能，分布式锁，还有特殊数据结构的特殊应用。
本篇文章都会一一说到。

接下来我们会：

- 使用Docker搭建redis环境
- 搭建springboot服务
- 使用redisson来操作redis
- 熟悉redis的五大基本数据类型
- 熟悉使用redis的高级数据结构及其应用

<!--more-->

## 使用Docker搭建redis环境

使用Docker搭建环境非常方便。 在本地安装好Docker后，使用`docker compose`可以方便快捷但搭好环境。

只需要创建`docker-compose.yaml`文件，内容如下：

```yaml
version: '3.9'

services:
  cache:
    image: redis:latest
    restart: always
    ports:
      - '6379:6379'
    command: redis-server --save 20 1 --loglevel warning --requirepass eYVX7EwVmmxKPCDmwMtyKVge8oLd2t81
    volumes:
      - ~/docker_volume/redis:/data
```

注意： command里面的password和volumes映射，按照自己需要配置

再使用命令行，cd到yaml的目录下，运行`docker-compose up`命令。

非常简单。

等启动好了之后，可以用redis工具连接，看下效果。

## 搭建springboot服务

IDEA启动等一个Springboot的项目。我这里使用的是gradle。maven当让也可以。

我的引用如下：

```gradle
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter'
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.redisson:redisson-spring-boot-starter:3.15.6'
    implementation group: 'de.ruedigermoeller', name: 'fst', version: '2.57'
    compileOnly 'org.projectlombok:lombok'
    annotationProcessor 'org.projectlombok:lombok'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    testImplementation 'io.projectreactor:reactor-test'
}
```

我这里使用的官方推荐的redis客户端`Redisson`,相关的依赖是`org.redisson:redisson-spring-boot-starter:3.15.6`和`de.ruedigermoeller', name: 'fst', version: '2.57`。这样我们就可以在我们的代码中使用`RedissonClient`了。

> 干净又卫生

## 使用redisson来操作redis

### keys

因为redis是键值对，大家可以把redis当成一个大的`HashMap`，所以知道它的所有的key很有必要。

```java
	public void keysTest() {
		RKeys keys = redissonClient.getKeys();
		System.out.println(keys.count());// org.redisson.client.protocol.RedisCommands.DBSIZE 命令
		Iterable<String> keysByPattern = keys
				.getKeysByPattern("*"); // org.redisson.client.protocol.RedisCommands.SCAN
		keys.getKeysStream()
				.forEach(System.out::println);// org.redisson.client.protocol.RedisCommands.SCAN
	}
```

Redisson中使用`RKeys`类来对redis中的keys做封装。

我们可以

- 拿到所有key的数量
- 把所有的key列出来
- 按照正则匹配key

注意，在redisson中，使用的不是redis提供的`keys`命令，而是`scan`命令。原因是因为，redis是单线程服务，如果key的量特别大，一次keys操作会消耗很长时间，整体拖慢服务性能。

### string

`String`是redis的最基础的数据结构。

```java
	//String
	@Test
	public void testString() {
		RBucket<String> hello = redissonClient.getBucket("hello");
		System.out.println(hello.get());// org.redisson.client.protocol.RedisCommands.GET
		hello.set("world ! ", 5,
				TimeUnit.MINUTES);// org.redisson.client.protocol.RedisCommands.PSETEX
		System.out.println(hello.get());
	}
```

redis中所有的key都是String类型，而一些对象的存储，都是把对象序列化成为字符串进行存储。

string作为redis中的最基础的数据类型，redis做了极致的优化，这个我们后面再说。

### list

list是redis中列表的数据结构。说实话，在工作中使用的比较少。可以用来做一个简单的消息队列。

```java
	// List
	@Test
	public void testList() {
		RedissonList<Person> list = (RedissonList) redissonClient.getList("list");
		Person tom = Person.tom();
		list.add(tom);// RPUSH
		list.addBefore(tom, new Person());
		list.readAll(); // LRANGE
		list.fastSet(1, new Person());
		System.out.println(list.size());
		list.forEach(System.out::println);
	}
```

我们可以把redis中的list，在Java中，当成LinkedList操作（插入、删除快；查找慢）。 使用简单方便。

list的底层是一个叫快速列表的数据结构（quick list）。

如果所示，它是由ziplist组成，ziplist是压缩列表，是紧凑的，再用link给串起来。

![](https://cp-images.oss-cn-hangzhou.aliyuncs.com/image.png)

### hash

hash，map，dict，说的都是一个东西，在Java中，我们叫做`HashMap`。

```java
	// Map
	@Test
	public void testMap() {
		RedissonMap<String, Object> myMap = (RedissonMap) redissonClient.getMap("myMap");
		myMap.put("Tom", Person.tom());
		RedissonMap<String, Object> map = (RedissonMap) redissonClient.getMap("myMap");
		System.out.println(map.get("Tom")); // HGET
	}
```

使用和Java无二。 当我们使用Map缓存数据时候，我们可以部分的获取一些字段，不用把整个对象都拿出来。

注意：一整个Map能设置过期删除。Map中单独的key-value是设置不了过期删除的。

和Java中`HashMap`不同的是，扩容rehash的过程中，redis当中是渐进式的，可能同时出现两个map。

### set

redis中的set和Java中的HashSet类型。使用方法也类似。

```java
	// Set
	@Test
	public void testSet() {
		RSet<Object> set = redissonClient.getSet("set");
		set.add(1);
		set.add(2);
		set = redissonClient.getSet("set");
		for (Object o : set) {
			System.out.println(o);
		}
	}
```

工作中使用的也不错，可以计算一些UV。

### zset

sort set，排序集合，redis特色结构。

在set的基础上，给每个value加上了分数，在zset中的值都是按照分数排好序的。

```java
	// ZSet
	@Test
	public void testZSet() {
		RSortedSet<Object> zset = redissonClient.getSortedSet("zset");
		zset.add(1);
		zset.add(2);
		zset.forEach(System.out::println);

		RScoredSortedSet<Object> szset = redissonClient.getScoredSortedSet("szset");
		szset.add(11.1, 1);// ZADD
		szset.add(1.1, 2);
		szset.forEach(System.out::println);
	}
```

zset底层使用的跳跃表（skip list），具体后面再说。

![](https://cp-images.oss-cn-hangzhou.aliyuncs.com/1653463112036-image.png)

主要的作用是可以做一些排行榜，像微博热搜。

---

这篇文章我们讲到了如果搭建一个redis环境，并使用`redisson`对redis做一些基本操作。

在下一篇文章中，我们将讲到redis中的一些高级数据结构。

- 位图
- 布隆过滤器
- 队列
- 分布式锁
- 限流器
- HyperLogLog
- Geo
