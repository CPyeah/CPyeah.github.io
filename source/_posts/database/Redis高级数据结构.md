---
title: Redis高级数据结构
date: 2021-04-26 15:15:43
tags: 
- redis 
- redisson 
- java
categories: database
---
## 介绍

这篇文章介绍一下高级Redis数据结构，及其用法。

在工作的使用频率很高。

- 分布式锁
- 队列
- 位图
- 布隆过滤器
- HyperLogLog
- 限流器
- GeoHash

<!--more-->

## 分布式锁

大家都知道，线程安全在我们的系统当中，非常重要。
如果多个线程同时操作一笔订单，很可能造成数据不一致。

而在分布式系统中，`synchronized`就排不上用场了，需要使用分布式锁来解决线程安全的问题。

而使用redis就是一个很好的选择。

使用`Redisson`封装的分布式锁，非常简单，功能也相当强大。

```java
@Test
public void lockTest() throws InterruptedException {
    RLock lock = redisson.getLock("my-lock");
    try {
        lock.lock();
        // do somethings
    } finally {
        lock.unlock();
    }
}
```

除了基础的分布式锁，Redisson还提供了：

- 可重入锁
- 公平锁
- 自旋锁
- 读写锁
- 锁超时时间
- 获取锁的时间限制

功能强大且使用方便，具体参考[官方文档](https://github.com/redisson/redisson/wiki/8.-distributed-locks-and-synchronizers)。

## 队列

队列是我们一种常用的数据结构。也有很多专业的中间件来实现。比如：Kafka，RabbitMQ，RocketMQ。。。
而redis也支持消息队列，而且功能还不少。有：

- 简单的消息队列
- 延迟队列
- 阻塞队列
- 消息的多播

Redis 的 list(列表) 数据结构常用来作为异步消息队列使用，使用`rpush/lpush`操作入队列，使用 `lpop/rpop` 来出队列。

延时队列可以通过 Redis 的 zset(有序列表) 来实现。我们将消息序列化成一个字符串作为 zset 的 value，这个消息的到期处理时间作为 score。

Redisson对此都有很好的支持：

```java
// 简单队列
@Test
public void simple() {
    RQueue<Object> queue = redissonClient.getQueue("simple");

    new Thread(() -> {
        for (int i = 0; i < 50; i++) {
            queue.add(i);
        }
    }).start();
    for (int i = 0; i < 50; i++) {
        Object poll = queue.poll();
        if (poll == null) {
            i--;
        } else {
            System.out.println(poll);
        }
    }
}

// 阻塞队列
@Test
public void blockQueue() throws InterruptedException {
    RBlockingQueue<Object> queue = redissonClient.getBlockingQueue("block");

    new Thread(() -> {
        for (int i = 0; i < 50; i++) {
            try {
                Thread.sleep(100);
                queue.put(i);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }).start();
    for (int i = 0; i < 50; i++) {
        // 阻塞take
        Object poll = queue.take();
        System.out.println(poll);
    }
}

// 延迟队列
@Test
public void delayQueue() throws InterruptedException {
    RBlockingQueue<Object> blockingQueue = redissonClient.getBlockingQueue("delayQueue1");
    RDelayedQueue<Object> delayQueue = redissonClient
            .getDelayedQueue(blockingQueue);
    for (int i = 0; i < 50; i++) {
        delayQueue.offerAsync(i, (500 - i * 10), TimeUnit.MILLISECONDS);
    }

    blockingQueue = redissonClient.getBlockingQueue("delayQueue");
    System.out.println(blockingQueue.size());
    redissonClient.getDelayedQueue(blockingQueue);
    for (int i = 0; i < blockingQueue.size(); i++) {
        Object poll = blockingQueue.take();
        System.out.println(poll);
    }
}

```

以上都是消息队列的`生产者-消费者模式`，而消息队列的另外一种模式`订阅模式`，redis也能实现。

有两种数据结构支持。

- PubSub
- Stream

## 位图

位图完全是为了节约空间而设计的，我们可以把它想象成一个巨大的List，而里面只存放`1/0`。

比如记录一个用户一年的签到记录，我们可以用位图来记录，签到的就是`1`，未签到的是`0`。

使用方法如下：

```java
@Test
public void test() {
    RBitSet bitset = redissonClient.getBitSet("bitset");
    bitset.set(0, true);
    System.out.println(bitset.get(0));
    System.out.println(bitset.get(1));
    System.out.println(bitset.incrementAndGetInteger(0, 1));
}
```

## 布隆过滤器

我们如何判断一个ID是否在我们的系统里。我们有两种基本的方法来判断：

- 我们把这个ID拿到数据查询一下就知道来
- 我们把所有的ID保存的一个Set里面，判断这个新的ID在不在这个Set里面

但是上面两种方法有一个致命的缺陷：
就是在数据量特别大的时候，非常消耗性能，不管是对数据库的IO，还是保存所有的ID。

这个是有个非常精妙的算法 - `布隆过滤器`。

![](https://cp-images.oss-cn-hangzhou.aliyuncs.com/HeAOem.png)

它的原理非常简单：
底层是一个位图，并且有若干种Hash算法。
在新增ID的时候，对每一个ID做多种Hash运算，并在位图中得到若干位置，把对应的位置变成`1`。
而判断一个ID是否在系统中，也对这个ID做同样的Hash运算，得到若干位置，判断位图中的对应位置是否都是`1`。
如果都是`1`的话，表示大概率可能存在于系统中，如果有一个`0`，表示肯定不在系统中。

Redisson也做了封装，使用非常方便。

```java
@Test
public void test() {
    RBloomFilter<String> bloom = redissonClient.getBloomFilter("bloom");
    bloom.tryInit(1000, 0.1);
    for (int i = 0; i < 100; i++) {
        bloom.add("user_id_" + i);
    }
    for (int i = 80; i < 120; i++) {
        boolean contains = bloom.contains("user_id_" + i);
        System.out.println("user_id_" + i + "  ->  " + contains);
    }
}
```

## HyperLogLog

HyperLogLog是一个很神奇的数据结构。

我们先来说两个概念： PV 和 UV。
PV：简单来说就是流量。只要有一次访问，不管是不是同一个人，就记一个流量。
UV：独立访客，单位时间内，一个新的访客，就算一个UV。

在我们的统计中，PV很好统计，来一次访问，PV值就往上加1，顶多需要保持一下自增的原子性。
而计算UV，在互联网领域中一个是一个难题。

通常来说我们计算一个网站一天的UV，我们需要把每一个访客的ID给记录下来，计算时去重。
这样就导致一个问题，如果访问量非常大，就非常的消耗存储空间。

这个时候，HyperLogLog就能派上用场了。

我们先看下代码：

```java
@Test
public void compareTest() {
    RHyperLogLog<String> today = redisson.getHyperLogLog("UV_" + LocalDate.now());
    HashSet<String> set = new HashSet<>();
    Random random = new Random();
    for (int i = 0; i < 1000; i++) {
        String userId = "id_" + random.nextInt(500);
        today.add(userId);
        set.add(userId);
    }
    System.out.println("RHyperLogLog: " + today.count());
    System.out.println("set count: " + set.size());
}
```

这里使用的Set方法，和HyperLogLog，两个做一个比较。
从结果可以看出，两者的差距非常小，但是HyperLogLog并没有把每一个ID都记录下来，节约了大量存储空间。

HyperLogLog的原理相对复杂，具体可以参考这篇[论文](http://algo.inria.fr/flajolet/Publications/FlFuGaMe07.pdf)。

这里我来介绍一下它的思想：
假设我们有两个网站，需要比较一下他们的UV，看哪个网站的访问率高。
假设访问用户的ID使用手机号码表示，如：1xxxxxxxxxx 
我们不想把每个访客的手机号码都记录下来，因为太浪费存储空间了。
我们可以定一个规则，我们只记录当天客户中，手机尾号为`0`位数最高的手机号码。
比如说，第一个网站记录下来的手机号码是：18322333000，末尾有3个0，
而第二的网站记录下来的手机号码是：18623000000，末尾有6个0。
我们就可以判断出，第二的网站的访问的人数更多。
大家体会一下。

HyperLogLog还有一个重要的概念，就是合并。
我们可以把连续7天的UV，合并成这一周的UV，并保证准确性。
```java
@Test
public void mergeTest() {
    RHyperLogLog<String> today = redisson.getHyperLogLog("UV_" + LocalDate.now());
    RHyperLogLog<String> nextDay = redisson.getHyperLogLog("UV_" + LocalDate.now().plusDays(1));
    HashSet<String> set = new HashSet<>();
    Random random = new Random();
    for (int i = 0; i < 1000; i++) {
        String userId = "id_" + random.nextInt(1000);
        today.add(userId);
        set.add(userId);
    }
    for (int i = 0; i < 1000; i++) {
        String userId = "id_" + random.nextInt(2000);
        nextDay.add(userId);
        set.add(userId);
    }
    today.mergeWith(nextDay.getName());
    System.out.println("RHyperLogLog: " + today.count());
    System.out.println("set count: " + set.size());
}
```

## 限流器
限流器在我们的生产中非常重要，也很常用。Redis也有封装好的限流器。
实现了分布式限流器的功能。
代码如下：
```java
@Test
public void test() throws InterruptedException {
    // redisson用了zset来记录请求的信息，这样可以非常巧妙的通过比较score，也就是请求的时间戳，来判断当前请求距离上一个请求有没有超过一个令牌生产周期。
    // 如果超过了，则说明令牌桶中的令牌需要生产，之前用掉了多少个就生产多少个，而之前用掉了多少个令牌的信息也在zset中保存了。
    RRateLimiter limiter = redissonClient.getRateLimiter("limiter_1");
    // 1号限流器， 每一秒 允许一个请求
    limiter.trySetRate(RateType.PER_CLIENT, 1, 1, RateIntervalUnit.SECONDS);

    for (int i = 0; i < 10; i++) {
        new Thread(() -> {
            // 阻塞
//			limiter.acquire(1);
            boolean result = limiter.tryAcquire(1, 100, TimeUnit.MILLISECONDS);
            if (!result) {
                // 快速失败
                System.out.println(Thread.currentThread().getName() + " failed !");
                return;
            }
            System.out.println(Thread.currentThread().getName() + " -> " + LocalDateTime.now());
            // do something
        }).start();
    }
    Thread.sleep(12000);
}

```
> Redisson真好用！

## Geo
Redis也提供了Geo功能，可以很方便实现附近的人，两坐标之间的距离等功能。
它的原理是把经纬度坐标，做Hash，得到一个分数，用zset做存储。
Hash算法的原理，是把整个地球当作一个平面，并把这个平面切割成很多很多的小块，并对每个方块做编码。
如果两个点所在的方块越接近，表示这两个点越接近。 当然，有一点点误差。
使用方法如下：
```java
@Test
public void test() throws ExecutionException, InterruptedException {
    RGeo<Object> geo = redissonClient.getGeo("geo");
    GeoEntry juejin = new GeoEntry(116.48105, 39.996794, "juejin");
    geo.add(juejin);
    geo.add(116.514203, 39.905409, "ireader");
    geo.add(116.489033, 40.007669, "meituan");
    geo.add(116.562108, 39.787602, "jd");
    geo.add(116.334255, 40.027400, "xiaomi");

    // 两点距离
    System.out.println(geo.dist("meituan", "jd", GeoUnit.METERS));
    // 坐标
    System.out.println(geo.pos("xiaomi"));
    // hash
    System.out.println(geo.hash("ireader"));
    // 附近
    System.out.println(geo.radiusWithPositionAsync("jd", 20, GeoUnit.KILOMETERS).get());
    // 附近
    System.out.println(
            geo.radiusWithPositionAsync(116.334255, 40.027400, 20, GeoUnit.KILOMETERS).get());
}
```
注意： 在一个地图应用中，车的数据、餐馆的数据、人的数据可能会有百万千万条，
如果使用Redis 的 Geo 数据结构，它们将全部放在一个 zset 集合中。
在 Redis 的集群环境中，集合可能会从一个节点迁移到另一个节点，如果单个 key 的数据过大，会对集群的迁移工作造成较大的影响，
在集群环境中单个 key 对应的数据量不宜超过 1M，否则会导致集群迁移出现卡顿现象，影响线上服务的正常运行。 
所以，这里建议 Geo 的数据使用单独的 Redis 实例部署，不使用集群环境。
如果数据量过亿甚至更大，就需要对 Geo 数据进行拆分，按国家拆分、按省拆分，按市拆分，在人口特大城市甚至可以按区拆分。
这样就可以显著降低单个 zset 集合的大小。

## 总结
今天我们一起学习了7种高级结构
- 分布式锁
- 队列
- 位图
- 布隆过滤器
- HyperLogLog
- 限流器
- GeoHash
希望以后大家能在工作中用到。