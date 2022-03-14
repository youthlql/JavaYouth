---
title: ElasticSearch-进阶篇
tags:
  - ElasticSearch
  - ELK
  - 全文检索
categories:
  - ElasticSearch
  - 用法
keywords: ElasticSearch,全文检索
description: ElasticSearch-进阶篇，ElasticSearch的一些实战用法，集成SpringBoot。
cover: 'https://gitee.com/youthlql/randombg/raw/master/logo/es.jpg'
abbrlink: 50e81c79
date: 2020-02-08 18:06:23
---



# 搭建工程

ES提供多种不同的客户端： 

1、TransportClient 

ES提供的传统客户端，官方计划8.0版本删除此客户端。 

2、RestClient 

RestClient是官方推荐使用的，它包括两种：Java Low Level REST Client和 Java High Level REST Client。 

ES在6.0之后提供 Java High Level REST Client， 两种客户端官方更推荐使用 Java High Level REST Client，不过当 

前它还处于完善中，有些功能还没有。 



我们采用SpringBoot2.x与ElasticSearch集成

## Maven依赖

部分依赖

```maven
     <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
        <java.version>1.8</java.version>
        <elasticsearch.version>6.3.2</elasticsearch.version>
     </properties>


<!-- ES -->
	 <dependencies>
        <dependency>
            <groupId>org.elasticsearch.client</groupId>
            <artifactId>transport</artifactId>
            <version>${elasticsearch.version}</version>
        </dependency>
        <dependency>
            <groupId>org.elasticsearch.client</groupId>
            <artifactId>elasticsearch-rest-high-level-client</artifactId>
            <version>${elasticsearch.version}</version>
        </dependency>
        <dependency>
            <groupId>org.elasticsearch</groupId>
            <artifactId>elasticsearch</artifactId>
            <version>${elasticsearch.version}</version>
        </dependency>
      <dependencies>

```

## application.properties

```properties
#elasticsearch配置
anshe.elasticsearch.hostlist=${eshostlist:你的IP地址:9200}
```



## 配置类

```java
package com.anshe.common.config.es;

import com.anshe.web.service.ISearchService;
import org.apache.http.HttpHost;
import org.elasticsearch.client.RestClient;
import org.elasticsearch.client.RestHighLevelClient;
import org.elasticsearch.client.transport.TransportClient;
import org.elasticsearch.common.settings.Settings;
import org.elasticsearch.common.transport.TransportAddress;
import org.elasticsearch.transport.client.PreBuiltTransportClient;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.net.InetAddress;

/**
 * @author Administrator
 * @version 1.0
 **/
@Configuration
public class ElasticsearchConfig {
    private static final Logger logger = LoggerFactory.getLogger(ISearchService.class);

    @Value("${anshe.elasticsearch.hostlist}")
    private String hostlist;

    @Bean
    public RestHighLevelClient restHighLevelClient(){
        //解析hostlist配置信息
        String[] split = hostlist.split(",");
        //创建HttpHost数组，其中存放es主机和端口的配置信息
        HttpHost[] httpHostArray = new HttpHost[split.length];
        for(int i=0;i<split.length;i++){
            String item = split[i];
            httpHostArray[i] = new HttpHost(item.split(":")[0], Integer.parseInt(item.split(":")[1]), "http");
        }
        //创建RestHighLevelClient客户端
        return new RestHighLevelClient(RestClient.builder(httpHostArray));
    }

    //项目主要使用RestHighLevelClient，对于低级的客户端暂时不用
    @Bean
    public RestClient restClient(){
        //解析hostlist配置信息
        String[] split = hostlist.split(",");
        //创建HttpHost数组，其中存放es主机和端口的配置信息
        HttpHost[] httpHostArray = new HttpHost[split.length];
        for(int i=0;i<split.length;i++){
            String item = split[i];
            httpHostArray[i] = new HttpHost(item.split(":")[0], Integer.parseInt(item.split(":")[1]), "http");
        }
        return RestClient.builder(httpHostArray).build();
    }

    @Bean(name = "transportClient")
    public TransportClient transportClient() {
        logger.info("Elasticsearch初始化开始。。。。。");
        TransportClient transportClient = null;
        try {
            // 配置信息
            Settings esSetting = Settings.builder()
                    .put("cluster.name", "elasticsearch_anshe") //集群名字
                    .put("client.transport.sniff", true)//增加嗅探机制，找到ES集群
                    .build();
            //配置信息Settings自定义
            transportClient = new PreBuiltTransportClient(esSetting);
            TransportAddress transportAddress = new TransportAddress(InetAddress.getByName("你的IP地址"), 9300);
            transportClient.addTransportAddresses(transportAddress);
        } catch (Exception e) {
            logger.error("elasticsearch TransportClient create error!!", e);
        }
        return transportClient;
    }
}


```



## 主启动类

```java
package com.anshe;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import tk.mybatis.spring.annotation.MapperScan;

@SpringBootApplication
@MapperScan(basePackages = "com.anshe.web.mapper")
public class AnsheApplication {

    public static void main(String[] args) {
        System.setProperty("es.set.netty.runtime.available.processors", "false");
        SpringApplication.run(AnsheApplication.class, args);
    }

}
```





# 索引管理

## 创建索引库

### API

**创建索引：** 

put http://localhost:9200/索引名称 

```json
{
    "settings":{
        "index":{
            "number_of_shards":"1", # 分片数
            "number_of_replicas":"0" # 副本数
        }
    }
}
```



**创建映射：** 

发送：put http://localhost:9200/索引库名称/类型名称/_mapping 

创建类型为xc_course的映射，共包括三个字段：name、description、studymodel 等

http://localhost:9200/xc_course/doc/_mapping 

```json
{
	"properties": {
		"name": {
			"type": "text",
			"analyzer": "ik_max_word",
			"search_analyzer": "ik_smart"
		},
		"description": {
			"type": "text",
			"analyzer": "ik_max_word",
			"search_analyzer": "ik_smart"
		},
		"studymodel": {
			"type": "keyword"
		},
		"price": {
			"type": "float"
		},
		"timestamp": {
			"type": "date",
			"format": "yyyy‐MM‐dd HH:mm:ss||yyyy‐MM‐dd||epoch_millis"
		}
	}
}
```



### Java客户端

```java
@Autowired
RestHighLevelClient client;

@Autowired
RestClient restClient;

//创建索引库
@Test
public void testCreateIndex() throws IOException {
    //创建索引对象
    CreateIndexRequest createIndexRequest = new CreateIndexRequest("xc_course");
    //设置参数
    createIndexRequest.settings(Settings.builder().put("number_of_shards","1").put("number_of_replicas","0"));
    //指定映射
    createIndexRequest.mapping("doc"," {\n" +
            " \t\"properties\": {\n" +
            "            \"studymodel\":{\n" +
            "             \"type\":\"keyword\"\n" +
            "           },\n" +
            "            \"name\":{\n" +
            "             \"type\":\"keyword\"\n" +
            "           },\n" +
            "           \"description\": {\n" +
            "              \"type\": \"text\",\n" +
            "              \"analyzer\":\"ik_max_word\",\n" +
            "              \"search_analyzer\":\"ik_smart\"\n" +
            "           },\n" +
            "           \"pic\":{\n" +
            "             \"type\":\"text\",\n" +
            "             \"index\":false\n" +
            "           }\n" +
            " \t}\n" +
            "}", XContentType.JSON);
    //操作索引的客户端
    IndicesClient indices = client.indices();
    //执行创建索引库
    CreateIndexResponse createIndexResponse = indices.create(createIndexRequest);
    //得到响应
    boolean acknowledged = createIndexResponse.isAcknowledged();
    System.out.println(acknowledged);

}
```



## 删除索引库

### API

```json
DELETE http://['你自己的Ip加Port']/test
{
    "acknowledged": true
}
```



### Java客户端

```java
//删除索引库
@Test
public void testDeleteIndex() throws IOException {
    //删除索引对象
    DeleteIndexRequest deleteIndexRequest = new DeleteIndexRequest("xc_course");
    //操作索引的客户端
    IndicesClient indices = client.indices();
    //执行删除索引
    DeleteIndexResponse delete = indices.delete(deleteIndexRequest);
    //得到响应
    boolean acknowledged = delete.isAcknowledged();
    System.out.println(acknowledged);

}
```



## 添加文档



### API

格式如下： PUT /{index}/{type}/{id} { "fifield": "value", ... } 

如果不指定id，ES会自动生成。 

一个例子： 

put http://localhost:9200/xc_course/doc/3 

```Json
{
	"name": "spring cloud实战",
	"description": "本课程主要从四个章节进行讲解： 1.微服务架构入门 2.spring cloud 基础入门 3.实战Spring Boot 4.注册中心eureka。",
	"studymodel": "201001",
	"price": 5.6
}
```





### Java客户端

```java
//添加文档
@Test
public void testAddDoc() throws IOException {
    //文档内容
    //准备json数据
    Map<String, Object> jsonMap = new HashMap<>();
    jsonMap.put("name", "spring cloud实战");
    jsonMap.put("description", "本课程主要从四个章节进行讲解： 1.微服务架构入门 2.spring cloud 基础入门 3.实战Spring Boot 4.注册中心eureka。");
    jsonMap.put("studymodel", "201001");
    SimpleDateFormat dateFormat =new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
    jsonMap.put("timestamp", dateFormat.format(new Date()));
    jsonMap.put("price", 5.6f);

    //创建索引创建对象
    IndexRequest indexRequest = new IndexRequest("xc_course","doc");
    //文档内容
    indexRequest.source(jsonMap);
    //通过client进行http的请求
    IndexResponse indexResponse = client.index(indexRequest);
    DocWriteResponse.Result result = indexResponse.getResult();
    System.out.println(result);

}
```



## 查询文档

### API

格式如下： GET /{index}/{type}/{id} 

### Java客户端

```java
//查询文档
@Test
public void testGetDoc() throws IOException {
    //查询请求对象
    GetRequest getRequest = new GetRequest("xc_course","doc","0fOCF2sBEYTsNRZ43I8b");
    GetResponse getResponse = client.get(getRequest);
    //得到文档的内容
    Map<String, Object> sourceAsMap = getResponse.getSourceAsMap();
    System.out.println(sourceAsMap);
}
```



## 更新文档

### API

ES更新文档的顺序是：先检索到文档、将原来的文档标记为删除、创建新文档、删除旧文档，创建新文档就会重建 

索引。 

通过请求Url有两种方法： 

**1、完全替换** 

Post：http://localhost:9200/xc_test/doc/3 

```json
{
	"name": "spring cloud实战",
	"description": "本课程主要从四个章节进行讲解： 1.微服务架构入门 2.spring cloud 基础入门 3.实战SpringBoot 4.注册中心eureka。",
	"studymodel": "201001",
	"price": 5.6
}
```

**2、局部更新** 

下边的例子是只更新price字段。 

post: http://localhost:9200/xc_test/doc/3/_update 

```json
{
	"doc": {
		"price": 66.6
	}
}
```



### Java客户端

使用 Client Api更新文档的方法同上边第二种局部更新方法。 

可以指定文档的部分字段也可以指定完整的文档内容。

```
//更新文档
@Test public void updateDoc() throws IOException {
	UpdateRequest updateRequest = new UpdateRequest("xc_course", "doc", "4028e581617f945f01617f9dabc40000");
	Map<String, String> map = new HashMap<>();
	map.put("name", "spring cloud实战");
	updateRequest.doc(map);
	UpdateResponse update = client.update(updateRequest);
	RestStatus status = update.status();
	System.out.println(status);
}
```





## 删除文档

### API

1、根据id删除，格式如下： 

DELETE /{index}/{type}/{id} 

2、搜索匹配删除，将搜索出来的记录删除，格式如下： 

POST /{index}/{type}/_delete_by_query 

下边是搜索条件例子： 

```json
{
	"query": {
		"term": {
			"studymodel": "201001"
		}
	}
}
```

上边例子的搜索匹配删除会将studymodel为201001的记录全部删除

### Java客户端

```java
//根据id删除文档
@Test 
public void testDelDoc() throws IOException { 
	//删除文档id
	String id = "eqP_amQBKsGOdwJ4fHiC"; 
	//删除索引请求对象
	DeleteRequest deleteRequest = new DeleteRequest("xc_course","doc",id); 
	//响应对象
	DeleteResponse deleteResponse = client.delete(deleteRequest);
	//获取响应结果
	DocWriteResponse.Result result = deleteResponse.getResult();
	System.out.println(result);
}
```

搜索匹配删除还没有具体的api，可以采用先搜索出文档id，根据文档id删除。



# -----下面是DSL搜索的内容-----

# DSL搜索环境准备

## 创建映射

创建xc_course索引库。 

创建如下映射 

post：http://localhost:9200/xc_course/doc/_mapping

```json
{
	"properties": {
		"description": {
			"type": "text",
			"analyzer": "ik_max_word",
			"search_analyzer": "ik_smart"
		},
		"name": {
			"type": "text",
			"analyzer": "ik_max_word",
			"search_analyzer": "ik_smart"
		},
		"pic": {
			"type": "text",
			"index": false
		},
		"price": {
			"type": "float"
		},
		"studymodel": {
			"type": "keyword"
		},
		"timestamp": {
			"type": "date",
			"format": "yyyy‐MM‐dd HH:mm:ss||yyyy‐MM‐dd||epoch_millis"
		}
	}
}
```



 

## 插入原始数据

向xc_course/doc中插入以下数据：

```json
http://localhost:9200/xc_course/doc/1
{
	"name": "Bootstrap开发",
	"description": "Bootstrap是由Twitter推出的一个前台页面开发框架，是一个非常流行的开发框架，此框架集成了 多种页面效果。此开发框架包含了大量的CSS、JS程序代码，可以帮助开发者（尤其是不擅长页面开发的程序人员）轻松 的实现一个不受浏览器限制的精美界面效果。",
	"studymodel": "201002",
	"price": 38.6,
	"timestamp": "2018‐04‐25 19:11:35",
	"pic": "group1/M00/00/00/wKhlQFs6RCeAY0pHAAJx5ZjNDEM428.jpg"
}


http://localhost:9200/xc_course/doc/2
{
	"name": "java编程基础",
	"description": "java语言是世界第一编程语言，在软件开发领域使用人数最多。",
	"studymodel": "201001",
	"price": 68.6,
	"timestamp": "2018‐03‐25 19:11:35",
	"pic": "group1/M00/00/00/wKhlQFs6RCeAY0pHAAJx5ZjNDEM428.jpg"
}


http://localhost:9200/xc_course/doc/3 
{
	"name": "spring开发基础",
	"description": "spring 在java领域非常流行，java程序员都在用。",
	"studymodel": "201001",
	"price": 88.6,
	"timestamp": "2018‐02‐24 19:11:35",
	"pic": "group1/M00/00/00/wKhlQFs6RCeAY0pHAAJx5ZjNDEM428.jpg"
}

```





>  DSL(Domain Specifific Language)是ES提出的基于json的搜索方式，在搜索时传入特定的json格式的数据来完成不 同的搜索需求。 DSL比URI搜索方式功能强大，在项目中建议使用DSL方式来完成搜索。





# 查询所有文档

## API

查询所有索引库的文档。 

发送：post http://localhost:9200/_search 

查询指定索引库指定类型下的文档。（通过使用此方法） 

发送：post http://localhost:9200/xc_course/doc/_search

```json
{
	"query": {
		"match_all": {}
	},
	"_source": [
		"name",
		"studymodel"
	]
}
```

_source：source源过虑设置，指定结果中所包括的字段有哪些。 

**结果说明：** 

took：本次操作花费的时间，单位为毫秒。 

timed_out：请求是否超时 

_shards：说明本次操作共搜索了哪些分片 

hits：搜索命中的记录 

hits.total ： 符合条件的文档总数 hits.hits ：匹配度较高的前N个文档 

hits.max_score：文档匹配得分，这里为最高分 

_score：每个文档都有一个匹配度得分，按照降序排列。 

_source：显示了文档的原始内容。 

## Java客户端

```java
	 @Autowired
    RestHighLevelClient client;

    @Autowired
    RestClient restClient;

//搜索全部记录
    @Test
    public void testSearchAll() throws IOException, ParseException {
        //搜索请求对象
        SearchRequest searchRequest = new SearchRequest("xc_course");
        //指定类型
        searchRequest.types("doc");
        //搜索源构建对象
        SearchSourceBuilder searchSourceBuilder = new SearchSourceBuilder();
        //搜索方式
        //matchAllQuery搜索全部
        searchSourceBuilder.query(QueryBuilders.matchAllQuery());
        //设置源字段过虑,第一个参数结果集包括哪些字段，第二个参数表示结果集不包括哪些字段
        searchSourceBuilder.fetchSource(new String[]{"name","studymodel","price","timestamp"},new String[]{});
        //向搜索请求对象中设置搜索源
        searchRequest.source(searchSourceBuilder);
        //执行搜索,向ES发起http请求
        SearchResponse searchResponse = client.search(searchRequest);
        //搜索结果
        SearchHits hits = searchResponse.getHits();
        //匹配到的总记录数
        long totalHits = hits.getTotalHits();
        //得到匹配度高的文档
        SearchHit[] searchHits = hits.getHits();
        //日期格式化对象
//        SimpleDateFormat dateFormat = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
        SimpleDateFormat dateFormat = new SimpleDateFormat("yyyy-MM-dd'T'HH:mm:ss.SSS'Z'");
        for(SearchHit hit:searchHits){
            //文档的主键
            String id = hit.getId();
            //源文档内容
            Map<String, Object> sourceAsMap = hit.getSourceAsMap();
            String name = (String) sourceAsMap.get("name");
            //由于前边设置了源文档字段过虑，这时description是取不到的
            String description = (String) sourceAsMap.get("description");
            //学习模式
            String studymodel = (String) sourceAsMap.get("studymodel");
            //价格
            Double price = (Double) sourceAsMap.get("price");
            //日期
            Date timestamp = dateFormat.parse((String) sourceAsMap.get("timestamp"));
            System.out.println(name);
            System.out.println(studymodel);
            System.out.println(description);
        }

    }
```



# 分页查询

## API

ES支持分页查询，传入两个参数：from和size。 

form：表示起始文档的下标，从0开始。 

size：查询的文档数量。 

发送：post http://localhost:9200/xc_course/doc/_search 

```json
{
	"from": 0,
	"size": 1,
	"query": {
		"match_all": {}
	},
	"_source": [
		"name",
		"studymodel"
	]
}
```



## Java客户端

```java
//分页查询
@Test
public void testSearchPage() throws IOException, ParseException {
    //搜索请求对象
    SearchRequest searchRequest = new SearchRequest("xc_course");
    //指定类型
    searchRequest.types("doc");
    //搜索源构建对象
    SearchSourceBuilder searchSourceBuilder = new SearchSourceBuilder();
    //设置分页参数
    //页码
    int page = 1;
    //每页记录数
    int size = 1;
    //计算出记录起始下标
    int from  = (page-1)*size;
    searchSourceBuilder.from(from);//起始记录下标，从0开始
    searchSourceBuilder.size(size);//每页显示的记录数
    //搜索方式
    //matchAllQuery搜索全部
    searchSourceBuilder.query(QueryBuilders.matchAllQuery());
    //设置源字段过虑,第一个参数结果集包括哪些字段，第二个参数表示结果集不包括哪些字段
    searchSourceBuilder.fetchSource(new String[]{"name","studymodel","price","timestamp"},new String[]{});
    //向搜索请求对象中设置搜索源
    searchRequest.source(searchSourceBuilder);
    //执行搜索,向ES发起http请求
    SearchResponse searchResponse = client.search(searchRequest);
    //搜索结果
    SearchHits hits = searchResponse.getHits();
    //匹配到的总记录数
    long totalHits = hits.getTotalHits();
    //得到匹配度高的文档
    SearchHit[] searchHits = hits.getHits();
    //日期格式化对象
    SimpleDateFormat dateFormat = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
    for(SearchHit hit:searchHits){
        //文档的主键
        String id = hit.getId();
        //源文档内容
        Map<String, Object> sourceAsMap = hit.getSourceAsMap();
        String name = (String) sourceAsMap.get("name");
        //由于前边设置了源文档字段过虑，这时description是取不到的
        String description = (String) sourceAsMap.get("description");
        //学习模式
        String studymodel = (String) sourceAsMap.get("studymodel");
        //价格
        Double price = (Double) sourceAsMap.get("price");
        //日期
        Date timestamp = dateFormat.parse((String) sourceAsMap.get("timestamp"));
        System.out.println(name);
        System.out.println(studymodel);
        System.out.println(description);
    }

}
```



# Term Query

## API

Term Query为精确查询，在搜索时会整体匹配关键字，不再将关键字分词。 

发送：post http://localhost:9200/xc_course/doc/_search 

```java
{
	"query": {
		"term": {
			"name": "spring"
		}
	},
	"_source": [
		"name",
		"studymodel"
	]
}
```

上边的搜索会查询name包括“spring”这个词的文档。



## Java客户端

```java
//TermQuery
@Test
public void testTermQuery() throws IOException, ParseException {
    //搜索请求对象
    SearchRequest searchRequest = new SearchRequest("xc_course");
    //指定类型
    searchRequest.types("doc");
    //搜索源构建对象
    SearchSourceBuilder searchSourceBuilder = new SearchSourceBuilder();
    //设置分页参数
    //页码
    int page = 1;
    //每页记录数
    int size = 1;
    //计算出记录起始下标
    int from  = (page-1)*size;
    searchSourceBuilder.from(from);//起始记录下标，从0开始
    searchSourceBuilder.size(size);//每页显示的记录数
    //搜索方式
    //termQuery
    searchSourceBuilder.query(QueryBuilders.termQuery("name","spring"));
    //设置源字段过虑,第一个参数结果集包括哪些字段，第二个参数表示结果集不包括哪些字段
    searchSourceBuilder.fetchSource(new String[]{"name","studymodel","price","timestamp"},new String[]{});
    //向搜索请求对象中设置搜索源
    searchRequest.source(searchSourceBuilder);
    //执行搜索,向ES发起http请求
    SearchResponse searchResponse = client.search(searchRequest);
    //搜索结果
    SearchHits hits = searchResponse.getHits();
    //匹配到的总记录数
    long totalHits = hits.getTotalHits();
    //得到匹配度高的文档
    SearchHit[] searchHits = hits.getHits();
    //日期格式化对象
    SimpleDateFormat dateFormat = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
    for(SearchHit hit:searchHits){
        //文档的主键
        String id = hit.getId();
        //源文档内容
        Map<String, Object> sourceAsMap = hit.getSourceAsMap();
        String name = (String) sourceAsMap.get("name");
        //由于前边设置了源文档字段过虑，这时description是取不到的
        String description = (String) sourceAsMap.get("description");
        //学习模式
        String studymodel = (String) sourceAsMap.get("studymodel");
        //价格
        Double price = (Double) sourceAsMap.get("price");
        //日期
        Date timestamp = dateFormat.parse((String) sourceAsMap.get("timestamp"));
        System.out.println(name);
        System.out.println(studymodel);
        System.out.println(description);
    }

}
```



# 根据id精确匹配

## API

ES提供根据多个id值匹配的方法： 

测试：

post： http://127.0.0.1:9200/xc_course/doc/_search 

```json
{
	"query": {
		"ids": {
			"type": "doc",
			"values": [
				"3",
				"4",
				"100"
			]
		}
	}
}
```



## Java客户端

```java
//根据id查询
@Test
public void testTermQueryByIds() throws IOException, ParseException {
    //搜索请求对象
    SearchRequest searchRequest = new SearchRequest("xc_course");
    //指定类型
    searchRequest.types("doc");
    //搜索源构建对象
    SearchSourceBuilder searchSourceBuilder = new SearchSourceBuilder();
    //搜索方式
    //根据id查询
    //定义id
    String[] ids = new String[]{"1","2"};
    searchSourceBuilder.query(QueryBuilders.termsQuery("_id",ids));
    //设置源字段过虑,第一个参数结果集包括哪些字段，第二个参数表示结果集不包括哪些字段
    searchSourceBuilder.fetchSource(new String[]{"name","studymodel","price","timestamp"},new String[]{});
    //向搜索请求对象中设置搜索源
    searchRequest.source(searchSourceBuilder);
    //执行搜索,向ES发起http请求
    SearchResponse searchResponse = client.search(searchRequest);
    //搜索结果
    SearchHits hits = searchResponse.getHits();
    //匹配到的总记录数
    long totalHits = hits.getTotalHits();
    //得到匹配度高的文档
    SearchHit[] searchHits = hits.getHits();
    //日期格式化对象
    SimpleDateFormat dateFormat = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
    for(SearchHit hit:searchHits){
        //文档的主键
        String id = hit.getId();
        //源文档内容
        Map<String, Object> sourceAsMap = hit.getSourceAsMap();
        String name = (String) sourceAsMap.get("name");
        //由于前边设置了源文档字段过虑，这时description是取不到的
        String description = (String) sourceAsMap.get("description");
        //学习模式
        String studymodel = (String) sourceAsMap.get("studymodel");
        //价格
        Double price = (Double) sourceAsMap.get("price");
        //日期
        Date timestamp = dateFormat.parse((String) sourceAsMap.get("timestamp"));
        System.out.println(name);
        System.out.println(studymodel);
        System.out.println(description);
    }

}
```



 

# match Query

## API

match Query即全文检索，它的搜索方式是先将搜索字符串分词，再使用各各词条从索引中搜索。 

match query与Term query区别是match query在搜索前先将搜索关键字分词，再拿各各词语去索引中搜索。 

发送：post http://localhost:9200/xc_course/doc/_search 

```json
{
	"query": {
		"match": {
			"description": {
				"query": "spring开发",
				"operator": "or"
			}
		}
	}
}
```

query：搜索的关键字，对于英文关键字如果有多个单词则中间要用半角逗号分隔，而对于中文关键字中间可以用 

逗号分隔也可以不用。 

operator：or 表示 只要有一个词在文档中出现则就符合条件，and表示每个词都在文档中出现则才符合条件。 

上边的搜索的执行过程是： 

1、将“spring开发”分词，分为spring、开发两个词 

2、再使用spring和开发两个词去匹配索引中搜索。 

3、由于设置了operator为or，只要有一个词匹配成功则就返回该文档。 



## Java客户端

```java
//MatchQuery
@Test
public void testMatchQuery() throws IOException, ParseException {
    //搜索请求对象
    SearchRequest searchRequest = new SearchRequest("xc_course");
    //指定类型
    searchRequest.types("doc");
    //搜索源构建对象
    SearchSourceBuilder searchSourceBuilder = new SearchSourceBuilder();

    //搜索方式
    //MatchQuery
    searchSourceBuilder.query(QueryBuilders.matchQuery("description","spring开发框架")
            .minimumShouldMatch("80%"));
    //设置源字段过虑,第一个参数结果集包括哪些字段，第二个参数表示结果集不包括哪些字段
    searchSourceBuilder.fetchSource(new String[]{"name","studymodel","price","timestamp"},new String[]{});
    //向搜索请求对象中设置搜索源
    searchRequest.source(searchSourceBuilder);
    //执行搜索,向ES发起http请求
    SearchResponse searchResponse = client.search(searchRequest);
    //搜索结果
    SearchHits hits = searchResponse.getHits();
    //匹配到的总记录数
    long totalHits = hits.getTotalHits();
    //得到匹配度高的文档
    SearchHit[] searchHits = hits.getHits();
    //日期格式化对象
    SimpleDateFormat dateFormat = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
    for(SearchHit hit:searchHits){
        //文档的主键
        String id = hit.getId();
        //源文档内容
        Map<String, Object> sourceAsMap = hit.getSourceAsMap();
        String name = (String) sourceAsMap.get("name");
        //由于前边设置了源文档字段过虑，这时description是取不到的
        String description = (String) sourceAsMap.get("description");
        //学习模式
        String studymodel = (String) sourceAsMap.get("studymodel");
        //价格
        Double price = (Double) sourceAsMap.get("price");
        //日期
        Date timestamp = dateFormat.parse((String) sourceAsMap.get("timestamp"));
        System.out.println(name);
        System.out.println(studymodel);
        System.out.println(description);
    }

}
```



# multi Query

## API

**1、基本使用**

上边学习的termQuery和matchQuery一次只能匹配一个Field，本节学习multiQuery，一次可以匹配多个字段。 

单项匹配是在一个fifield中去匹配，多项匹配是拿关键字去多个Field中匹配。 

例子： 

发送：post http://localhost:9200/xc_course/doc/_search 

拿关键字 “spring css”去匹配name 和description字段。

```json
{
	"query": {
		"multi_match": {
			"query": "spring css",
			"minimum_should_match": "50%",
			"fields": [
				"name",
				"description"
			]
		}
	}
}
```



**2、提升boost** 

匹配多个字段时可以提升字段的boost（权重）来提高得分 

例子： 

提升boost之前，执行下边的查询： 

```json
{
	"query": {
		"multi_match": {
			"query": "spring框架",
			"minimum_should_match": "50%",
			"fields": [
				"name",
				"description"
			]
		}
	}
}
```

通过查询发现Bootstrap排在前边。 

提升boost，通常关键字匹配上name的权重要比匹配上description的权重高，这里可以对name的权重提升

```json
{
	"query": {
		"multi_match": {
			"query": "spring框架",
			"minimum_should_match": "50%",
			"fields": [
				"name^10",
				"description"
			]
		}
	}
}
```

“name^10” 表示权重提升10倍，执行上边的查询，发现name中包括spring关键字的文档排在前边。



## Java客户端

```java
//MultiMatchQuery
@Test
public void testMultiMatchQuery() throws IOException, ParseException {
    //搜索请求对象
    SearchRequest searchRequest = new SearchRequest("xc_course");
    //指定类型
    searchRequest.types("doc");
    //搜索源构建对象
    SearchSourceBuilder searchSourceBuilder = new SearchSourceBuilder();

    //搜索方式
    //MultiMatchQuery
    searchSourceBuilder.query(QueryBuilders.multiMatchQuery("spring css","name","description")
            .minimumShouldMatch("50%")
            .field("name",10));
    //设置源字段过虑,第一个参数结果集包括哪些字段，第二个参数表示结果集不包括哪些字段
    searchSourceBuilder.fetchSource(new String[]{"name","studymodel","price","timestamp"},new String[]{});
    //向搜索请求对象中设置搜索源
    searchRequest.source(searchSourceBuilder);
    //执行搜索,向ES发起http请求
    SearchResponse searchResponse = client.search(searchRequest);
    //搜索结果
    SearchHits hits = searchResponse.getHits();
    //匹配到的总记录数
    long totalHits = hits.getTotalHits();
    //得到匹配度高的文档
    SearchHit[] searchHits = hits.getHits();
    //日期格式化对象
    SimpleDateFormat dateFormat = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
    for(SearchHit hit:searchHits){
        //文档的主键
        String id = hit.getId();
        //源文档内容
        Map<String, Object> sourceAsMap = hit.getSourceAsMap();
        String name = (String) sourceAsMap.get("name");
        //由于前边设置了源文档字段过虑，这时description是取不到的
        String description = (String) sourceAsMap.get("description");
        //学习模式
        String studymodel = (String) sourceAsMap.get("studymodel");
        //价格
        Double price = (Double) sourceAsMap.get("price");
        //日期
        Date timestamp = dateFormat.parse((String) sourceAsMap.get("timestamp"));
        System.out.println(name);
        System.out.println(studymodel);
        System.out.println(description);
    }

}
```



# 布尔查询

## API

布尔查询对应于Lucene的BooleanQuery查询，实现将多个查询组合起来。 

- 三个参数： 

  - must：文档必须匹配must所包括的查询条件，相当于 “AND” 

  - should：文档应该匹配should所包括的查询条件其中的一个或多个，相当于 "OR" 

  - must_not：文档不能匹配must_not所包括的该查询条件，相当于“NOT”

分别使用must、should、must_not测试下边的查询： 

发送：POST http://localhost:9200/xc_course/doc/_search 

```json
{
	"_source": [
		"name",
		"studymodel",
		"description"
	],
	"from": 0,
	"size": 1,
	"query": {
		"bool": {
			"must": [
				{
					"multi_match": {
						"query": "spring框架",
						"minimum_should_match": "50%",
						"fields": [
							"name^10",
							"description"
						]
					}
				},
				{
					"term": {
						"studymodel": "201001"
					}
				}
			]
		}
	}
}
```

must：表示必须，多个查询条件必须都满足。（通常使用must） 

should：表示或者，多个查询条件只要有一个满足即可。 

must_not：表示非。



## Java客户端

```java
//BoolQuery其实是一个过滤搜索
@Test
public void testBoolQuery() throws IOException, ParseException {
    //搜索请求对象
    SearchRequest searchRequest = new SearchRequest("xc_course");
    //指定类型
    searchRequest.types("doc");
    //搜索源构建对象
    SearchSourceBuilder searchSourceBuilder = new SearchSourceBuilder();

    //boolQuery搜索方式
    //先定义一个MultiMatchQuery
    MultiMatchQueryBuilder multiMatchQueryBuilder = QueryBuilders.multiMatchQuery("spring css", "name", "description")
            .minimumShouldMatch("50%")
            .field("name", 10);
    //再定义一个termQuery
    TermQueryBuilder termQueryBuilder = QueryBuilders.termQuery("studymodel", "201001");

    //定义一个boolQuery
    BoolQueryBuilder boolQueryBuilder = QueryBuilders.boolQuery();
    boolQueryBuilder.must(multiMatchQueryBuilder);
    boolQueryBuilder.must(termQueryBuilder);

    searchSourceBuilder.query(boolQueryBuilder);
    //设置源字段过虑,第一个参数结果集包括哪些字段，第二个参数表示结果集不包括哪些字段
    searchSourceBuilder.fetchSource(new String[]{"name","studymodel","price","timestamp"},new String[]{});
    //向搜索请求对象中设置搜索源
    searchRequest.source(searchSourceBuilder);
    //执行搜索,向ES发起http请求
    SearchResponse searchResponse = client.search(searchRequest);
    //搜索结果
    SearchHits hits = searchResponse.getHits();
    //匹配到的总记录数
    long totalHits = hits.getTotalHits();
    //得到匹配度高的文档
    SearchHit[] searchHits = hits.getHits();
    //日期格式化对象
    SimpleDateFormat dateFormat = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
    for(SearchHit hit:searchHits){
        //文档的主键
        String id = hit.getId();
        //源文档内容
        Map<String, Object> sourceAsMap = hit.getSourceAsMap();
        String name = (String) sourceAsMap.get("name");
        //由于前边设置了源文档字段过虑，这时description是取不到的
        String description = (String) sourceAsMap.get("description");
        //学习模式
        String studymodel = (String) sourceAsMap.get("studymodel");
        //价格
        Double price = (Double) sourceAsMap.get("price");
        //日期
        Date timestamp = dateFormat.parse((String) sourceAsMap.get("timestamp"));
        System.out.println(name);
        System.out.println(studymodel);
        System.out.println(description);
    }

}
```





# 过虑器

## API

​	过虑是针对搜索的结果进行过虑，过虑器主要判断的是文档是否匹配，不去计算和判断文档的匹配度得分，所以过 虑器性能比查询要高，且方便缓存，推荐尽量使用过虑器去实现查询或者过虑器和查询共同使用。 过虑器在布尔查询中使用，下边是在搜索结果的基础上进行过虑： 

```json
{
	"_source": [
		"name",
		"studymodel",
		"description",
		"price"
	],
	"query": {
		"bool": {
			"must": [
				{
					"multi_match": {
						"query": "spring框架",
						"minimum_should_match": "50%",
						"fields": [
							"name^10",
							"description"
						]
					}
				}
			],
			"filter": [
				{
					"term": {
						"studymodel": "201001"
					}
				},
				{
					"range": {
						"price": {
							"gte": 60,
							"lte": 100
						}
					}
				}
			]
		}
	}
}
```

range：范围过虑，保留大于等于60 并且小于等于100的记录。

term：项匹配过虑，保留studymodel等于"201001"的记录。 

注意：range和term一次只能对一个Field设置范围过虑。



## Java客户端

```java
//filter
@Test
public void testFilter() throws IOException, ParseException {
    //搜索请求对象
    SearchRequest searchRequest = new SearchRequest("xc_course");
    //指定类型
    searchRequest.types("doc");
    //搜索源构建对象
    SearchSourceBuilder searchSourceBuilder = new SearchSourceBuilder();

    //boolQuery搜索方式
    //先定义一个MultiMatchQuery
    MultiMatchQueryBuilder multiMatchQueryBuilder = QueryBuilders.multiMatchQuery("spring css", "name", "description")
            .minimumShouldMatch("50%")
            .field("name", 10);

    //定义一个boolQuery
    BoolQueryBuilder boolQueryBuilder = QueryBuilders.boolQuery();
    boolQueryBuilder.must(multiMatchQueryBuilder);
    //定义过虑器
    boolQueryBuilder.filter(QueryBuilders.termQuery("studymodel","201001"));
    boolQueryBuilder.filter(QueryBuilders.rangeQuery("price").gte(90).lte(100));

    searchSourceBuilder.query(boolQueryBuilder);
    //设置源字段过虑,第一个参数结果集包括哪些字段，第二个参数表示结果集不包括哪些字段
    searchSourceBuilder.fetchSource(new String[]{"name","studymodel","price","timestamp"},new String[]{});
    //向搜索请求对象中设置搜索源
    searchRequest.source(searchSourceBuilder);
    //执行搜索,向ES发起http请求
    SearchResponse searchResponse = client.search(searchRequest);
    //搜索结果
    SearchHits hits = searchResponse.getHits();
    //匹配到的总记录数
    long totalHits = hits.getTotalHits();
    //得到匹配度高的文档
    SearchHit[] searchHits = hits.getHits();
    //日期格式化对象
    SimpleDateFormat dateFormat = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
    for(SearchHit hit:searchHits){
        //文档的主键
        String id = hit.getId();
        //源文档内容
        Map<String, Object> sourceAsMap = hit.getSourceAsMap();
        String name = (String) sourceAsMap.get("name");
        //由于前边设置了源文档字段过虑，这时description是取不到的
        String description = (String) sourceAsMap.get("description");
        //学习模式
        String studymodel = (String) sourceAsMap.get("studymodel");
        //价格
        Double price = (Double) sourceAsMap.get("price");
        //日期
        Date timestamp = dateFormat.parse((String) sourceAsMap.get("timestamp"));
        System.out.println(name);
        System.out.println(studymodel);
        System.out.println(description);
    }

}
```





 

# 排序

## API

可以在字段上添加一个或多个排序，支持在keyword、date、flfloat等类型上添加，text类型的字段上不允许添加排 

序。

发送 POST http://localhost:9200/xc_course/doc/_search 

过虑0--10元价格范围的文档，并且对结果进行排序，先按studymodel降序，再按价格升序 

```json
{
	"_source": [
		"name",
		"studymodel",
		"description",
		"price"
	],
	"query": {
		"bool": {
			"filter": [
				{
					"range": {
						"price": {
							"gte": 0,
							"lte": 100
						}
					}
				}
			]
		}
	},
	"sort": [
		{
			"studymodel": "desc"
		},
		{
			"price": "asc"
		}
	]
}
```



## Java客户端

```java
//Sort
@Test
public void testSort() throws IOException, ParseException {
    //搜索请求对象
    SearchRequest searchRequest = new SearchRequest("xc_course");
    //指定类型
    searchRequest.types("doc");
    //搜索源构建对象
    SearchSourceBuilder searchSourceBuilder = new SearchSourceBuilder();

    //boolQuery搜索方式
    //定义一个boolQuery
    BoolQueryBuilder boolQueryBuilder = QueryBuilders.boolQuery();
    //定义过虑器
    boolQueryBuilder.filter(QueryBuilders.rangeQuery("price").gte(0).lte(100));

    searchSourceBuilder.query(boolQueryBuilder);
    //添加排序
    searchSourceBuilder.sort("studymodel", SortOrder.DESC);
    searchSourceBuilder.sort("price", SortOrder.ASC);
    //设置源字段过虑,第一个参数结果集包括哪些字段，第二个参数表示结果集不包括哪些字段
    searchSourceBuilder.fetchSource(new String[]{"name","studymodel","price","timestamp"},new String[]{});
    //向搜索请求对象中设置搜索源
    searchRequest.source(searchSourceBuilder);
    //执行搜索,向ES发起http请求
    SearchResponse searchResponse = client.search(searchRequest);
    //搜索结果
    SearchHits hits = searchResponse.getHits();
    //匹配到的总记录数
    long totalHits = hits.getTotalHits();
    //得到匹配度高的文档
    SearchHit[] searchHits = hits.getHits();
    //日期格式化对象
    SimpleDateFormat dateFormat = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
    for(SearchHit hit:searchHits){
        //文档的主键
        String id = hit.getId();
        //源文档内容
        Map<String, Object> sourceAsMap = hit.getSourceAsMap();
        String name = (String) sourceAsMap.get("name");
        //由于前边设置了源文档字段过虑，这时description是取不到的
        String description = (String) sourceAsMap.get("description");
        //学习模式
        String studymodel = (String) sourceAsMap.get("studymodel");
        //价格
        Double price = (Double) sourceAsMap.get("price");
        //日期
        Date timestamp = dateFormat.parse((String) sourceAsMap.get("timestamp"));
        System.out.println(name);
        System.out.println(studymodel);
        System.out.println(description);
    }

}
```





# 高亮显示

## API

高亮显示可以将搜索结果一个或多个字突出显示，以便向用户展示匹配关键字的位置。 

在搜索语句中添加highlight即可实现，如下： 

Post： http://127.0.0.1:9200/xc_course/doc/_search 

```json
{
	"_source": [
		"name",
		"studymodel",
		"description",
		"price"
	],
	"query": {
		"bool": {
			"must": [
				{
					"multi_match": {
						"query": "开发框架",
						"minimum_should_match": "50%",
						"fields": [
							"name^10",
							"description"
						],
						"type": "best_fields"
					}
				}
			],
			"filter": [
				{
					"range": {
						"price": {
							"gte": 0,
							"lte": 100
						}
					}
				}
			]
		}
	},
	"sort": [
		{
			"price": "asc"
		}
	],
	"highlight": {
		"pre_tags": [
			"<tag1>"
		],
		"post_tags": [
			"</tag2>"
		],
		"fields": {
			"name": {},
			"description": {}
		}
	}
}
```



## Java客户端

```java
  //Highlight
    @Test
    public void testHighlight() throws IOException, ParseException {
        //搜索请求对象
        SearchRequest searchRequest = new SearchRequest("xc_course");
        //指定类型
        searchRequest.types("doc");
        //搜索源构建对象
        SearchSourceBuilder searchSourceBuilder = new SearchSourceBuilder();

        //boolQuery搜索方式
        //先定义一个MultiMatchQuery
        MultiMatchQueryBuilder multiMatchQueryBuilder = QueryBuilders.multiMatchQuery("开发框架", "name", "description")
                .minimumShouldMatch("50%")
                .field("name", 10);

        //定义一个boolQuery
        BoolQueryBuilder boolQueryBuilder = QueryBuilders.boolQuery();
        boolQueryBuilder.must(multiMatchQueryBuilder);
        //定义过虑器
        boolQueryBuilder.filter(QueryBuilders.rangeQuery("price").gte(0).lte(100));

        searchSourceBuilder.query(boolQueryBuilder);
        //设置源字段过虑,第一个参数结果集包括哪些字段，第二个参数表示结果集不包括哪些字段
        searchSourceBuilder.fetchSource(new String[]{"name","studymodel","price","timestamp"},new String[]{});

        //设置高亮
        HighlightBuilder highlightBuilder = new HighlightBuilder();
        highlightBuilder.preTags("<tag>");
        highlightBuilder.postTags("</tag>");
        highlightBuilder.fields().add(new HighlightBuilder.Field("name"));
        highlightBuilder.fields().add(new HighlightBuilder.Field("description"));
        searchSourceBuilder.highlighter(highlightBuilder);

        //向搜索请求对象中设置搜索源
        searchRequest.source(searchSourceBuilder);
        //执行搜索,向ES发起http请求
        SearchResponse searchResponse = client.search(searchRequest);
        //搜索结果
        SearchHits hits = searchResponse.getHits();
        //匹配到的总记录数
        long totalHits = hits.getTotalHits();
        //得到匹配度高的文档
        SearchHit[] searchHits = hits.getHits();
        //日期格式化对象
//        SimpleDateFormat dateFormat = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
        SimpleDateFormat dateFormat = new SimpleDateFormat("yyyy-MM-dd'T'HH:mm:ss.SSS Z");
        for(SearchHit hit:searchHits){
            //文档的主键
            String id = hit.getId();
            //源文档内容
            Map<String, Object> sourceAsMap = hit.getSourceAsMap();
            //源文档的name字段内容
            String name = (String) sourceAsMap.get("name");
            //取出高亮字段
            Map<String, HighlightField> highlightFields = hit.getHighlightFields();
            if(highlightFields!=null){
                //取出name高亮字段
                HighlightField nameHighlightField = highlightFields.get("name");
                if(nameHighlightField!=null){
                    Text[] fragments = nameHighlightField.getFragments();
                    StringBuffer stringBuffer = new StringBuffer();
                    for(Text text:fragments){
                        stringBuffer.append(text);
                    }
                    name = stringBuffer.toString();
                }
            }

            //由于前边设置了源文档字段过虑，这时description是取不到的
            String description = (String) sourceAsMap.get("description");
            //学习模式
            String studymodel = (String) sourceAsMap.get("studymodel");
            //价格
            Double price = (Double) sourceAsMap.get("price");
            //日期
            Date timestamp = dateFormat.parse((String) sourceAsMap.get("timestamp"));
            System.out.println(name);
            System.out.println(studymodel);
            System.out.println(description);
        }

    }
```

