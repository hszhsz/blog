---
bg: craftsman.jpg
layout: post
title:  "[Cassandra]Cassandra如何存储数据"
crawlertitle: "[Cassandra]Cassandra如何存储数据"
summary: "[Cassandra]Cassandra如何存储数据"
date:   2019-04-12 09:04:47 +0800
categories: posts
tags: ['Cassandra','翻译']
author: 世农
---
*翻译文章，侵权秒删*

为了从Cassandra获得最优的性能，理解它如何在磁盘中存储数据是非常重要的。有RDBMS经验的Cassandra新用户在设计Column Families(表)的时候往往会忽略查询场景，这是一个常见的问题。Cassandra的CQL接口返回表化的数据会给我们一个错觉，认为我们可以像查询RDBMS那样查询Cassandra, 其实不是这样的。

所有的数据都存在数据目录的SSTable（Sorted String tables）中。默认的数据目录是：<code>$CASSANDRA_HOME/data/data</code>。你可以通过配置文件<code>$CASSANDRA_HOME/data/data</code>中的<code>data_file_directories/code>修改它。在一个新安装的Cassandra环境，下面是数据目录的样子：
```aidl
data/
├── system
├── system_auth
├── system_distributed
├── system_schema
└── system_traces
```
data目录下的每一个文件夹代表一个keyspace。上面是Cassandra内部使用的keyspace。

让我们创建一个新的keyspace:
```aidl
CREATE KEYSPACE ks1 WITH replication={'class':'SimpleStrategy','replication_factor':1};
```
由于我们还没有创建任何表，你在数据目录看不到ks1命名的文件夹，但是可以查询Cassandra的表：<code>system_schema.keyspace</code>
```aidl
cqlsh> select * from system_schema.keyspaces where keyspace_name = 'ks1';

 keyspace_name | durable_writes | replication
---------------+----------------+-------------------------------------------------------------------------------------
           ks1 |           True | {'class': 'org.apache.cassandra.locator.SimpleStrategy', 'replication_factor': '1'}
```
现在让我们创建一张表：
```aidl
CREATE TABLE user_tracking (
  user_id text,
  action_category text,
  action_id text,
  action_detail text,

  PRIMARY KEY(user_id, action_category, action_id)
);
```
一旦创建表，数据目录就会变成这样：
```aidl
ks1/
└── tb1-ed4784f0b64711e7b18a2f179b6f38f9
    └── backups
```
Cassandra根据<code><table>-<table_id></code>为每张表创建文件夹。表的所有数据都会在这个文件夹下，可以把这个文件夹移动到其它位置，然后创建一个软链接代替。

我们可以在system_schema.tables表中查看创建的表：
```aidl
cqlsh:ks1> select * from system_schema.tables where keyspace_name = 'ks1';

 keyspace_name | table_name | bloom_filter_fp_chance | caching                                       | comment | compaction                                                                                                                | compression                                                                             | crc_check_chance | dclocal_read_repair_chance | default_time_to_live | extensions | flags        | gc_grace_seconds | id                                   | max_index_interval | memtable_flush_period_in_ms | min_index_interval | read_repair_chance | speculative_retry
---------------+------------+------------------------+-----------------------------------------------+---------+---------------------------------------------------------------------------------------------------------------------------+-----------------------------------------------------------------------------------------+------------------+----------------------------+----------------------+------------+--------------+------------------+--------------------------------------+--------------------+-----------------------------+--------------------+--------------------+-------------------
           ks1 |        tb1 |                   0.01 | {'keys': 'ALL', 'rows_per_partition': 'NONE'} |         | {'class': 'org.apache.cassandra.db.compaction.SizeTieredCompactionStrategy', 'max_threshold': '32', 'min_threshold': '4'} | {'chunk_length_in_kb': '64', 'class': 'org.apache.cassandra.io.compress.LZ4Compressor'} |                1 |                        0.1 |                    0 |           {} | {'compound'} |           864000 | ed4784f0-b647-11e7-b18a-2f179b6f38f9 |               2048 |                           0 |                128 |                  0 |      99PERCENTILE

(1 rows)
```
你能注意到一些默认设置被启用，例如：<code>gc_grace_seconds</code>和<code>memtable_flush_period_in_ms</code>

让我们向这些表中写入一些数据：
```aidl
insert into ks1.user_tracking(user_id, action_category, action_id,  action_detail) VALUES ('user1', 'auth', 'a1',  'Logged in from home page');
```
现在让我们再来看数据目录。你会注意到没有任何新的文件被创建。那是因为Cassandra会先将数据写入到内存，当特定的阈值被触发时再刷到磁盘。为了持久化的考虑，数据同时会被写入"只追加"的commit log。 Commit log是被keyspace 共享的，它的默认路径是<code>$CASSANDRA_HOME/data/commitlog</code>,可以通过<code>conf/cassandra.yaml</code>修改。

以下命令可以强制将数据从mem-table刷到磁盘：
```aidl
bin/nodetool flush  //flush all tables from of all keyspace
bin/nodetool flush <keyspace>    //flush all tables for a single keyspace 
bin/nodetool flush <keyspace> <table_name>  //flush a single table from a keyspace
```
刷完数据后，下面是表对应的目录的样子。每一次刷数据都会在磁盘上产生一个SSTable，每个SSTable都会产生如下文件：
```aidl
user_tracking-49eb78d0b65a11e7b18a2f179b6f38f9/
├── backups
├── mc-5-big-CompressionInfo.db
├── mc-5-big-Data.db
├── mc-5-big-Digest.crc32
├── mc-5-big-Filter.db
├── mc-5-big-Index.db
├── mc-5-big-Statistics.db
├── mc-5-big-Summary.db
└── mc-5-big-TOC.txt

1 directory, 8 files
```
所有的文件名字都有共同的后缀：*.db
* *version* 是一个由字母组成的字符串，代表SSTable的版本，例如上面例子的 mc
* *generation* 是一个标号，每次新的SSTable创建时会加1，例如上面例子的 5
* *componet* 代表文件中的数据类型，例如上面例子中的：CompressionInfo, Data, Digest, Index 等。

下面提供每个组建大致的介绍：
```aidl
File Description
mc-1-big-TOC.txt  A file that lists the components for the given SSTable.
mc-1-big-Digest.crc32 A file that consists of a checksum of the data file.
mc-1-big-CompressionInfo.db   A file that contains meta data for the compression algorithm, if enabled.
mc-1-big-Statistics.db    A file that holds statistical metadata about the SSTable.
mc-1-big-Index.db A file that contains the primary index data.
mc-1-big-Summary.db   This file provides summary data of the primary index, e.g. index boundaries, and is supposed to be stored in memory.
mc-1-big-Filter.db    This file embraces a data structure used to validate if row data exists in memory i.e. to minimize the access of data on disk.
mc-1-big-Data.db  This file contains the base data itself. Note: All the other component files can be regenerated from the base data file.
```
让我们向表中写入更多的记录，看看数据目录有什么变化：
```aidl
insert into ks1.user_tracking(user_id, action_category, action_id,  action_detail) VALUES ('user1', 'auth', 'a1',  'Logged in from home page');

insert into ks1.user_tracking(user_id, action_category, action_id,  action_detail) VALUES ('user1', 'auth', 'a2', 'Logged in from email link');
insert into ks1.user_tracking(user_id, action_category, action_id,  action_detail) VALUES ('user1', 'dashboard', 'a3', 'Opened dashboard link');
insert into ks1.user_tracking(user_id, action_category, action_id,  action_detail) VALUES ('user2', 'auth', 'a4', 'Logged in');
```
再一次执行<code>bin/nodetool flush</code>将数据刷到SSTable，来看文件系统的样子：
```aidl
sam@sam-ub:ks1$ tree user_tracking-49eb78d0b65a11e7b18a2f179b6f38f9/
user_tracking-49eb78d0b65a11e7b18a2f179b6f38f9/
├── backups
├── mc-7-big-CompressionInfo.db
├── mc-7-big-Data.db
├── mc-7-big-Digest.crc32
├── mc-7-big-Filter.db
├── mc-7-big-Index.db
├── mc-7-big-Statistics.db
├── mc-7-big-Summary.db
├── mc-7-big-TOC.txt
├── mc-8-big-CompressionInfo.db
├── mc-8-big-Data.db
├── mc-8-big-Digest.crc32
├── mc-8-big-Filter.db
├── mc-8-big-Index.db
├── mc-8-big-Statistics.db
├── mc-8-big-Summary.db
└── mc-8-big-TOC.txt

1 directory, 16 files
```
你会注意到，一个新的generation 号码为7的SSTable被创建了。 Cassandra会定期通过一个叫compaction的进程合并SSTable。可以通过以下命令手工触发一次compaction:
```aidl
sam@sam-ub:ks1$ tree user_tracking-49eb78d0b65a11e7b18a2f179b6f38f9/
user_tracking-49eb78d0b65a11e7b18a2f179b6f38f9/
├── backups
├── mc-7-big-CompressionInfo.db
├── mc-7-big-Data.db
├── mc-7-big-Digest.crc32
├── mc-7-big-Filter.db
├── mc-7-big-Index.db
├── mc-7-big-Statistics.db
├── mc-7-big-Summary.db
├── mc-7-big-TOC.txt
├── mc-8-big-CompressionInfo.db
├── mc-8-big-Data.db
├── mc-8-big-Digest.crc32
├── mc-8-big-Filter.db
├── mc-8-big-Index.db
├── mc-8-big-Statistics.db
├── mc-8-big-Summary.db
└── mc-8-big-TOC.txt

1 directory, 16 files
```
你会注意到compaction之后，只有一个带着新的generation号码的SSTable存在。Compaction 会产生一个磁盘IO和使用率的临时峰值，因为有一段时间旧的SSTable和新的SSTable同时存在。当compaction结束时，老的SSTable占用的磁盘空间会被释放。

现在让我们看看数据到底是怎么在SSTable中存储的：
![avatar](https://saumitra.me/images/posts/cass1.png)

Cassandra安装包附带一个叫sstabledump的工具，可以查看SSTable的内容。以下是SSTable的样子：
```aidl
[
  {
    "partition" : {
      "key" : [ "user2" ],
      "position" : 0
    },
    "rows" : [
      {
        "type" : "row",
        "position" : 45,
        "clustering" : [ "auth", "a4" ],
        "liveness_info" : { "tstamp" : "2017-10-24T20:33:32.772370Z" },
        "cells" : [
          { "name" : "action_detail", "value" : "Logged in" }
        ]
      }
    ]
  },
  {
    "partition" : {
      "key" : [ "user1" ],
      "position" : 46
    },
    "rows" : [
      {
        "type" : "row",
        "position" : 104,
        "clustering" : [ "auth", "a1" ],
        "liveness_info" : { "tstamp" : "2017-10-24T20:33:32.074848Z" },
        "cells" : [
          { "name" : "action_detail", "value" : "Logged in from home page" }
        ]
      },
      {
        "type" : "row",
        "position" : 104,
        "clustering" : [ "auth", "a2" ],
        "liveness_info" : { "tstamp" : "2017-10-24T20:33:32.085959Z" },
        "cells" : [
          { "name" : "action_detail", "value" : "Logged in from email link" }
        ]
      },
      {
        "type" : "row",
        "position" : 145,
        "clustering" : [ "dashboard", "a3" ],
        "liveness_info" : { "tstamp" : "2017-10-24T20:33:32.099739Z" },
        "cells" : [
          { "name" : "action_detail", "value" : "Opened dashboard link" }
        ]
      }
    ]
  }
]
```
这里我们看到2"行"数据。别把这里的"行"和RDBMS的行搞混淆了，这里的"行"实际上是一个分区。SSTable里的行数是由主键决定的。

主键由两部分组成：分区键和分组键。

在我们的例子中，主键的第一部分,即:"user" 被用来分区，其它的部分：action_category和action_id被用来分组。

为了更好地理解这一点，让我们创建一张完全一样的表，只是分组键不一样：
```aidl
CREATE TABLE user_tracking_new (
  user_id text,
  action_category text,
  action_id text,
  action_detail text,

  PRIMARY KEY((user_id, action_category), action_id)   
);
```
在上面的定义中，分区键是：user_id + action_category,分组键是 action_id。让我们写入两条记录，观察它们是怎么在SSTable中分组的：
```aidl
insert into ks1.user_tracking_new(user_id, action_category, action_id,  action_detail) VALUES ('user1', 'auth', 'a1', 'Logged in from home page');
insert into ks1.user_tracking_new(user_id, action_category, action_id,  action_detail) VALUES ('user1', 'auth', 'a2', 'Logged in from email link');
insert into ks1.user_tracking_new(user_id, action_category, action_id,  action_detail) VALUES ('user1', 'dashboard', 'a3', 'Opened dashboard link');
insert into ks1.user_tracking_new(user_id, action_category, action_id,  action_detail) VALUES ('user2', 'auth', 'a4', 'Logged in');
```
```aidl
[
  {
    "partition" : {
      "key" : [ "user1", "dashboard" ],
      "position" : 0
    },
    "rows" : [
      {
        "type" : "row",
        "position" : 67,
        "clustering" : [ "a3" ],
        "liveness_info" : { "tstamp" : "2017-10-24T20:32:45.633901Z" },
        "cells" : [
          { "name" : "action_detail", "value" : "Opened dashboard link" }
        ]
      }
    ]
  },
  {
    "partition" : {
      "key" : [ "user2", "auth" ],
      "position" : 68
    },
    "rows" : [
      {
        "type" : "row",
        "position" : 118,
        "clustering" : [ "a4" ],
        "liveness_info" : { "tstamp" : "2017-10-24T20:32:45.648367Z" },
        "cells" : [
          { "name" : "action_detail", "value" : "Logged in" }
        ]
      }
    ]
  },
  {
    "partition" : {
      "key" : [ "user1", "auth" ],
      "position" : 119
    },
    "rows" : [
      {
        "type" : "row",
        "position" : 182,
        "clustering" : [ "a1" ],
        "liveness_info" : { "tstamp" : "2017-10-24T20:32:45.614746Z" },
        "cells" : [
          { "name" : "action_detail", "value" : "Logged in from home page" }
        ]
      },
      {
        "type" : "row",
        "position" : 182,
        "clustering" : [ "a2" ],
        "liveness_info" : { "tstamp" : "2017-10-24T20:32:45.624710Z" },
        "cells" : [
          { "name" : "action_detail", "value" : "Logged in from email link" }
        ]
      }
    ]
  }
]
```
在这个例子中你会注意到，对于每一个用户，每个category的数据都分到不同的分区中。

数据建模的第一个原则是：你应该选择你的分区键使得每次查询只读取一个分区的数据。

让我们跑一些查询看看分区键的影响：

```aidl
cqlsh:ks1> select * from user_tracking where user_id = 'user1';

 user_id | action_category | action_id | action_detail
---------+-----------------+-----------+---------------------------
   user1 |            auth |        a1 |  Logged in from home page
   user1 |            auth |        a2 | Logged in from email link
   user1 |       dashboard |        a3 |     Opened dashboard link

(3 rows)
```
查询新的表：
```aidl
cqlsh:ks1> select * from user_tracking_new where user_id = 'user2';
InvalidRequest: Error from server: code=2200 [Invalid query] message="Partition key parts: action_category must be restricted as other parts are"
```
看到上面错误的原因是，你应该在查询中指定分区键中的每一个字段。Cassandra在查询数据之前需要先知道查询数据的分区。第一个查询例子有效是因为表中只有user_id是分区键。

这就意味着，我们需要同时知道user_id和category才能从表user_tracking_new中查询数据。你可能会想为什么我们会更倾向于后面的这个表，因为第一张表的分区会变得很大，导致性能上的问题。我们的目标是保持分区在合理的大小范围，不会影响查询性能。

让我们试一下另一个查询：
```aidl
cqlsh:ks1> select * from user_tracking_new where user_id = 'user1' and action_category = 'auth' and action_detail = 'Logged in from home page';

InvalidRequest: Error from server: code=2200 [Invalid query] message="Cannot execute this query as it might involve data filtering and thus may have unpredictable performance. If you want to execute this query despite the performance unpredictability, use ALLOW FILTERING"

```
你会看到查询结果失败。原因是上面的查询跨越了两个分区，Cassandra发出警告。如果我们要强制查询，可以在查询后面加<code>ALLOW FILTERING</code>
```aidl
cqlsh:ks1> select * from user_tracking_new where user_id = 'user1' and action_category = 'auth' and action_detail = 'Logged in from home page' ALLOW FILTERING;

 user_id | action_category | action_id | action_detail
---------+-----------------+-----------+--------------------------
   user1 |            auth |        a1 | Logged in from home page

(1 rows)
```
## 分组排序
这是Cassandra 另外一个非常强大的特性，它允许我们根据特定的字段自然地按特定顺序存储数据。所以每次我们向表中写入数据时，Cassandra 知道向哪个物理分区以我们指定的顺序写入数据。按顺序存储数据极大地提高了范围查询性能，这对于时序数据非常有意义。
```aidl
CREATE TABLE user_tracking_ordered (
  user_id text,
  action_category text,
  action_id text,
  action_detail text,

  PRIMARY KEY((user_id, action_category), action_id)
) WITH CLUSTERING ORDER BY (action_id DESC); 
```
现在向表中写入数据：
```aidl
insert into ks1.user_tracking_ordered(user_id, action_category, action_id,  action_detail) VALUES ('user1', 'auth', 'a1', 'Logged in');
insert into ks1.user_tracking_ordered(user_id, action_category, action_id,  action_detail) VALUES ('user1', 'auth', 'a2', 'Logged in');
insert into ks1.user_tracking_ordered(user_id, action_category, action_id,  action_detail) VALUES ('user1', 'auth', 'a3', 'Logged in');
insert into ks1.user_tracking_ordered(user_id, action_category, action_id,  action_detail) VALUES ('user1', 'auth', 'a4', 'Logged in');
insert into ks1.user_tracking_ordered(user_id, action_category, action_id,  action_detail) VALUES ('user1', 'auth', 'a5', 'Logged in');
```
如果你查询一行数据，得到的数据是排序过的
```aidl
cqlsh:ks1> select * from user_tracking_ordered limit 1;

 user_id | action_category | action_id | action_detail
---------+-----------------+-----------+---------------
   user1 |            auth |        a1 |     Logged in

(1 rows)
```
如果你改变分组顺序，倒序:<code>WITH CLUSTERING ORDER BY (action_id DESC)</code>，用同样的查询，数据以倒序的顺序返回：
```aidl
cqlsh:ks1> select * from user_tracking_ordered limit 1;

 user_id | action_category | action_id | action_detail
---------+-----------------+-----------+---------------
   user1 |            auth |        a5 |     Logged in

(1 rows)
```
可以看到SSTable中的顺序也相应改变：
```aidl
$ sstabledump mc-*-big-Data.db

[
  {
    "partition" : {
      "key" : [ "user1", "auth" ],
      "position" : 0
    },
    "rows" : [
      {
        "type" : "row",
        "position" : 50,
        "clustering" : [ "a5" ],
        "liveness_info" : { "tstamp" : "2017-10-25T08:21:43.628921Z" },
        "cells" : [
          { "name" : "action_detail", "value" : "Logged in" }
        ]
      },
      {
        ...
        "clustering" : [ "a4" ],
        ...
      },
      {
        ...
        "clustering" : [ "a3" ],
        ...
      },
      {
        ...
        "clustering" : [ "a2" ],
        ...
      },
      {
          ...
        "clustering" : [ "a1" ],
        ...
      }
    ]
  }
]
```
你可以注意到，对于存储在Cassandra中的每个字段的数据，必须存储一些元数据。当做容量规划时应该考虑到这些开销。

我希望你通过本文理解为什么cqlsh看起来像SQL，但表现出来的却不是。用不同的组合键来看SSTable中的数据有什么不同，从而理解数据存储方式，这样有利于根据自己的查询需求来进行数据建模。

原文地址：[How Cassandra Stores Data on Filesystem](https://saumitra.me/blog/how-cassandra-stores-data-on-filesystem/ "How Cassandra Stores Data on Filesystem")