---
title: "阿里云tableStore介绍"
date: "2017-03-17T05:28:51.000Z"
tags:
  - "aliyun"
source_url: "https://gpdream.github.io/2017/03/17/aliyun/tableStore/"
recovery_source: "html"
---
# 阿里云tableStore介绍

Table Store前身是阿里云的OTS\
贴一段阿里云上Table Store（原OTS）的产品概述

表格存储（Table Store）是构建在阿里云飞天分布式系统之上的 NoSQL 数据存储服务，提供海量结构化数据的存储和实时访问。

1. 表格存储以实例和表的形式组织数据，通过数据分片和负载均衡技术，达到规模的无缝扩展。
2. 表格存储向应用程序屏蔽底层硬件平台的故障和错误，能自动从各类错误中快速恢复，提供非常高的服务可用性。
3. 表格存储管理的数据全部存储在 SSD 中并具有多个备份，提供了快速的访问性能和极高的数据可靠性。
4. 用户在使用表格存储服务时，只需要按照预留和使用的资源进行付费，无需关心数据库的软硬件升级维护、集群缩容扩容等复杂问题。

   ## 简介

   Table Store前身是阿里云的OTS\
   贴一段阿里云上Table Store（原OTS）的产品概述

表格存储（Table Store）是构建在阿里云飞天分布式系统之上的 NoSQL 数据存储服务，提供海量结构化数据的存储和实时访问。

1. 表格存储以实例和表的形式组织数据，通过数据分片和负载均衡技术，达到规模的无缝扩展。
2. 表格存储向应用程序屏蔽底层硬件平台的故障和错误，能自动从各类错误中快速恢复，提供非常高的服务可用性。
3. 表格存储管理的数据全部存储在 SSD 中并具有多个备份，提供了快速的访问性能和极高的数据可靠性。
4. 用户在使用表格存储服务时，只需要按照预留和使用的资源进行付费，无需关心数据库的软硬件升级维护、集群缩容扩容等复杂问题。

### 简介解读

Table Store是原OTS，OTS是据说是根据谷歌论文的BigTable自己实现的。hadoop生态圈中对应bigTable的是hbase。而OTS完美兼容hbase .解读ots的时候和hbase的做一些简单对比。

1. 简介1解读\
   简介1的重点是数据模型和数据自动分片，自动负载均衡，无缝扩展。\
   表格存储的数据模型概念包括表、行、主键和属性，如下图所示(图片来自阿里云TableStore)。\
   ![image](http://docs-aliyun.cn-hangzhou.oss.aliyun-inc.com/assets/pic/27281/cn_zh/1470029774660/DataModel.jpg)\
   如果有对应数据的话，那数据展示就会如下所示

| ID | Type | ISBN |
| --- | --- | --- |
| 4476 | Book | NULL |
| 65555 | NULL | Music |

这个模型完美的说明了简介1.这是一个很常见的K/V结构,主键以外任意的字段可以为空。这种形式和hbase的数据模型是完全相同的。能自动根据id来做分区达成自动数据分片和负载均衡。和hbase的Resgion会自动分片数据，Resgion自动增长切割迁移也是一样。\
差异在于，主键有4个字段，而且每一列都包装好了数值类型，在操作的时候会更方便。

1. 简介2解读\
   简介2是服务高可用。\
   ots提供的服务如果和hbase一样，hbase提供服务的region server，会在出现故障的时候自动迁移到集群的其他机器中。
2. 简介3解读\
   简介3是数据高可靠，多备份，高速访问。TableStore的数据存储在盘古上，自然解决了这些问题，用的还是SSD的盘古。
3. 简介4解读\
   一些维护啥的就不用关心了，给钱我帮你做。

## 使用方法

DataStore有http api和阿里云的DataStore的sdk两种。

### http api

他们的http api看上去比hbase rest api要强大。支持大部分的功能，但是为了尽可能多的支持功能，他们的api里只能用post方法请求。

### dataStroe sdk的api

sdk有aliyun_tablestore_java_sdk和TableStore HBase Client。使用TableStore HBase Client 支持 1.x.x 版本以上的开源 HBase API。无休修改代码就可将hbase上的业务迁移过来。

## TableStore和hbase的区别

TableStore的官网介绍里有文章[表格存储和 HBase 的区别](https://help.aliyun.com/document_detail/50220.html?spm=5176.doc27299.6.586.2QpxJ8) 写的是他们和hbase的差异。支持表示TableStore有的而hbase没有的。不支持表示hbase有的TableStore没有的。

### Table

Table

不支持多列族，只支持单列族。

#### 解读

hbase的多列族比较鸡肋，给干掉了。

### Row和Cell

不支持设置 ACL\
不支持设置 Cell Visibility\
不支持设置 Tag

#### 解读

DataStore做了权限，而hbase没有。cell可见和tag，也是方便拿来做权限的。

### GET

表格存储只支持单列族，所以不支持列族相关的接口，包括：\
setColumnFamilyTimeRange(byte[] cf, long minStamp, long maxStamp)\
setMaxResultsPerColumnFamily(int limit)\
setRowOffsetPerColumnFamily(int offset)

#### 解读

还是多列族单列族的事情。

### SCAN

类似于 GET，既不支持列族相关的接口，也不能设置优化类的部分接口，包括：

setBatch(int batch)\
setMaxResultSize(long maxResultSize)\
setAllowPartialResults(boolean allowPartialResults)\
setLoadColumnFamiliesOnDemand(boolean value)\
setSmall(boolean small)

####\
还是列族的事情

### Batch

暂时不支持 BatchCallback。

#### 解读

没有对应hbase的batchCallback接口。

### Mutations 和 Deletions

不支持删除特定列族\
不支持删除最新时间戳的版本\
不支持删除小于某个时间戳的所有版本

#### 解读

我去，这个都不支持？

### Increment 和 Append

暂时不支持

### Filter

支持 ColumnPaginationFilter\
支持 FilterList\
部分支持 SingleColumnValueFilter，比较器仅支持 BinaryComparator\
其他 Filter 暂时都不支持

#### 解读

我要这filter又有何用，我要这过滤器又如何？这支持的几个都是hbase后面版本也支持的。

### Optimization

HBase 的部分接口涉及到访问、存储优化等，这些接口目前没有开放：

blockcache：默认为 true，不允许用户更改\
blocksize：默认为 64K，不允许用户更改\
IsolationLevel：默认为 READ_COMMITTED，不允许用户更改\
Consistency：默认为 STRONG，不允许用户更改\
Admin

#### 解读

我曹，解读不下去了，下面还有一堆的区别对比。你丫写个区别对比全是自己和hbase相比的缺点，要你何用。

## 总结

OTS参考的hbase版本太早了，开源社区更新太快，导致hbase有很多特性在ots里面不支持。阿里云现在也上了hbase，和DataStore是强竞争关系。
