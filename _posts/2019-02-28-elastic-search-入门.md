---
categories: tool
layout: post
---

- Table
{:toc}
# 基础

Elasticsearch是一个高度可伸缩的开源全文搜索和分析引擎。它允许你以接近线性的速度高效地存储，搜索和分析海量数据。它通常被用作底层引擎来驱动拥有复杂搜索特性和需求的上层应用。

## 用例

下面是一些用例：

- 你经营一个线上商店，允许你的顾客搜索你销售的商品，这场景下你使用ES存储你整个产品目录和存货清单，并向顾客提供搜索和自动补全建议
- 你希望收集日志或事务相关数据，并希望通过分析开采这些数据以提取趋势，静态数据，总结或是异常。这种场景下，你可以使用Logstash来搜集、聚合并解析数据，并通过Logstash将这些数据喂给ES。一旦数据到了ES，你就可以通过搜索和聚合来挖掘任何感兴趣的信息。
- 你可以允许一个价格预警平台，允许价格敏感客户指定一个规则，比如“我想要购买一个特定的电子小配件，并且我希望下个月内如果任何供应商中这个配件的价格跌落到$x下我能收到通知”。在这种情况下，你可以爬取供应商价格，并推送到ES中，通过ES的反向搜索的能力来查找匹配客户请求的价格波动信息，一旦发现匹配则以警告的方式推送到客户。
- 你有分析/智能业务需求并希望能快速地研究、分析、可视化并在数十亿几笔的数据上点对点提问。在这种情况下，你可以使用ES存储你的数据并使用Kibana来构建控制面板，来渲染你认为重要的数据层面。你可以额外使用ES的聚合能力来基于你的数据实现复杂的智能业务查询。

## 近线性（Near Realtime，NRT）

ES是近线性的搜索平台，这意味着在你索引你的文档后直到这个文档可以被搜索存在一个轻微的延迟（一般是一秒）。

## 节点

节点是你的集群中的一台服务器，存储你的数据，并参与集群中的索引和搜索过程。和集群一样，一个节点由名字唯一标识，名字默认是节点启动时被分配的一个随机UUID。如果你不打算使用默认名字，你可以定义任何节点名字。当你希望了解服务器和节点的对应关系时，名字对管理有着重要价值。

一个节点可以通过集群名字加入到指定的集群中。默认情况下，所有节点都被设置为加入到名字是elasticsearch的集群。这意味着如果你启动了若干个节点，并假定他们能互相发现，那么他们会自动组成一个单独的名为elasticsearch的集群。

在一个集群中，可以包含任意多的节点。更进一步，如果没有其他的ES节点处于运行状态，启动单独节点默认会形成一个单节点集群，集群名字是elasticsearch。

## 索引

索引是拥有一些相似特性的文档的集合，比如，你可以为顾客数据增加一个索引，为产品目录增加一个索引，并为订单数据增加一个索引。一个索引通过名字唯一标识（名字必须为全小写），当对索引内稳定执行索引、查询、更新、删除操作时需要通过索引名字引用这个搜易。

在一个集群中，你可以定义任意多的索引。

## 文档

文档是可以被索引的基础信息单元。比如，你可以为一个单独的顾客创建一个稳定，为一个单独的产品创建稳定，并为一个单独的订单创建文档。文档以JSON格式表示，在一个索引中，你可以存储任意多的文档。

## 分片和副本

一个索引可能存储大量的数据，最终超出单个节点硬件的限制。比如拥有十亿级别的文档的单个索引占用了1TB的硬盘空间，这可能无法在单个节点上存储，或在单个节点上处理请求会变得非常慢。

要解决这个问题，ES提供了将索引切分成多个分片的能力。当你创建一个索引，你可以简单定义你希望的分片的数量。每个分片都是完整而独立的索引，并可以保存在任何集群中的节点上。

需要分片有两个重要原因：

- 它允许你水平扩展你的卷大小
- 它允许你分散索引提供的操作从而提高性能

而具体的如何分散分片以及如果将数个请求结果聚合为一个响应都由ES完全管理，这些对用户来说是透明的。

在一个集群中，失败可能在任何时候发生，非常推荐有一个失败恢复的机制以避免单个切片或节点因某些原因下线或消失。ES允许你为你的索引分片创建多份副本。

需要副本有两个重要原因：

- 它为你提供了高可用性，不会因为单个分片或节点丢失而失败。（一个节点最多存储分片的一个副本）
- 它允许你水平扩展并通过在副本上执行读操作分担搜索带来的压力，从而提高搜索的吞吐量。

总结下来，一个索引可以有多个分片，一个分片可以有多个副本。多个副本中第一个创建的副本即为主副本（其它副本均拷贝自主副本）。不同的索引都可以有自定义的分片和副本数，在索引创建的时候一同指定。同时你可以在索引创建后修改副本数。默认情况下，ES中的索引的副本数都是2（即一个主副本，一个普通副本）。

# 安装

## 单机安装

使用docker直接部署，compose文件如下。

```yaml
version: "3.0"

services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:6.4.2
    container_name: elasticsearch
    environment:
      - cluster.name=docker-cluster
      - bootstrap.memory_lock=false
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - esdata1:/usr/share/elasticsearch/data
    ports:
      - 9200:9200
    networks:
      - esnet
  elasticsearch2:
    image: docker.elastic.co/elasticsearch/elasticsearch:6.4.2
    container_name: elasticsearch2
    environment:
      - cluster.name=docker-cluster
      - bootstrap.memory_lock=false
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
      - "discovery.zen.ping.unicast.hosts=elasticsearch"
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - esdata2:/usr/share/elasticsearch/data
    networks:
      - esnet
  kibana:
    image: docker.elastic.co/kibana/kibana:6.4.2
    environment:
      SERVER_NAME: kibana
      ELASTICSEARCH_URL: http://elasticsearch:9200
      SERVER_PORT: 9201
    ports:
      - 9201:9201
    networks:
      - esnet
volumes:
  esdata1:
    driver: local
  esdata2:
    driver: local

networks:
  esnet:
```

# 上手

现在我们的ES已经正常启动了。接下来我们来探索ES提供的各种特性。非常幸运的是，ES提供了非常易于理解和强力的REST风格API，用于与集群交互。通过这些API，你可以完成下面事情：

- 检查集群、节点和索引的健康、状况和数据。
- 管理你的集群、节点、索引数据和元数据
- 对你的索引指向CRUD和搜索操作
- 指向高级搜索操作，比如分页，排序，过滤，脚本，聚合和其它。

## 集群健康检查

让我们先从基础健康检查入手，借此我们可以看到我们的集群是如何工作的。下面我们通过Kibana的Dev Tools来完成请求，但是你也可以借助其它可以允许你指向HTTP请求的工具。假设我们现在位于ES启动的相同节点上。

要检查集群健康，我们需要使用_cat接口。

```http
$ GET /_cat/health?v
-----------------------------
epoch      timestamp cluster        status node.total node.data shards pri relo init unassign pending_tasks max_task_wait_time active_shards_percent
1551325245 03:40:45  docker-cluster green           2         2      0   0    0    0        0             0                  -                100.0%
```

可以看到我们的集群docker-cluster已经启动了并包含2个节点，而状态为green。集群的健康状况可以取下面三个值。

- 绿色——所有的部分都良好工作
- 黄色——所有的数据都是可用的，但是一些副本还没有被分配，不影响ES的功能
- 红色——一些数据因为某些原因不可用了

当一个集群是红色的，它依旧可以正常处理一些仅使用可用分片的操作。

查看我们两个节点的状态。

```http
GET /_cat/nodes?v
-----------------------------
ip         heap.percent ram.percent cpu load_1m load_5m load_15m node.role master name
10.0.2.115           18          92   3    0.02    0.05     0.17 mdi       -      yMb1R4E
10.0.2.113           22          92   4    0.02    0.05     0.17 mdi       *      QGjErJl

```

## 列举索引

列举索引使用的也会_cat接口。

```http
GET /_cat/indices?v
-----------------------------
health status index uuid pri rep docs.count docs.deleted store.size pri.store.size
```

可以看到目前我们还未建立任何索引。

## 创建索引

接下来我们创建一个名为customer的索引，使用pretty在请求的后面，要求ES返回格式化的JSON响应。

```http
PUT /customer?pretty
-----------------------------
#! Deprecation: the default number of shards will change from [5] to [1] in 7.0.0; if you wish to continue using the default of [5] shards, you must manage this on the create index request or with an index template
{
  "acknowledged": true,
  "shards_acknowledged": true,
  "index": "customer"
}
```

接下来列出索引。

```http
$ GET /_cat/indices?v
-----------------------------
health status index    uuid                   pri rep docs.count docs.deleted store.size pri.store.size
green  open   customer AVFURWHdSjilknSD6AYfBg   5   1          0            0      2.2kb          1.1kb
```

rep字段表示副本数目为1。

## 索引和查询文档

接下来我们向之前创建的customer索引加入点文档。

```http
PUT /customer/_doc/1?pretty
{
  "name": "John Doe"
}
-----------------------------
{
  "_index": "customer",
  "_type": "_doc",
  "_id": "1",
  "_version": 1,
  "result": "created",
  "_shards": {
    "total": 2,
    "successful": 2,
    "failed": 0
  },
  "_seq_no": 0,
  "_primary_term": 1
}
```

可以看到一个新的顾客文档在customer索引内成功创建。这个文档同时以1作为外部ID。

需要注意的是你不必保证在索引创建后再加入文档，如果文档加入时索引不存在，ES会为你自动创建索引。

接下来让我们通过外部ID查询文档。

```http
GET /customer/_doc/1?pretty
-----------------------------
{
  "_index": "customer",
  "_type": "_doc",
  "_id": "1",
  "_version": 1,
  "found": true,
  "_source": {
    "name": "John Doe"
  }
}
```

found指示再索引中确实找到了对应的文档，而_source中存放找到的文档。

```http
GET /_cat/indices?v
-----------------------------
health status index    uuid                   pri rep docs.count docs.deleted store.size pri.store.size
green  open   customer AVFURWHdSjilknSD6AYfBg   5   1          1            0      8.9kb          4.4kb
```

customer索引的文档数变成了1。

## 删除索引

接下来让我们删除刚创建的索引。

```http
DELETE /customer?pretty
-----------------------------
{
  "acknowledged": true
}
```

acknowledged字段表示删除操作被接受。现在我们又回到了开始的状态。

让我们回顾一下至今对索引的操作。

```http
PUT /customer
PUT /customer/_doc/1
{
  "name": "John Doe"
}
GET /customer/_doc/1
DELETE /customer
```

可以总结出访问ES中的数据的模式。

```http
<HTTP Verb> /<Index>/<Endpoint>/<ID>
```

这个REST访问模式在所有的API命令中是如此的普遍，以致于你只要简单记住它，你就在主宰ES的路上开了个好头。

## 替换文档

ES在近线性时间内提供数据操作和搜索能力。默认情况下，你可以期待在你修改了数据的一秒延迟后，你的修改就会在搜索结果中体现出来。这是ES与其他平台（比如SQL，数据在事务完成后立即可用）的重要不同。

我们之前已经了解过如果索引一个文档。

```http
PUT /customer/_doc/1?pretty
{
  "name": "John Doe"
}
-----------------------------
{
  "_index": "customer",
  "_type": "_doc",
  "_id": "1",
  "_version": 1,
  "result": "created",
  "_shards": {
    "total": 2,
    "successful": 2,
    "failed": 0
  },
  "_seq_no": 0,
  "_primary_term": 1
}
```

如果我们再次以不同（或相同）的文档执行上面命令时，ES会替换ID为1的文档的内容。

```http
PUT /customer/_doc/1?pretty
{
  "name": "Jane Doe"
}
-----------------------------
{
  "_index": "customer",
  "_type": "_doc",
  "_id": "1",
  "_version": 3,
  "result": "updated",
  "_shards": {
    "total": 2,
    "successful": 2,
    "failed": 0
  },
  "_seq_no": 2,
  "_primary_term": 1
}
```

可以看到两次操作的result是不同的，前者是created，后者是updated。

如果我们在提交文档时不指定id，那么ES会为这个文档生成一个随机ID，这个ID会在结果的_id字段返回。

```http
POST /customer/_doc?pretty
{
  "name": "Jane Doe"
}
-----------------------------
{
  "_index": "customer",
  "_type": "_doc",
  "_id": "GGheMmkBQQQPZW-HwQOp",
  "_version": 1,
  "result": "created",
  "_shards": {
    "total": 2,
    "successful": 2,
    "failed": 0
  },
  "_seq_no": 1,
  "_primary_term": 1
}
```

## 更新文档

除了完整替换文档外，我们还可以选择更新部分文档。注意ES并不会在后台执行原址更新，每次更新文档，都会删除旧的文档并索引新的文档。

```http
POST /customer/_doc/1/_update?pretty
{
  "doc": { "name": "Jane Doe" }
}
-----------------------------
{
  "_index": "customer",
  "_type": "_doc",
  "_id": "1",
  "_version": 5,
  "result": "noop",
  "_shards": {
    "total": 0,
    "successful": 0,
    "failed": 0
  }
}
```

我们不仅可以修改已有字段，还能增加新的字段。

```http
POST /customer/_doc/1/_update?pretty
{
  "doc": { "name": "Jane Doe", "age": 20 }
}
-----------------------------
{
  "_index": "customer",
  "_type": "_doc",
  "_id": "1",
  "_version": 6,
  "result": "noop",
  "_shards": {
    "total": 0,
    "successful": 0,
    "failed": 0
  }
}
```

我们还能通过一个简单脚本来实现更新。

```http
POST /customer/_doc/1/_update?pretty
{
  "script" : "ctx._source.age += 5"
}
-----------------------------
{
  "_index": "customer",
  "_type": "_doc",
  "_id": "1",
  "_version": 11,
  "result": "updated",
  "_shards": {
    "total": 2,
    "successful": 2,
    "failed": 0
  },
  "_seq_no": 10,
  "_primary_term": 1
}
```

上面这个脚本中，ctx._source引用了当前被更新的文档。

## 删除文档

删除一个稳定是相当直接的。

```http
DELETE /customer/_doc/2?pretty
-----------------------------
{
  "_index": "customer",
  "_type": "_doc",
  "_id": "2",
  "_version": 2,
  "result": "deleted",
  "_shards": {
    "total": 2,
    "successful": 2,
    "failed": 0
  },
  "_seq_no": 2,
  "_primary_term": 1
}
```

## 批量处理

除了一次操作一个文档，ES还提供了用于批量执行操作的_bulk接口。这个功能提供了一种有效的机制快速执行逗哥操作，并减少网络往返。

创建两个文档。

```http
POST /customer/_doc/_bulk?pretty
{"index":{"_id":"1"}}
{"name": "John Doe" }
{"index":{"_id":"2"}}
{"name": "Jane Doe" }
-----------------------------
{
  "took": 63,
  "errors": false,
  "items": [
    {
      "index": {
        "_index": "customer",
        "_type": "_doc",
        "_id": "1",
        "_version": 12,
        "result": "updated",
        "_shards": {
          "total": 2,
          "successful": 2,
          "failed": 0
        },
        "_seq_no": 11,
        "_primary_term": 1,
        "status": 200
      }
    },
    {
      "index": {
        "_index": "customer",
        "_type": "_doc",
        "_id": "2",
        "_version": 1,
        "result": "created",
        "_shards": {
          "total": 2,
          "successful": 2,
          "failed": 0
        },
        "_seq_no": 3,
        "_primary_term": 1,
        "status": 201
      }
    }
  ]
}
```

下面这个命令更新了文档1并删除文档2。

```http
POST /customer/_doc/_bulk?pretty
{"update":{"_id":"1"}}
{"doc": { "name": "John Doe becomes Jane Doe" } }
{"delete":{"_id":"2"}}
-----------------------------
{
  "took": 29,
  "errors": false,
  "items": [
    {
      "update": {
        "_index": "customer",
        "_type": "_doc",
        "_id": "1",
        "_version": 13,
        "result": "updated",
        "_shards": {
          "total": 2,
          "successful": 2,
          "failed": 0
        },
        "_seq_no": 12,
        "_primary_term": 1,
        "status": 200
      }
    },
    {
      "delete": {
        "_index": "customer",
        "_type": "_doc",
        "_id": "2",
        "_version": 2,
        "result": "deleted",
        "_shards": {
          "total": 2,
          "successful": 2,
          "failed": 0
        },
        "_seq_no": 4,
        "_primary_term": 1,
        "status": 200
      }
    }
  ]
}
```

由于删除只需要提供ID，因此你不需要在请求尾部加上额外的文档。

Bulk接口不会因为一个操作失败而停止，它会执行完所有的操作，并将结果一同返回，结果的顺序和请求中操作的顺序一致，你可以根据结果判断每个操作是否成功。

## 探索数据

现在我们已经瞥了一眼基础，接着让我们在更加真实的数据集上工作。我已经准备了一个虚构的JSON文档，其中记录顾客银行账户信息。每个文档有如下格式：

```json
{
    "account_number": 0,
    "balance": 16623,
    "firstname": "Bradshaw",
    "lastname": "Mckenzie",
    "age": 29,
    "gender": "F",
    "address": "244 Columbus Place",
    "employer": "Euron",
    "email": "bradshawmckenzie@euron.com",
    "city": "Hobucken",
    "state": "CO"
}
```

数据由[https://www.json-generator.com/](https://www.json-generator.com/)生成，所以不必探究实际值和数据语法。

你可以在[这里](https://raw.githubusercontent.com/elastic/elasticsearch/master/docs/src/test/resources/accounts.json)下载到样例数据集，将它提取到目录下并通过批量接口提交这些操作。

## 搜索接口

接下来让我们执行一些简单的搜索。有两种执行搜索的方式：一种是通过REST请求URI的方式发送搜索参数，或者通过REST请求body的方式发送搜索参数。body的方式更具表达力，并将你的搜索以可读的JSON格式进行定义。

为了演示，我们还是会展示如何使用URI的方式提交搜索参数，但是之后我们将统一使用body的方式提交参数。

 ```http
GET /bank/_search?q=*&sort=account_number:asc&pretty
-----------------------------
{
  "took": 234,
  "timed_out": false,
  "_shards": {
    "total": 5,
    "successful": 5,
    "skipped": 0,
    "failed": 0
  },
  "hits": {
    "total": 1000,
    "max_score": null,
    "hits": [
      {
        "_index": "bank",
        "_type": "_doc",
        "_id": "0",
        "_score": null,
        "_source": {
          "account_number": 0,
          "balance": 16623,
          "firstname": "Bradshaw",
          "lastname": "Mckenzie",
          "age": 29,
          "gender": "F",
          "address": "244 Columbus Place",
          "employer": "Euron",
          "email": "bradshawmckenzie@euron.com",
          "city": "Hobucken",
          "state": "CO"
        },
        "sort": [
          0
        ]
      },
 	  ...
    ]
  }
}
 ```

来了解一下URI的含义，bank指定在bank索引下，_search端点指定搜索操作，q=\*参数要求ES匹配索引中存储的所有文档。sort=account_number:asc参数指示按照account_numbre字段升序排序结果。pretty参数告诉ES返回格式化后的JSON结果。

除了hits中的请求结果外，还包含了下面部分的内容：

- took - ES执行搜索花费的毫秒数
- timed_out - 告诉我们搜索是否超时
- _shards - 告诉我们有搜索了多少分片，以及多少分片搜索成功，多少失败。
- hits - 搜索结果
- hits.total - 匹配我们搜索条件的文档总数
- hits.hits - 实际结果数组（默认前10条）
- hits.sort - 用于排序的字段（如果按分数排序则为空）
- hits._score和max_score - 目前不用管

将上面URI形式的查询请求转换为更清晰的body格式。

```http
GET /bank/_search
{
  "query": { "match_all": {} },
  "sort": [
    { "account_number": "asc" }
  ]
}
```

理解一旦你取回搜索结果，ES就完成了请求并且不会再服务器端保存任何维护结果的资源是非常重要的。这与其他诸如SQL的带状态的平台不同，在这些平台中，你可以获取结果的子集，之后可以通过不断请求服务器去获取剩下的内容。

## 介绍查询语言

ES提供了JSON风格的DSL，你可以通过使用DSL来执行查询。这种DSL称为Query DSL。这种语言非常易于理解，在初见时可能会感觉非常复杂，但是学习它的最好的方法是通过几个简单的案例。

回到我们最后的例子：

```http
GET /bank/_search
{
  "query": { "match_all": {} },
  "sort": [
    { "account_number": "asc" }
  ]
}
```

query部分指定了查询条件，match_all部分会不做任何过滤。除了query部分，我们还可以传递其它参数来影响搜索结果，上面我们传递了sort，下面我们将使用size。

```http
GET /bank/_search
{
  "query": { "match_all": {} },
  "size": 1
}
```

size指定最多能返回几条结果，默认值是10。

下面例子返回第10~19条记录。

```http
GET /bank/_search
{
  "query": { "match_all": {} },
  "from": 10,
  "size": 10
}
```

下面请求我们按照balance进行逆向排序。

```http
GET /bank/_search
{
  "query": { "match_all": {} },
  "sort": { "balance": { "order": "desc" } }
}
```

## 执行搜索

我们已经了解了一些基础的查询参数，让我们在Query DSL上再深挖一些。让我们看一下返回的文档中的字段，默认情况下，整个JSON文档都会作为_source被返回。如果你不需要整个文档，我们可以仅请求原始文档的部分的字段。

下面这个案例仅返回文档的account_number和balance字段。

```http
GET /bank/_search
{
  "query": { "match_all": {} },
  "_source": ["account_number", "balance"]
}
```

接下来让我们关注query部分，之前我们已经看到match_all请求可以用来匹配所有文档，接下来让我们介绍match查询。match查询可以认为是对于单个基础字段的过滤。

下面的例子返回account_numer为20的文档。

```http
GET /bank/_search
{
  "query": { "match": { "account_number": 20 } }
}
```

下面的例子返回所有地址中包含单词（term）"mill"的文档。

```http
GET /bank/_search
{
  "query": { "match": { "address": "mill" } }
}
```

下面的例子返回所有地址中包含单词"mill"或"lane"的文档。

```http
GET /bank/_search
{
  "query": { "match": { "address": "mill lane" } }
}
```

下面例子使用了match的变种match_phrase来搜索所有地中中包含短语"mill lane"的文档。

```http
GET /bank/_search
{
  "query": { "match_phrase": { "address": "mill lane" } }
}
```

下面让我们介绍bool查询。bool查询允许我们将一些较小的查询通过布尔逻辑组合成一个更大的查询。

下面的例子使用must以逻辑且组合了两个match查询搜索所有地址包含"mill"和"lane"的文档。

```http
GET /bank/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "address": "mill" } },
        { "match": { "address": "lane" } }
      ]
    }
  }
}
```

上面的must条件会对所有的子条件进行且运算。因此要通过must过滤，必须通过它的所有子条件。

相反，下面的例子通过should以逻辑或组合了两个match条件，返回地址包含"mill"或"lane"的文档。

```http
GET /bank/_search
{
  "query": {
    "bool": {
      "should": [
        { "match": { "address": "mill" } },
        { "match": { "address": "lane" } }
      ]
    }
  }
}
```

下面例子通过逻辑或后取反组合了两个查询条件，返回所有地址既不包含"mill"也不包含"lane"的文档。

```http
GET /bank/_search
{
  "query": {
    "bool": {
      "must_not": [
        { "match": { "address": "mill" } },
        { "match": { "address": "lane" } }
      ]
    }
  }
}
```

bool下面可以有多个子查询，它们以且运算组合。下面例子返回所有40岁但是不居住在ID州的文档。

```http
GET /bank/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "age": "40" } }
      ],
      "must_not": [
        { "match": { "state": "ID" } }
      ]
    }
  }
}
```

你可以将bool作为普通的过滤器任意组合。比如下面我们搜索返回所有40岁但是不居住在ID州的文档和

所有地址包含"mill"和"lane"的文档的并集。

```http
GET /bank/_search?pretty
{
    "query": {
        "bool": {
            "should": [
                {
                    "bool": {
                        "must": [{
                            "match": {
                                "age": "40"
                            }
                        }],
                        "must_not": [{
                            "match": {
                                "state": "ID"
                            }
                        }]
                    }
                },
                {
                    "bool": {
                        "must": [{
                                "match": {
                                    "address": "mill"
                                }
                            },
                            {
                                "match": {
                                    "address": "lane"
                                }
                            }
                        ]
                    }
                }
            ]
        }
    }
}
```

## 执行过滤器

之前的章节，我们跳过了文档分数的细节（对应结果中的_score字段）。文档分数是一个数值，用于评估文档和搜索条件的匹配程度。分数越高，文档越接近条件，分数越低，文档越偏离条件。

但是查询不一定会生成分数，尤其条件仅用来过滤文档集合。ES会监测到这些场景，并自动优化执行的查询，以避免计算无用的分数。

我们之前介绍的bool查询，也支持filter条件，允许我们使用一个查询来约束返回的文档，而不必修改分数的计算方式。作为一个样例，让我们介绍range查询，允许我们通过一个范围来过滤文档。这个通常用于数值或日期字段。

这个例子使用一个bool查询来返回所有拥有20000到30000之间余额的文档。

```http
GET /bank/_search
{
  "query": {
    "bool": {
      "must": { "match_all": {} },
      "filter": {
        "range": {
          "balance": {
            "gte": 20000,
            "lte": 30000
          }
        }
      }
    }
  }
}
```

仔细分析上面的请求，bool查询包含一个match_all查询，以及一个过滤器。

## 执行聚合器

聚合器提供了分组和从数据中提取统计数据的能力。可以简单地将聚合器视作SQL的GROUP BY和SQL的聚合函数。ES可以在执行搜索并返回hits的同时返回聚合结果。这是非常有限的方式，你可以在执行搜索的同时指定多个聚合器，并一同作为结果接受，通过更加紧密和简单的接口避免了网络的往返。

作为开始，下面这个样例对所有账户通过state进行分组，返回人口量前10的州。

```http
GET /bank/_search
{
  "size": 0,
  "aggs": {
    "group_by_state": {
      "terms": {
        "field": "state.keyword"
      }
    }
  }
}
-----------------------------
{
  "took": 59,
  "timed_out": false,
  "_shards": {
    "total": 5,
    "successful": 5,
    "skipped": 0,
    "failed": 0
  },
  "hits": {
    "total": 1000,
    "max_score": 0,
    "hits": []
  },
  "aggregations": {
    "group_by_state": {
      "doc_count_error_upper_bound": 20,
      "sum_other_doc_count": 770,
      "buckets": [
        {
          "key": "ID",
          "doc_count": 27
        },
        {
          "key": "TX",
          "doc_count": 27
        },
        {
          "key": "AL",
          "doc_count": 25
        },
        {
          "key": "MD",
          "doc_count": 25
        },
        {
          "key": "TN",
          "doc_count": 23
        },
        {
          "key": "MA",
          "doc_count": 21
        },
        {
          "key": "NC",
          "doc_count": 21
        },
        {
          "key": "ND",
          "doc_count": 21
        },
        {
          "key": "ME",
          "doc_count": 20
        },
        {
          "key": "MO",
          "doc_count": 20
        }
      ]
    }
  }
}
```

等价的SQL大概是

```sql
SELECT state, COUNT(*) FROM bank GROUP BY state ORDER BY COUNT(*) DESC LIMIT 10;
```

注意我们的请求中将size设置为了0，禁止服务器返回hits信息。

在上面案例的基础上，下面这个案例计算每个州的平均余额，并仅返平均人数最多的10个州。

```http
GET /bank/_search
{
  "size": 0,
  "aggs": {
    "group_by_state": {
      "terms": {
        "field": "state.keyword"
      },
      "aggs": {
        "average_balance": {
          "avg": {
            "field": "balance"
          }
        }
      }
    }
  }
}
-----------------------------
{
  "took": 73,
  "timed_out": false,
  "_shards": {
    "total": 5,
    "successful": 5,
    "skipped": 0,
    "failed": 0
  },
  "hits": {
    "total": 1000,
    "max_score": 0,
    "hits": []
  },
  "aggregations": {
    "group_by_state": {
      "doc_count_error_upper_bound": 20,
      "sum_other_doc_count": 770,
      "buckets": [
        {
          "key": "ID",
          "doc_count": 27,
          "average_balance": {
            "value": 24368.777777777777
          }
        },
        {
          "key": "TX",
          "doc_count": 27,
          "average_balance": {
            "value": 27462.925925925927
          }
        },
        {
          "key": "AL",
          "doc_count": 25,
          "average_balance": {
            "value": 25739.56
          }
        },
        {
          "key": "MD",
          "doc_count": 25,
          "average_balance": {
            "value": 24963.52
          }
        },
        {
          "key": "TN",
          "doc_count": 23,
          "average_balance": {
            "value": 29796.782608695652
          }
        },
        {
          "key": "MA",
          "doc_count": 21,
          "average_balance": {
            "value": 29726.47619047619
          }
        },
        {
          "key": "NC",
          "doc_count": 21,
          "average_balance": {
            "value": 26785.428571428572
          }
        },
        {
          "key": "ND",
          "doc_count": 21,
          "average_balance": {
            "value": 26303.333333333332
          }
        },
        {
          "key": "ME",
          "doc_count": 20,
          "average_balance": {
            "value": 19575.05
          }
        },
        {
          "key": "MO",
          "doc_count": 20,
          "average_balance": {
            "value": 24151.8
          }
        }
      ]
    }
  }
}
```

注意到上面我们再group_by_state聚合中嵌套了average_balance聚合器，这是所有聚合器的一种通用模式。你可以任意嵌套聚合器以从数据中提取摘要。

在上一个例子的基础上，接下来让我们将聚合信息按照平均余额降序排序。

```http
GET /bank/_search
{
  "size": 0,
  "aggs": {
    "group_by_state": {
      "terms": {
        "field": "state.keyword",
        "order": {
          "average_balance": "desc"
        }
      },
      "aggs": {
        "average_balance": {
          "avg": {
            "field": "balance"
          }
        }
      }
    }
  }
}
-----------------------------
{
  "took": 84,
  "timed_out": false,
  "_shards": {
    "total": 5,
    "successful": 5,
    "skipped": 0,
    "failed": 0
  },
  "hits": {
    "total": 1000,
    "max_score": 0,
    "hits": []
  },
  "aggregations": {
    "group_by_state": {
      "doc_count_error_upper_bound": -1,
      "sum_other_doc_count": 918,
      "buckets": [
        {
          "key": "AL",
          "doc_count": 6,
          "average_balance": {
            "value": 41418.166666666664
          }
        },
        {
          "key": "SC",
          "doc_count": 1,
          "average_balance": {
            "value": 40019
          }
        },
        {
          "key": "AZ",
          "doc_count": 10,
          "average_balance": {
            "value": 36847.4
          }
        },
        {
          "key": "VA",
          "doc_count": 13,
          "average_balance": {
            "value": 35418.846153846156
          }
        },
        {
          "key": "DE",
          "doc_count": 8,
          "average_balance": {
            "value": 35135.375
          }
        },
        {
          "key": "WA",
          "doc_count": 7,
          "average_balance": {
            "value": 34787.142857142855
          }
        },
        {
          "key": "ME",
          "doc_count": 3,
          "average_balance": {
            "value": 34539.666666666664
          }
        },
        {
          "key": "OK",
          "doc_count": 9,
          "average_balance": {
            "value": 34529.77777777778
          }
        },
        {
          "key": "CO",
          "doc_count": 13,
          "average_balance": {
            "value": 33379.769230769234
          }
        },
        {
          "key": "MI",
          "doc_count": 12,
          "average_balance": {
            "value": 32905.916666666664
          }
        }
      ]
    }
  }
}
```

下面的案例按照年龄段（20-29，30-39，40-49）进行分组，之后按照性别分组，最终计算平均余额。

```http
GET /bank/_search
{
  "size": 0,
  "aggs": {
    "group_by_age": {
      "range": {
        "field": "age",
        "ranges": [
          {
            "from": 20,
            "to": 30
          },
          {
            "from": 30,
            "to": 40
          },
          {
            "from": 40,
            "to": 50
          }
        ]
      },
      "aggs": {
        "group_by_gender": {
          "terms": {
            "field": "gender.keyword"
          },
          "aggs": {
            "average_balance": {
              "avg": {
                "field": "balance"
              }
            }
          }
        }
      }
    }
  }
}
-----------------------------
{
  "took": 84,
  "timed_out": false,
  "_shards": {
    "total": 5,
    "successful": 5,
    "skipped": 0,
    "failed": 0
  },
  "hits": {
    "total": 1000,
    "max_score": 0,
    "hits": []
  },
  "aggregations": {
    "group_by_state": {
      "doc_count_error_upper_bound": -1,
      "sum_other_doc_count": 918,
      "buckets": [
        {
          "key": "AL",
          "doc_count": 6,
          "average_balance": {
            "value": 41418.166666666664
          }
        },
        {
          "key": "SC",
          "doc_count": 1,
          "average_balance": {
            "value": 40019
          }
        },
        {
          "key": "AZ",
          "doc_count": 10,
          "average_balance": {
            "value": 36847.4
          }
        },
        {
          "key": "VA",
          "doc_count": 13,
          "average_balance": {
            "value": 35418.846153846156
          }
        },
        {
          "key": "DE",
          "doc_count": 8,
          "average_balance": {
            "value": 35135.375
          }
        },
        {
          "key": "WA",
          "doc_count": 7,
          "average_balance": {
            "value": 34787.142857142855
          }
        },
        {
          "key": "ME",
          "doc_count": 3,
          "average_balance": {
            "value": 34539.666666666664
          }
        },
        {
          "key": "OK",
          "doc_count": 9,
          "average_balance": {
            "value": 34529.77777777778
          }
        },
        {
          "key": "CO",
          "doc_count": 13,
          "average_balance": {
            "value": 33379.769230769234
          }
        },
        {
          "key": "MI",
          "doc_count": 12,
          "average_balance": {
            "value": 32905.916666666664
          }
        }
      ]
    }
  }
}
```
