---
title: Java操作ElasticSearch
date: 2021-06-11 13:44:15
tags: 
  - java 
  - elasticsearch
  - springboot
  - jpa
  - docker
categories:
  - 技术
---
## 介绍

今天我们将使用Java来操作ElasticSearch。
我们将：

- 使用Docker搭建ES集群环境
- 使用SpringBoot搭建Java环境
- 在Java模型中定义ES的索引
- 使用两种方式来操作ES（JPA和ElasticSearchTemplate）

<!--more-->

## 搭建集群环境

今天我们继续使用docker来搭建环境。
`docker-compose.yaml`奉上。
想必大家都很清楚docker compose的用法了，这里就不再赘述。

```yaml
version: '2.2'
services:
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

因为这里是集群，除了`volumes`需要注意之外，还需要注意`network`需要配置一下。
启动完成之后，浏览器访问:
[http://localhost:9200](http://localhost:9200)
如果展示：
![](https://cp-images.oss-cn-hangzhou.aliyuncs.com/image.png)
表示ES集群启动成功。

## 搭建Java环境

我们创建一个SpringBoot项目。
在`pom.xml`中引入elasticsearch的starter，即可。
不过要注意的是，因为ElasticSearch版本迭代很快，我们的版本要对应的上。
我们先查看一下ElasticSearch的版本： 7.13.2
再看一下[Spring文档](https://docs.spring.io/spring-data/elasticsearch/docs/current/reference/html/)，找到对应的SpringData的版本。
![](https://cp-images.oss-cn-hangzhou.aliyuncs.com/J73lBg.png)
在pom中我们添加上引用：
```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-data-elasticsearch</artifactId>
  <version>2.6.7</version>
</dependency>
```
刷新下maven，即完成环境搭建。