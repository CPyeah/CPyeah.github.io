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

刷新下maven。 再继续在`application.properties`中配置:

```properties
spring.data.elasticsearch.cluster-name=es-docker-cluster
spring.data.elasticsearch.cluster-nodes=localhost:9200
```

即完成环境搭建。

## 在Java模型中定义ES的索引

好，接下来来到我们的重头戏，设计ES的索引。
在ES中，索引对应类似MySQL中的表，而Mapping，相当于MySQL中的表结构。
在一切开始之前，设计一个足够好的表结构尤为重要。可以直接影响到数据库的性能。

直接上结果：

```java
@Data
@Document(indexName = "product")
@Setting(shards = 3, replicas = 2)
public class Product {

	@Id
	@Field(type = FieldType.Text, index = false)
	private String id;

	@Field(type = FieldType.Keyword, ignoreAbove = 128)
	private String name;

	@Field(type = FieldType.Text, analyzer = "ik_smart", searchAnalyzer = "ik_smart")
	private String description;

	@Field(type = FieldType.Keyword)
	private String brandName;

	@Field(type = FieldType.Long)
	private Long stock;

	@Field(type = FieldType.Double)
	private BigDecimal price;

	@Field(type = FieldType.Date, format = {}, pattern = {"yyyy-MM-dd HH:mm:ss", "yyyy-MM-dd",
			"epoch_millis"})
	private LocalDateTime updateTime;

	@Field(type = FieldType.Nested)
	private List<Sku> skuList;
}

```

### 定义模型

我们首先定义了一个商品的Java模型`Product`，有各种Java类型（String、Long、BigDecimal、LocalDateTime、List）。
在类的上面添加`Document`注解，表示被Spring所管理。
`org.springframework.data.elasticsearch.annotations.Document`中还有一些其他的属性。大家可以参考官方文档。

### 定义Setting

定义好模型之后，我们再确定好索引的配置，`Setting`。
这里面主要配置，这个索引在ES中的分片数，和副本数。

```java
@Setting(shards = 3, replicas = 2)
```

当然，除了分片数和副本数，还有其他的配置，大家参考官方文档。

### 定义Mapping

接下来，就到了定义最重要的Mapping。

#### 确定数据结构

首先我们要知道，ES有着自己的数据结构，而且种类还很多。如图：
![](https://cp-images.oss-cn-hangzhou.aliyuncs.com/JtmYzS.png)
我们需要在每一个Java的数据结构中，对应上ES的数据结构。
比如`@Field(type = FieldType.Text)`表示它在ES中，是一个文本型的数据。
而文本型，是要被分词，生成倒排索引的。
这里提几点选择策略：

- 文本类型分两种，需要分词，选text，不需要分词，选keyword
- 如果你未来不确定你到底有多少数据，将类型设置为long是比较合适的
- keyword检索比较短的字符会更快，如果字符很大，那么会增加存储和检索成本，可以使用`ignoreAbove`属性

#### 确定是否需要被索引

默认情况下，索引的字段都是需要被索引的，但是为了节省空间，我们可以把一些字段设置成不建索引。
比如ID一类的参数，不想被索引：`@Field(type = FieldType.Text, index = false)`

#### 检索的方式

我们可以通过分词器，来定义我们的分词方式，比如
`@Field(type = FieldType.Text, analyzer = "ik_smart", searchAnalyzer = "ik_smart")`
这样，我们就可以使用中文的倒排索引啦。。。

#### 特殊的类型

如果我们需要使用时间类型，在Java中，我们使用`LocalDateTime`,对应ES中是`Date`类型。
但是需要注意的是，我们需要定义格式化的方式，不然大概率时分秒要丢失。
`@Field(type = FieldType.Date, format = {}, pattern = {"yyyy-MM-dd HH:mm:ss", "yyyy-MM-dd", "epoch_millis"})`

嵌套类型，使用`Nested`
`@Field(type = FieldType.Nested)`

## 使用两种方式来操作ES

![](https://cp-images.oss-cn-hangzhou.aliyuncs.com/MHdECp.png)

### 索引操作

首先我们来看看如何来管理ES的索引

#### JPA

如果使用JPA，我们只需保持默认配置，JPA就会替我们按照我们定义的Map来创建索引。
不需要我们操心。

我们可以使用[http://localhost:9200/product/_mapping](http://localhost:9200/product/_mapping)来查看对应索引信息。

#### ElasticsearchRestTemplate

template对应的Api如下：

```java
@Test
public void index() {
    // 拿到对应索引的操作引用
    IndexOperations indexOperations = elasticsearchRestTemplate.indexOps(Product.class);
    // 查看索引是否存在
    boolean exists = indexOperations.exists();
    if (exists) {
        // 删除索引（删除操作要慎重）
        indexOperations.delete();
    }
    // 创建索引
//		indexOperations.create(); //这个没有配置mapping
    indexOperations.createWithMapping();
  
    // 查看索引的Setting
    Settings settings = indexOperations.getSettings();
    // 查看索引的Mapping
    Map<String, Object> mapping = indexOperations.getMapping();
}
```

### 增删改查操作

#### JPA

使用JPA非常方便，只需要定义好`Repository`，就可以方便快捷的操作Document了。

```java
@Repository
public interface ProductRepository extends ElasticsearchRepository<Product, String> {

}
```

然后我们就可以这样使用：

```java
@SpringBootTest
public class CURDTest {

	@Autowired
	private ProductRepository repository;

	@Test
	public void create() {
		Product product = repository.save(Product.get());
		Assertions.assertNotNull(product);
	}

	@Test
	public void createMore() {
		List<Product> products = new ArrayList<>();
		for (int i = 0; i < 10; i++) {
			products.add(Product.get());
		}
		repository.saveAll(products);
	}

	@Test
	public void listAll() {
		Iterable<Product> all = repository.findAll();
		all.forEach(System.out::println);
	}

	@Test
	public void page() {
		Page<Product> page = repository.findAll(Pageable.ofSize(4).withPage(0));
		System.out.println(page);
	}

	@Test
	public void getById() {
		Page<Product> page = repository.findAll(Pageable.ofSize(1).withPage(0));
		Product product = page.getContent().get(0);
		Product product1 = repository.findById(product.getId()).orElse(null);
		Assertions.assertNotNull(product1);
		System.out.println(product1);
		Assertions.assertEquals(product.getId(), product1.getId());
	}

	@Test
	public void update() {
		Page<Product> page = repository.findAll(Pageable.ofSize(1).withPage(0));
		Product product = page.getContent().get(0);
		Product newProduct = Product.get();
		newProduct.setId(product.getId());
		Product save = repository.save(newProduct);
		Assertions.assertEquals(product.getId(), save.getId());
		Assertions.assertNotEquals(product.getName(), save.getName());
		System.out.println(product);
		System.out.println(save);
	}

	@Test
	public void delete() {
		Page<Product> page = repository.findAll(Pageable.ofSize(1).withPage(0));
		Product product = page.getContent().get(0);
		repository.delete(product);

		Product product1 = repository.findById(product.getId()).orElse(null);
		Assertions.assertNull(product1);
	}

}
```

JPA的封装可以让我们无差别的操作任何数据库。 yyds

#### ElasticsearchRestTemplate

使用ElasticsearchRestTemplate操作文档相对麻烦一些：

```java
@SpringBootTest
public class CURDTest2 {

	@Autowired
	private ElasticsearchRestTemplate elasticsearchRestTemplate;


	@Test
	public void create() {
		Product product = elasticsearchRestTemplate.save(Product.get());
		Assertions.assertNotNull(product);
	}

	@Test
	public void createMore() {
		List<Product> products = new ArrayList<>();
		for (int i = 0; i < 10; i++) {
			products.add(Product.get());
		}
		elasticsearchRestTemplate.save(products);
	}

	@Test
	public void listAll() {
		Query query = elasticsearchRestTemplate.matchAllQuery();
		SearchHits<Product> search = elasticsearchRestTemplate.search(query, Product.class);
	}

	@Test
	public void getById() {
		Product product = elasticsearchRestTemplate.get("ID", Product.class);
		Assertions.assertNotNull(product);
	}

	@Test
	public void update() {
		Product oldProduct = elasticsearchRestTemplate.get("ID", Product.class);
		Assertions.assertNotNull(oldProduct);

		Product newProduct = Product.get();
		Document document = newProduct.toDocument();
		UpdateQuery updateQuery = UpdateQuery.builder("ID").withDocument(document).build();
		UpdateResponse product = elasticsearchRestTemplate
				.update(updateQuery, IndexCoordinates.of("product"));

	}

	@Test
	public void delete() {
		String id = elasticsearchRestTemplate.delete("ID", Product.class);
	}

}
```

### 搜索

#### JPA

JPA的搜索很简单，就把当成一个普通的数据库搜索就行。
我们只需要在repository中定义

```java
@Repository
public interface ProductRepository extends ElasticsearchRepository<Product, String> {

	List<Product> findAllByNameContaining(String name);

}
```

我没就可以在所有名称中包含莫个字段了。
在调用这个方法的时候，可以在控制台看到，本质上是JPA帮我我们发了一个`_search`的POST请求。
其他更复杂的方法，可以参考JPA的文档。

#### ElasticsearchRestTemplate

使用ElasticsearchRestTemplate操作search，也很不错。
我们可以这样写：

```java
@Test
public void searchTest() {
    // 查询所有
    Query searchQuery = Query.findAll();
    IndexCoordinates indexCoordinates = IndexCoordinates.of("product");
    SearchHits<Product> result = elasticsearchRestTemplate
            .search(searchQuery, Product.class, indexCoordinates);
    System.out.println(result);

    // 按名称 搜索
    MatchQueryBuilder matchQueryBuilder = QueryBuilders.matchQuery("spuName", "473");
    searchQuery = new NativeSearchQueryBuilder()
            .withQuery(matchQueryBuilder)
            .build();
    result = elasticsearchRestTemplate
            .search(searchQuery, Product.class, indexCoordinates);
    System.out.println(result);

    // 按价格搜索
    Criteria criteria = new Criteria("price")
            .greaterThan(500.0)
            .lessThan(800.0);

    searchQuery = new CriteriaQuery(criteria);

    result = elasticsearchRestTemplate
            .search(searchQuery, Product.class, indexCoordinates);
    System.out.println(result);

}
```

这样写有一个好处，就是在返回结果数据的同时，还会返回命中率，这个有时候也是会用到的。

## 总结

至此，我们已经学会了如何把ES操作引入到我们到Java项目中，
还知道了如何设计并建立一个索引，
还有使用两种方法来操作ES。
希望大家都能够在工作当中用上哦！！
