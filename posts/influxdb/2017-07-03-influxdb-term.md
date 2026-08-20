---
title: "influxdb术语"
date: "2017-07-03T08:42:02.000Z"
tags:
  - "influxdb"
source_url: "https://gpdream.github.io/2017/07/03/influxdb/influxdb-term/"
recovery_source: "html"
---
# influxdb术语

以翻译官网的术语文档为主，顺带着介绍一点概念\
\
题外话。\
每个point都有一个key，这个key的组成形式类似于\
database, retention policy, measurement, tag sets, field name, timestamp

- aggreation\
  influxQL 的聚合函数，根据某个点返回聚合结果
- batch\
  批量操作，可以通过一个http交互提交批量请求到influxdb中，influxdb建议一次批量提交的量为5000-10000
- Continuous queries\
  直译为连续查询，可以定期自动查询。必须要使用group by才行。\
  举例：

  ```plain
  // 创建连续查询，内容为每隔1小时将查询到的内容insert into到average_passengers中
  CREATE CONTINUOUS QUERY "cq_basic" ON "transportation"
  BEGIN
    SELECT mean("passengers") INTO "average_passengers" FROM "bus_data" GROUP BY time(1h)
  END
  ```

那么两小时候，查询average_passengers表的内容就会为\

```plain
SELECT * FROM "average_passengers"
name: average_passengers
------------------------
time                   mean
2016-08-28T07:00:00Z   7
2016-08-28T08:00:00Z   13.75
```

- database\
  逻辑概念，包含user,Continous queries,Retention policies（后面讲）,time series data（后面讲）。
- duration\
  持续时间，该值确认retention policies在influxdb中能保存多久，超过该值的数据会自动删除。
- field\
  influxdb中的数据结构实际上是key value对，field是不会被索引的，基于field的查询都将会是全表扫描。于之对应的tag有一定的索引作用。
- field key\
  field kv中的key，field key是字符串，存储在metastore中
- field set\
  一个point钟所有field key,value的集合
- field value\
  field实际的值，值可以是 strings, floats, integers, or booleans，一个field value永远会关联一个timestamp
- function\
  InfluxQL aggregations, selectors, and transformations.聚合函数，选择器，转换等。有必要给个文档专门写function
- identifier\
  标识符，包括 continuous query names, database names, field keys, measurement names, retention policy names, subscription names, tag keys, and user names.
- line protocol\
  写一个Point到influxdb的协议
- Measurement\
  influxdb数据结构的一种，描述存储的数据和字段的关联。使用上相当于关系数据库的table。
- metastore\
  influxdb的元数据，包括user information, databases, retention policies, shard metadata, continuous queries, and subscriptions.
- node\
  一个独立的infulxdb进程，与之对应的是server，node和server组成集群。
- point\
  influxdb的一种数据结构,是一个时间序列里所有的字段的集合，等同于关系数据库里的一行数据。每个point都是不同的，根据serial和timestamp来确认唯一。\
  同一个point可以插入多个记录。因为point下面是同一个serial/timestamp的field集合，而field是KV结构的，因此会覆盖掉相同Key的值。举例：

  ```plain
  Old point: cpu_load,hostname=server02,az=us_west val_1=24.5,val_2=7 1234567890000000
  New point: cpu_load,hostname=server02,az=us_west val_1=5.24 1234567890000000
  \\查询
  > SELECT * FROM "cpu_load" WHERE time = 1234567890000000
  name: cpu_load
  --------------
  time                      az        hostname   val_1   val_2
  1970-01-15T06:56:07.89Z   us_west   server02   5.24    7
  ```

可以看到返回的结果是同字段覆盖，其他字段合并。

- points per second\
  每秒写入到influxdb的point数
- query\
  查询操作
- replication factor\
  retention policy的一个配置项，决定在influxdb中数据有多少个副本
- retention policy\
  终于到这货了，存储策略，简写RP，influxdb的一种数据结构。RP决定数据保存多久(duration)，有多少个副本(replication factor)，多少时间一个shard group(shard group duration).每个database的rp都不同,每个database至少有一个rp。rp中可以使用measurement和tag来定义一个series.
- schema\
  influxdb中数据是如何组织的，包括下面这些：\
  databases, retention policies, series, measurements, tag keys, tag values, and field keys
- selector\
  influxdb function的一种，从一个区域的Point中选择出一个point
- series\
  Measurement 和 tag set的为同一个serie.如 measurement_name,result=reject,status=done 表示一个 series。默认一个数据库中 Series 总数量不能超过 100w。同一个series下的数据在物理上会按照时间排序存储。具体的数据结构为:\
  type Series struct {\
   mu sync.RWMutex\
   Key string // series key\
   Tags map[string]string // tags\
   id uint64 // id\
   measurement *Measurement // measurement\
  }.
- series cardinality\
  直译series基数，这个翻译绝对有问题。举例一个database下面有一个measurement，measurement下面有2个tag。tag的值分别是email3个和status 2个，那么这个measurement下的series cardinality就有3*2.数据会把这6个不同的serial按照time顺序存储起来。

| email | status |
| --- | --- |
| 1@email.com | start |
| 1@email.com | stop |
| 2@mail.com | start |
| 2@emai.com | stop |
| 3@mail.com | start |
| 3@emai.com | stop |

- server\
  A machine, virtual or physical, that is running InfluxDB. There should only be one InfluxDB process per server.\
  啥意思，没看懂
- shard\
  数据分片，一个存储策略下会有多个分片，分片怎么折腾要看分片策略。例如按小时来分片，那就可能7点的数据都在一个分片，8点的数据都在一个分片这样。
- shard duration\
  分片存活时间
- shard group
- subscription
- tag\
  标签，官网说是相当于索引，感觉更像是数据目录。tag合集对应一个serial，tag还是比较重要的
- tag key
- tag value
- tag set
- timestamp
- transformation\
  转换，function的一种
- tsm (Time Structured Merge tree)
- user\
  有admin和非admin用户，admin用户对所有database有读写权限，非admin用户需要授权。
- wal\
  预写日志，wal日志合并使用tsm
