---
title: "由于缓存文件存在导致gpinitsystem失败"
date: "2017-01-04T05:28:51.000Z"
tags:
  - "greenplum"
source_url: "https://gpdream.github.io/2017/01/04/greenplum/gpinitsystem%20faild%20by%20cache%20file/"
recovery_source: "html"
---
# 由于缓存文件存在导致gpinitsystem失败

GP集群：master, segment1, segment2

问题描述：\
segment1和segment2上各配置8个primary segment，8个mirror segment，集群初始化，gpinitsystem过程中遇到了问题,集群初始化失败。

在集群初始化过程中，segment实例目录已按照预期分别在segment1和segment2上建立，在后续的gpstart启动集群时，有些segment找不到DBID启动不了，输出提示集群为初始化完全。\
\
初步分析：\
通过查日志发现，在gpstart启动集群时，总是从segment1上去找所有segment实例的DBID，然而有一半的segment实例是在segment2上的，segment1上当然是找不到这些segment实例，另一方面segment1上的mirror segment是segment2上primary segment的镜像，这些mirror segment因为找不到primary也无法启动，只有segment1上的primary能够启动。由于gpinitsystem过程中不能将所有segment拉起，集群初始化失败。

以下是问题定位及解决步骤：\
1、阅读gpdb/gpMgmt/bin/gpstart.py

> 在函数_prepare_segment_start()中确定要启动的segment,是通过gparray.get_valid_segdbs()来获得的

2、阅读gpdb/gpMgmt/bin/gppylib/gparray.py

> gparray执行initFromCatalog()初始化，是通执行SQL语句，根据pg_catalog中查询得到的信进行息初始化的，segment相关信息通过执行如下SQL查询：\
>
> ```plain
> SELECT dbid, content, role, preferred_role, mode, status,
> hostname, address, port, replication_port, fs.oid, fselocation
> FROM pg_catalog.gp_segment_configuration
> JOIN pg_catalog.pg_filespace_entry on (dbid = fsedbid)
> JOIN pg_catalog.pg_filespace fs on (fsefsoid = fs.oid)
> ORDER BY content, preferred_role DESC, fs.oid
> ```

于是，gpstart启动集群（只有1/4的segment实例能启动,即segment1上的primary能启动），psql template1，执行上述SQL，得到下图结果：\
![](https://gpdream.github.io/2017/01/04/greenplum/gpinitsystem%20faild%20by%20cache%20file/attachment/segment_result.png)\
果然，所有segment的hostname都为segment1,从而导致gpstart启动时都从segment1上找segment实例。于是想到，通过修改该表能否修复问题。

3、修改pg_catalog.gp_segment_configuration （该方法无法解决问题）

> 由于pg_catalog是系统表，是在集群初始化过程中生成的，并在运行过程中由系统自动维护，因此不能够直接修改，需要进入维护模式才能修改。具体步骤如下：

```plain
# 停止数据库
gpstop

# 以维护模式启动
gpstart -m

# 使用utility模式连接数据库
PGOPTIONS='-c gp_session_role=utility' psql template1

# 在psql中执行修改语句
update pg_catalog.gp_segment_configuration set hostname='segment2' where dbid=2 or ...

# 退出psql

# 停止维护模式
gpstop -m

# 启动生成模式
gpstart
```

通过上述步骤后，启动过程中提示hostname没有segment2,该方法无法有效解决问题。

4、阅读gpdb/gpMgmt/bin/gpinitsystem

> 由于pg_catalog表最初是在集群初始化过程中生成的，于是阅读了一下gpinitsystem的源码，是shell脚本，脚本中使用函数HOST_LOOKUP()查找hostname，该函数将hostaddress作为输入，调用gphostcache.py

5、阅读gpdb/gpMgmt/bin/gppylib/gphostcache.py

> 该脚本将interface与hostname的映射记录在~/.gphostcache文件中，如果有新的interface加进来会更新.gphostcache，具体代码逻辑没仔细看，对该文件的更新策略不是很清楚，也还没弄明白除了gpinitsystem、gpstart外，哪些操作过程需要查该文件。

6、修改.gphostcache

> 打开.gphostcache(存放在/home/gpadmin/目录下)，内容如下：\
> master:master\
> segment1:segment1
>
> 确实缺少segment2的映射，修改该文件，添加segment2:segment2，然后删除原有集群并重新gpinitsystem初始化集群，初始化成功。至此，初始化问题解决。

导致该问题的原因估计是第一次初始化集群时，主机名配置有误，而.gphostcache文件没有被重写可能是更新策略导致（更新策略待研究）。总的来说，为了避免安装、初始化中出现各种怪异问题，主要还是得按步骤配置好系统及各项参数，例如主机名、网络规划、host、ssh、硬盘挂载、参数设置等。
