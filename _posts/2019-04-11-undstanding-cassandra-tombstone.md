---
bg: cassandra.jpg
layout: post
title:  "[Cassandra]深入理解tombstones"
crawlertitle: "[Cassandra]深入理解tombstones"
summary: "[Cassandra]深入理解tombstones"
date:   2019-04-11 16:04:47 +0800
categories: posts
tags: ['Cassandra','翻译']
author: 世农
---
*翻译文章，侵权秒删*


我们最近在生产上部署了一套分布式系统，是用Cassandra支撑持久化存储的。

很快，我们发现Cassandra日志中有很多关于tombstone的警告。

```$xslt
WARN  [SharedPool-Worker-2] 2017-01-20 16:14:45,153 ReadCommand.java:508 - 
Read 5000 live rows and 4771 tombstone cells for query 
SELECT * FROM warehouse.locations WHERE token(address) >= token(D3-DJ-21-B-02) LIMIT 5000 
(see tombstone_warn_threshold)
```

一开始我们没有预料到数据库中有如此多的tombstones,因为我们才写入了很少量的数据。问了一圈人发现没有人能说清楚Cassandra内部发生了什么。

事实上，人们对tombstone最大的误解是总是把它和删除操作联系在一起。没错，删除数据时会产生tombstone,但接下来我们就会发现那不是唯一的场景。

## 查看sstables
Cassandra 提供了一个工具查看sstable中的内容：sstabledump. 这个工具在"cassandra-tools" 工具包中，默认安装是没有的。在debian家族的系统中（例如：Ubuntu)安装该工具非常简单：
```$xslt
sudo apt-get update
sudo apt-get install cassandra-tools
```
接下来我们将用sstabledump 来理解Cassandra如何存储数据，何时产生tombstone。

使用sstabledump的语法非常直观：
```$xslt
sstabledump /var/lib/cassandra/data/warehouse/locations-660dbcb0e4a211e6814a9116fc548b6b/mc-1-big-Data.db
```
sstabledump根据sstable文件以json格式显示它的内容。我们需要用nodetool把内存中的数据刷到磁盘才能看到sstable文件：
```$xslt
nodetool flush warehouse locations
```
现在一切准备就绪，让我们一起看一些产生tombstones的场景。

## Null值产生tombstones
写入操作也会产生tombstones。为什么？因为Cassandra不会存储"null"值。Null表示数据不存在。当某个字段不存在时，Cassandra返回Null.因此，当某个字段是Null时，Cassandra需要删除原有的值。
```$xslt
INSERT INTO movements (
  id,
  address, 
  item_id, 
  quantity,
  username
) VALUES (
  103,
  'D3-DJ-21-B-02', 
  '3600029145',
  2,
  null
);
```
上面的语句覆盖任何id为103的记录。那么，Cassandra是如何做的呢？是的，通过tombstone.
![avatar](http://www.beyondthelines.net/wp-content/uploads/2017/01/movement-tombstone-1.png)
相应的sstabledump产出如下：
```$xslt
[
  {
    "partition" : {
      "key" : [ "103" ],
      "position" : 0
    },
    "rows" : [
      {
        "type" : "row",
        "position" : 18,
        "liveness_info" : { "tstamp" : "2017-01-27T15:09:50.065224Z" },
        "cells" : [
          { "name" : "address", "value" : "D3-DJ-21-B-02" },
          { "name" : "item_d", "value" : "3600029145" },
          { "name" : "quantity", "value" : "2" },
          { "name" : "username", "deletion_info" : { "local_delete_time" : "2017-01-27T15:09:50Z" }
        ]
      }
    ]
  }
]

```
Cassandra在设计上对性能特意优化过，任何操作都被写入"只追加"的日志文件。当数据被删除时，我们无法直接从日志中删除数据，取而代之的是在日志中追加tombstone。

另外，Cassandra在写入之前不会先读对应的数据，因为那是一个很"昂贵"的操作（除一些轻量的事务外）。

因此，当上面的写入操作执行时，Cassandra会为username字段写入一个tombstone（即使该key之前没有数据）

下面我们看一个和前面非常相似的语句：
```$xslt
INSERT INTO movements (
  id,
  address, 
  item_id, 
  quantity
) VALUES (
  103,
  'D3-DJ-21-B-02', 
  '3600029145',
  2
);
```
不同的是,下面的语句不会产生tombstones。

![avatar](http://www.beyondthelines.net/wp-content/uploads/2017/01/movement-row-1.png)
```$xslt
[
  {
    "partition" : {
      "key" : [ "103" ],
      "position" : 0
    },
    "rows" : [
      {
        "type" : "row",
        "position" : 18,
        "liveness_info" : { "tstamp" : "2017-01-27T15:09:50.065224Z" },
        "cells" : [
          { "name" : "address", "value" : "D3-DJ-21-B-02" },
          { "name" : "item_d", "value" : "3600029145" },
          { "name" : "quantity", "value" : "2" }
        ]
      }
    ]
  }
]
```
如果是这个key第一次写入数据，两个语句执行得到相同的结果。不同的是第一个语句会产生tombstone，而第二个语句不会。

如果该key之前有数据，那么username这个字段最终的值会不一样。第一个语句执行后，username总是会伴随着tombstone的产生被删掉，不再返回值；第二个语句执行后，username字段不变，返回原有的值。
![avatar](http://www.beyondthelines.net/wp-content/uploads/2017/01/movement-upsert-1.png)

**因此，我们应该力图只更新需要变更的字段**

比如说，我们需要更新location的状态。那么，我们只会更新status字段，而不是整个记录。这样就可以避免因为其它字段中有Null值而产生tombstone.

![avatar](http://www.beyondthelines.net/wp-content/uploads/2017/01/location-row-1.png)
*<center>The ‘properties’ field is not set in the query so no value is stored in Cassandra.</center>*

下面这个语句正式我们需要的：
```$xslt
UPDATE locations SET status = 'damaged' WHERE location_address = 'D3-DJ-21-B-02';
```
sstabledump输出结果如下：
```$xslt
[
  {
    "partition" : {
      "key" : [ "D3-DJ-21-B-02" ],
      "position" : 0
    },
    "rows" : [
      {
        "type" : "row",
        "position" : 28,
        "cells" : [
          { "name" : "status", "value" : "damaged", "tstamp" : "2017-01-28T11:55:18.146255Z" }
        ]
      }
    ]
  }
]
```
相反，下面的这个语句更新整个记录，它会产生tombstone,因为恰好properties字段没有值。
```$xslt
INSERT INTO locations (
  address,
  status,
  properties
) VALUES (
  'D3-DJ-21-B-02',
  'damaged',
  null
);
```
![avatar](http://www.beyondthelines.net/wp-content/uploads/2017/01/location-empty-collection-1.png)
*<center>An empty collection is stored as a tombstone cell
</center>*

```$xslt
[
  {
    "partition" : {
      "key" : [ "D3-DJ-21-B-02" ],
      "position" : 0
    },
    "rows" : [
      {
        "type" : "row",
        "position" : 18,
        "liveness_info" : { "tstamp" : "2017-01-28T11:58:59.160898Z" },
        "cells" : [
          { "name" : "status", "value" : "damaged" },
          { "name" : "properties", "deletion_info" : { "marked_deleted" : "2017-01-28T11:58:59.160897Z", "local_delete_time" : "2017-01-28T11:58:59Z" } }
        ]
      }
    ]
  }
]
```
## 注意集合类型
在前面的例子中，'properties'字段是一个集合类型（很有可能是个set)，下面我们来讨论一些集合类型，因为它比详细的难搞。

用下面的语句写入一个新的location:
```$xslt
INSERT INTO locations (
  address,
  status,
  properties
) VALUES (
  'C3-BE-52-C-01',
  'normal',
  {'pickable'}
);
```
看起来很完美，对吗？每个字段都有值，应该不会有tombstone生成。但事实上，properties会生成tombstone.
```$xslt
[
  {
    "partition" : {
      "key" : [ "C3-BE-52-C-01" ],
      "position" : 0
    },
    "rows" : [
      {
        "type" : "row",
        "position" : 18,
        "liveness_info" : { "tstamp" : "2017-01-28T12:01:00.256789Z" },
        "cells" : [
          { "name" : "status", "value" : "normal" },
          { "name" : "properties", "deletion_info" : { "marked_deleted" : "2017-01-28T12:01:00.256788Z", "local_delete_time" : "2017-01-28T12:01:00Z" } },
          { "name" : "properties", "path" : [ "pickable" ], "value" : "" }
        ]
      }
    ]
  }
]
```
为了一探究竟，让我们先看看Cassandra在底层是如何存储集合类型数据的。
![avatar](http://www.beyondthelines.net/wp-content/uploads/2017/01/location-collection-1.png)
*<center>The collection field includes a tombstone cell to empty the collection before adding a value.</center>*

Cassandra会往set中追加新的数据，所以当我们想要容器只存储我们设置的值就需要先删除任何之前存在的数据。这就是为什么Cassandra需要先写入一个tombstone,然后才是我们操作中设置的值。这样能保证set只会保存'pickable'

这是我们只更新需要更新字段的另一个理由。

## 小心物化视图
物化视图是Cassandra维护的一张表。它主要的特性是我们可以在一张基表上定义不同的主键。我们可以重新排序主键组里面的字段，也可以在主键组里新加入一个字段。

这个特性非常棒，它允许我们定义不同的partition 和 clustering 字段，但它同时带来一个问题：产生很多的tombstone. 让我们看一个例子来理解到底发生了什么。

假设我们想要根据status查询locations，例如我们想要查询所有'damaged' locations. 我们来创建一个物化视图来解释这个案例：
```aidl
CREATE MATERIALIZED VIEW locations_by_status AS
  SELECT
    status,
    address,
    properties
  FROM locations
  WHERE status IS NOT NULL
  AND address IS NOT NULL
  PRIMARY KEY (status, address);
```
非常好，这样我们可以根据status查询所有的locations.

但是，让我们来想一想当我们更新location的status会发生什么：
```aidl
UPDATE locations
SET status = 'damaged'
WHERE address = 'C3-BE-52-C-01';
```
前面我们看过在基表中更新其中的一个字段不会产生tombstone. 但是现在status是物化视图主键的一部分，当status变更时，partition 键也会跟着改变。
```aidl
[
  {
    "partition" : {
      "key" : [ "normal" ],
      "position" : 0
    },
    "rows" : [
      {
        "type" : "row",
        "position" : 18,
        "clustering" : [ "C3-BE-52-C-01" ],
        "deletion_info" : { "marked_deleted" : "2017-01-20T10:34:27.707604Z", "local_delete_time" : "2017-01-20T10:46:14Z" },
        "cells" : [ ]
      }
    ]
  },
  {
    "partition" : {
      "key" : [ "damaged" ],
      "position" : 31
    },
    "rows" : [
      {
        "type" : "row",
        "position" : 49,
        "clustering" : [ "C3-BE-52-C-01" ],
        "liveness_info" : { "tstamp" : "2017-01-20T10:46:14.285730Z" },
        "cells" : [
          { "name" : "properties", "deletion_info" : { "marked_deleted" : "2017-01-20T10:46:14.285729Z", "local_delete_time" : "2017-01-20T10:46:14Z" } }
        ]
      }
    ]
  }
]
```
为了保持物化视图和基表同步，Cassandra需要删除原来的记录，并在新的分区写入一条新纪录。删除就意味着tombstone。

![avatar](http://www.beyondthelines.net/wp-content/uploads/2017/01/materialized-view-tombstones-1.png)
*<center>The update in the base table triggers a partition change in the materialised view which creates a tombstone to remove the row from the old partition.</center>*

当我们设计一个物化视图的主键时要深入思考一下，尤其是物化视图主键比基表多一个字段时。也就是说，除非用物化视图是唯一的解决方案，并且由它带来的负面都在可接受范围内，除非不会采用这种设计。

此外，我们也要考虑主键字段值变化的频率。在上面的例子中，我们应该评估location status值变化的速率。 变化越不频繁，在tombstone方面对我们越有利。

最终，tombstone会消失，当compaction触发的时候。一般间隔是10天（由配置参数"gc_grace_seconds"决定）。如果tombstone过多，我们可以考虑将这个值设置小一点。

## 总结
如我们所见，tombstone不仅和删除操作有关，有时候会变得棘手。有很多其它的场景可能产生tombstone。

Tombstone不是我们极力要去避免的坏事情，它只是在"只追加"存储结构中一种删除数据的途径。

但是tombstone产生时会影响性能，因此当我们设计数据模型和操作时应该考虑这一点。

当遇到100,000条tombstone时，Java客户端会报错。

知道了这些，我们应该限制tombstone的产生，除非有必要的情况。我们应该判断tombstone是否会产生，是否会成为一个问题。如果会，那我们应该考虑调优'gc_grace_seconds'参数，使得compaction更频繁一些。

原文地址：[Understanding Cassandra tombstones](http://www.beyondthelines.net/databases/cassandra-tombstones/ "Understanding Cassandra tombstones")