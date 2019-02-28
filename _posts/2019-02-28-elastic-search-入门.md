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

```sh
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

```sh
GET /_cat/nodes?v
-----------------------------
ip         heap.percent ram.percent cpu load_1m load_5m load_15m node.role master name
10.0.2.115           18          92   3    0.02    0.05     0.17 mdi       -      yMb1R4E
10.0.2.113           22          92   4    0.02    0.05     0.17 mdi       *      QGjErJl

```

## 列举索引

列举索引使用的也会_cat接口。

```sh
GET /_cat/indices?v
-----------------------------
health status index uuid pri rep docs.count docs.deleted store.size pri.store.size
```

可以看到目前我们还未建立任何索引。

## 创建索引

接下来我们创建一个名为customer的索引，使用pretty在请求的后面，要求ES返回格式化的JSON响应。

```sh
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

```sh
$ GET /_cat/indices?v
-----------------------------
health status index    uuid                   pri rep docs.count docs.deleted store.size pri.store.size
green  open   customer AVFURWHdSjilknSD6AYfBg   5   1          0            0      2.2kb          1.1kb
```

rep字段表示副本数目为1。

## 索引和查询文档

接下来我们向之前创建的customer索引加入点文档。

```sh
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

```sh
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

```sh
GET /_cat/indices?v
-----------------------------
health status index    uuid                   pri rep docs.count docs.deleted store.size pri.store.size
green  open   customer AVFURWHdSjilknSD6AYfBg   5   1          1            0      8.9kb          4.4kb
```

customer索引的文档数变成了1。

## 删除索引

接下来让我们删除刚创建的索引。

```sh
DELETE /customer?pretty
-----------------------------
{
  "acknowledged": true
}
```

acknowledged字段表示删除操作被接受。现在我们又回到了开始的状态。

让我们回顾一下至今对索引的操作。

```sh
PUT /customer
PUT /customer/_doc/1
{
  "name": "John Doe"
}
GET /customer/_doc/1
DELETE /customer
```

可以总结出访问ES中的数据的模式。

```sh
<HTTP Verb> /<Index>/<Endpoint>/<ID>
```

这个REST访问模式在所有的API命令中是如此的普遍，以致于你只要简单记住它，你就在主宰ES的路上开了个好头。

## 替换文档

ES在近线性时间内提供数据操作和搜索能力。默认情况下，你可以期待在你修改了数据的一秒延迟后，你的修改就会在搜索结果中体现出来。这是ES与其他平台（比如SQL，数据在事务完成后立即可用）的重要不同。

我们之前已经了解过如果索引一个文档。

```sh
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

```sh
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

```sh
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

```sh
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

```sh
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

```sh
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

```sh
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

```sh
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

```sh
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

 ```sh
GET /bank/_search?q=*&sort=account_number:asc&pretty
-----------------------------
 ```

