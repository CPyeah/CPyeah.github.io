---
title: ElasticSearch学习及原理浅析
date: 2021-05-27 15:29:29
tags:
- elasticsearch
- java 
categories: 知识
---

## 简介
这篇文章我们一起来学习一下最最流行的搜索引擎 - `ElasticSearch`
我们将学习到：
- ES集群、节点类型、分片
- 服务发现机制
- 选举机制
- Docker搭建集群环境
- 基础API操作
- ES的数据结构及文件系统
- 存储文档的过程
- 索引及搜索

<!--more-->

![](https://cp-images.oss-cn-hangzhou.aliyuncs.com/16448952278198.jpg)
重点特性有以下几条：
- 存储、管理数据（文档）
- 快速搜索数据
- 对于多种数据格式的高效索引
- 分布式支持高扩展性
- 弹性伸缩支持高可用性
- TA的好搭档-Kibana

## 集群
### 集群概述
![](https://cp-images.oss-cn-hangzhou.aliyuncs.com/16448956295920.jpg)

单节点服务往往能干的很少，而且容错很低，特别是在大量数据的情况之下，所以为了提高服务的整体的高可用、高扩展性，一个服务往往都是一个集群。

而在ES的集群中，由若干个不同角色的节点组成（ES进程），一个集群有且只有一个master节点。

ES中的数据是由索引（Index）管理的，每一个索引相当于数据库中的表。

索引通过分片的方式，把数据横向分为若干分片，放在不同的节点上，进行分布式的存储。类似于数据库的分表操作。

分片可以设置副本，防止在宕机下的数据丢失。

### 节点类型
* master # 主节点
* data # 通用数据节点
* data_content # 数据目录节点，数据不常变化
* data_hot # 热点数据节点，数据的生命周期
* data_warm # 中温数据节点
* data_cold # 冷数据节点
* data_frozen # 封存数据节点
* ingest # 数据摄入节点， 只用于执行预处理管道
* ml # 机器学习节点
* remote_cluster_client # 远程集群节点
* transform # 转换节点
  * 转换节点会进行一种特殊操作，通过特定聚集语句计算，然后将结果写到新的索引中。如果需要使用远成集群数据，请务必在转换节点中添加remote_cluster_client；转换节点设置方法

#### Tips：
1. 一个集群的简单配置，需要确保有`mater`节点和`data`节点。
2. 如果生产环境，且有机器学习（machine learning）任务或转换（transform）任务（**CPU密集型**），建议将候选的主节点（Master-eligible node）与数据节点（`data node`）、机器学习节点（`machine learning node`）和转换节点（`transforming node`）分开是很有必要的。
3. 每个节点都默认为协调节点（Coordinating node），如果`node.roles`设置为`[]`那么该节点将只执行协调节点功能。

### 分片
![](https://cp-images.oss-cn-hangzhou.aliyuncs.com/16448967886435.jpg)
> 数据分片指按照某个维度将存放在单一数据库中的数据分散地存放至多个数据库或表中以达到提升性能瓶颈以及可用性的效果。
- 提升总体数据的存储量（通过分布式）
- 提升数据可用性（通过副本）
- 数据管理的复杂度会提升

### 服务发现
![](https://cp-images.oss-cn-hangzhou.aliyuncs.com/16449078323130.jpg)

假如有三个节点，那么他们应该达成共识，并全部都知道这个集群是个什么样子的。 图中是个反例。

#### 7.x之前之后的实现不同
![](https://cp-images.oss-cn-hangzhou.aliyuncs.com/16449067746885.jpg)
![](https://cp-images.oss-cn-hangzhou.aliyuncs.com/16449077934455.jpg)
![](https://cp-images.oss-cn-hangzhou.aliyuncs.com/16449069256031.jpg)

- 类gossip算法
- 与种子节点互相交换信息
- 达到最终的共识

### 选举选主
#### 选举-Bully（7.x之前）
![](https://cp-images.oss-cn-hangzhou.aliyuncs.com/16449089276566.jpg)
1. 节点1向节点，节点3发送选举，并且带上自己的序号1
2. 节点2，3接收到消息之后，进行序号比较，发觉自己的序号更大，向节点1返回应答消息Answer (Alive) Message，告知节点1被踢出选主序列
3. 节点2向节点3发送选举请求，节点3找不到更高序号的节点发送选举请求了
4. 节点3向节点2返回应答消息，节点3收不到其他节点的应答消息了
5. 节点3被认为是leader，向其他节点发送Coordinator Message，选举成功的请求，将自己是master节点广播到节点1，节点2

#### 选举-类Raft（7.x之后）
![](https://cp-images.oss-cn-hangzhou.aliyuncs.com/16449090661855.jpg)
举例来说，如果node1，node2，node3进行选主
- 如果node1当选leader，
- 但是node2发来了投票要求，那么node1无条件退出leader状态，node2选为主节点
- 但是node3也发来了投票要求，那么node2退出leader状态，node3当选主节点。
  保证最后当选的leader为主leader

相比于Raft算法，Es的选主算法有如下不同
1. 初始为 Candidate状态
2. 允许多次投票，也就是每个有投票资格的节点可以投多票
3. 候选人可以有投票的机会
4. 可能会产生多个主节点

## 本地docker搭环境
```yaml
version: '2.2'
services:
  cerebro:
    image: lmenezes/cerebro:0.9.4 
    container_name: cerebro
    ports:
      - "9000:9000"
    command:
      - -Dhosts.0.host=http://es01:9200
    networks:
      - elastic
  kibana:
    image: docker.elastic.co/kibana/kibana:7.13.2
    container_name: kibana
    environment:
      ELASTICSEARCH_URL: http://es01:9200
      ELASTICSEARCH_HOSTS: http://es01:9200
      LOGGING_VERBOSE: "true"
    ports:
      - "5601:5601"
    networks:
      - elastic
  es01:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.13.2
    container_name: es01
    environment:
      - node.name=es01
      - cluster.name=es-docker-cluster
      - discovery.seed_hosts=es02,es03
      - cluster.initial_master_nodes=es01,es02,es03
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms128m -Xmx128m"
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - /Users/chengpeng/docker_volume/elasticsearch/data01:/usr/share/elasticsearch/data
    ports:
      - 9200:9200
    networks:
      - elastic
  es02:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.13.2
    container_name: es02
    environment:
      - node.name=es02
      - cluster.name=es-docker-cluster
      - discovery.seed_hosts=es01,es03
      - cluster.initial_master_nodes=es01,es02,es03
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms128m -Xmx128m"
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - /Users/chengpeng/docker_volume/elasticsearch/data02:/usr/share/elasticsearch/data
    networks:
      - elastic
  es03:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.13.2
    container_name: es03
    environment:
      - node.name=es03
      - cluster.name=es-docker-cluster
      - discovery.seed_hosts=es01,es02
      - cluster.initial_master_nodes=es01,es02,es03
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms128m -Xmx128m"
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - /Users/chengpeng/docker_volume/elasticsearch/data03:/usr/share/elasticsearch/data
    networks:
      - elastic

volumes:
  data01:
    driver: local
  data02:
    driver: local
  data03:
    driver: local

networks:
  elastic:
    driver: bridge
```

`docker-compose up`

- es: http://localhost:9200/
- kinaba: http://localhost:5601/
- cerebro: http://localhost:9000/

## API操作
- 创建索引
    ```
    PUT books
    {
        "settings" : {
            "number_of_shards" : 4,
            "number_of_replicas" : 2
        },
        "mappings" : {
                "properties" : {
                    "name" : { "type" : "text" },
                    "price" : {"type" : "double"}
                }
        }
    }
    ```
- 新增数据
    ```
    PUT books/_doc/1
    {
        "name" : " Effective Java",
        "price" : 52.00,
        "message" : "本书介绍了在Java编程中78条极具实用价值的经验规则"
    }
    
    PUT books/_doc/2
    {
        "name" : "Effective C",
        "price" : 58.5,
        "message" : "An Introduction to Professional C Programming"
    }
    
    
    PUT books/_doc/3
    {
        "name" : "Thinking in Java",
        "price" : 66.5,
        "message" : "Thinking in Java should be read cover to cover by every Java programmer, then kept close at hand for frequent reference. "
    }
    ```

- 查询数据
    ```
    GET books/_search
    ```
- 搜索
    ```
    GET /books/_search?q=name:java

    GET /books/_search
    {
        "query" : {
            "term" : { "message": "java" }
        }
    }
    ```

## ES的数据结构及文件系统
![](https://cp-images.oss-cn-hangzhou.aliyuncs.com/16449122545087.jpg)


| 名称                                                                                                                         | 文件          | 描述                     |
|----------------------------------------------------------------------------------------------------------------------------|-------------|------------------------|
| [Segments File](https://lucene.apache.org/core/5_1_0/core/org/apache/lucene/index/SegmentInfos.html)                       | segments\_N | 存储关于提交点的信息commit point |
| [Segment Info](https://lucene.apache.org/core/5_1_0/core/org/apache/lucene/codecs/lucene50/Lucene50SegmentInfoFormat.html) | .si         | segment的元信息            |
| [Compound File](https://lucene.apache.org/core/5_1_0/core/org/apache/lucene/codecs/lucene50/Lucene50CompoundFormat.html)   | .cfs, .cfe  | 一些“虚拟”信息，少量的数据会存在这     |
| [Fields](https://lucene.apache.org/core/5_1_0/core/org/apache/lucene/codecs/lucene50/Lucene50FieldInfosFormat.html)        | .fnm        | 字段的信息                  |
| [Field Index](https://lucene.apache.org/core/5_1_0/core/org/apache/lucene/codecs/lucene50/Lucene50StoredFieldsFormat.html) | .fdx        | 字段索引的信息                |
| [Field Data](https://lucene.apache.org/core/5_1_0/core/org/apache/lucene/codecs/lucene50/Lucene50StoredFieldsFormat.html)  | .fdt        | 存储文档字段数据               |
| [Term Dictionary](https://lucene.apache.org/core/5_1_0/core/org/apache/lucene/codecs/lucene50/Lucene50PostingsFormat.html) | .tim        | 术语词典                   |
| [Term Index](https://lucene.apache.org/core/5_1_0/core/org/apache/lucene/codecs/lucene50/Lucene50PostingsFormat.html)      | .tip        | 术语索引                   |
| [Live Documents](https://lucene.apache.org/core/5_1_0/core/org/apache/lucene/codecs/lucene50/Lucene50LiveDocsFormat.html)  | .liv        | 文档存活的信息                |

## 存储文档的过程
![](https://cp-images.oss-cn-hangzhou.aliyuncs.com/16449123445377.jpg)
1. master节点接收请求
2. 通过对_id的hash（或者路由）来盘点存储到哪一个分片
3. 对应的分片本地存储数据
4. 再同步给副本分片

![](https://cp-images.oss-cn-hangzhou.aliyuncs.com/16449134143491.jpg)
1. 写入请求会将索引（Index）存放到内存区域，叫做 Index Buffer。此时的索引文件暂时是不能被ES搜索到的。
2. 默认情况下 ES 每秒执行一次 Refresh 操作，将 Index Buffer 中的 index 写入到 Filesystem 中，这个也是一片内存区域。
3. ES 每次 refresh 都会生成一个 Segment，定期对 Segment 进行合并（Merge）操作，也就是将多个小 Segment 合并成一个 Segment
4. 在合并完成后，会将新的 Segment 文件 Flush 写入磁盘。此时 ES 会创建一个 Commit Point 文件，该文件用来标识被 Flush 到磁盘上的 Segment。旧的 Segment 以及合并之前的小 Segment  会被从中移除

![](https://cp-images.oss-cn-hangzhou.aliyuncs.com/16449134603831.jpg)
1. 在 ES 处理用户请求时追加 Translog，追加的内容就是对ES的请求操作。此时会根据配置同步或者异步的方式将操作记录追加信息保存到磁盘中

## 搜索（重点）
### 常见索引结构
![](https://cp-images.oss-cn-hangzhou.aliyuncs.com/16449758531230.png)
### 倒排索引结构
#### 比如我们现在有一个这样的表
| id  | Text       | 其他字段。。。 |
|-----|------------|---------|
| 1   | Alan Alice |         |
| 2   |            |         |
| 3   | Alan       |         |
| 4   | Alan       |         |
| 5   | Alan Brad  |         |
| 6   | Alice      |         |
| 7   | Alan       |         |
| 8   | Alan       |         |
| 9   | Alan Alice |         |
| 10  | Brad       |         |
| 11  | Brad       |         |
| ... |            |         |

#### 我们现在需要查询包含`Alan`的所有数据
```sql
 select * from table where text like '%Alan%'
```
在Mysql中，这样的查询需要全表扫描，速度慢

#### 而在ES中
![](https://cp-images.oss-cn-hangzhou.aliyuncs.com/16449760553786.png)
- 通过分词，将这些关键词（`Term`）提前提取出来
- 这些关键词的集合就叫做`Term Dictionary`
- 每一个关键词，都有一个对应文档的列表`Posting List`
- 通过FST算法，构建一个Term的索引`Term dict index`

1. Terms Dictionary 存储 Term 的索引文件叫做 Terms Index，存储的格式是 .tip
2. Terms Dictionary 的文件存储格式为 .tim，存储了Term和对应的Postings List指针。
3. Postings List 被拆成三个文件存储：
  1. .doc后缀文件：记录 Postings 的 docId 信息和 Term 的词频
  2. .pay后缀文件：记录 Payload 信息和偏移量信息
  3. .pos后缀文件：记录位置信息

#### 通过ID查找之SkipList
![](https://cp-images.oss-cn-hangzhou.aliyuncs.com/16449777126594.png)
- 如果我们要查询id=12的文档
- 正常链表通过遍历，一个一个查询，时间复杂度是 O(n)
- 跳跃表是通过跳跃分层， 时间复杂度可以达到 O(log n)
  - 第一层， 0～15
  - 第二层， 8～15
  - 第三层， 12

### 搜索数据-BKDTree
对于索引一些多维数据，比如 坐标，使用的是BKD-Tree来索引
![](https://cp-images.oss-cn-hangzhou.aliyuncs.com/16449785200616.jpg)
- 如图上的点，我们是怎么给他们做2D平面划分的。
- 我们需要一个鉴别器`discriminator`
- 简单的鉴别器就是对层级取模
  - `discriminator = level % N`
- 会得到下面的树形结构
![](https://cp-images.oss-cn-hangzhou.aliyuncs.com/16449787811410.jpg)
- 多维的就是 x y z 一次类推
- 构建BKD树，就可以更好的根据多维数据来查询对应的文档了

## ES的优化

## ES相关问题

## 参考文章
- [官方文档（7.x)](https://www.elastic.co/guide/en/elasticsearch/reference/7.17/index.html)
- [Github](https://github.com/elastic/elasticsearch)
- [维基百科](https://zh.wikipedia.org/wiki/Elasticsearch)
- [Elasticsearch Distributed Consistency Principles Analysis (1) — Node](https://alibaba-cloud.medium.com/elasticsearch-distributed-consistency-principles-analysis-1-node-b512e2b839f8)
- [ES集群中各节点角色功能简介](https://juejin.cn/post/7038828692671299620)
- [Discovery in Elasticsearch](https://mincong.io/2020/08/22/discovery-in-elasticsearch/)
- [ElasticSearch-新老选主算法对比](https://yemilice.com/2021/06/16/elasticsearch-新老选主算法对比/)
- [Elasticsearch数据结构存储流程](https://blog.csdn.net/weixin_43902449/article/details/112449240)
- [elasticsearch 每个shard对应的文件含义](https://elasticsearch.cn/question/5173)
- [A Dive into the Elasticsearch Storage](https://www.elastic.co/cn/blog/found-dive-into-elasticsearch-storage#lucene-index-files)
- [Elasticsearch写入原理，一看便知！](https://mp.weixin.qq.com/s/wms9j22YcHfb8V9CBR65zQ)
- [ElasticSearch检索的核心-倒排索引解读](https://yemilice.com/2021/05/14/elasticsearch检索的核心-倒排索引解读/)
- [设计高可用的ElasicSearch索引](https://yemilice.com/2021/09/09/设计高可用的elasicsaerch索引/)
- [BKD 树，用于 Elasticsearch](https://medium.com/swlh/bkd-trees-used-in-elasticsearch-40e8afd2a1a4)