---
title: "greenplum的worlkoad详解"
date: "2017-01-04T05:28:51.000Z"
tags:
  - "greenplum"
source_url: "https://gpdream.github.io/2017/01/04/greenplum/gp%20workload/"
recovery_source: "html"
---
# greenplum的worlkoad详解

默认8192，内存资源限制。计算公式为（x*单台机器物理内存)/主segment数目。x为1到1.5之间。例如物理机126G内存，主segment16个。则该值为(1*126)/16=7.875GB.则该值设置为7875.\
gp_vmem_idle_resource_timeout ：默认 eager_free，有none, auto, eager_free三种。当设置为auto，查询内存使用被statement_mem和resource queue memory limits所限，设置为none,当设置为None，memory management与4.1版本GP相同。当设置为eager_free时，将会最大程度的使用内存，但不超过max_statement_mem和resourece queue的内存限制。

# 描述

主要起控制查询队列和防止系统过载的作用。\
当用户提交一个查询的时候，这个查询会被评估是否超过了资源队列的限制。如果这个查询没有超过资源限制，则会被立即执行。如果该查询超过了资源队列的限制，则查询必须等到资源队列里前面的查询执行结束有空闲才会进行。查询请求默认先进先出。如果启用了查询优先级的设置，超级管理员的。\
使Workload生效需要如下几步：

- 配置workload Managermnat的配置项（可使用默认配置）
- 创建资源队列，并设置资源队列的limit值
- 将用户添加到资源队列中
- 检查/监控资源队列状态

# WorkLoad配置项

## 资源队列通用配置项

- max_resource_queues ：最多有多少个资源队列
- max_resource_portals_per_transaction：每个事务最多能打开多少游标，每个打开的游标都会在其中资源队列中占用一个活动的查询槽。
- resource_select_only ：设置该值为on，则资源队列只对会触发select的语句有效，insert,alert等不生效
- resource_cleanup_gangs_on_wait :在占用资源对列中一个槽位前，先清理掉空闲的工作进程
- stats_queue_level ：收集资源队列的状态信息

  ## 内存相关设置
- gp_resqueue_memory_policy ：默认 eager_free，有none, auto, eager_free三种。当设置为auto，查询内存使用被statement_mem和resource queue memory limits所限，设置为none,当设置为None，memory management与4.1版本GP相同。当设置为eager_free时，将会最大程度的使用内存，但不超过max_statement_mem和resourece queue的内存限制。
- statement_mem 和 max_statement_mem ：指令占用的内存和指令占用的最大内存
- gp_vmem_protect_limit ：默认8192，内存资源限制。计算公式为（x*单台机器物理内存)/主segment数目。x为1到1.5之间。例如物理机126G内存，主segment16个。则该值为(1*126)/16=7.875GB.则该值设置为7875.当查询小号的内存大于该值的时候，查询语句会被取消。
- gp_vmem_idle_resource_timeout 和gp_vmem_protect_segworker_cache_limit：当连接超过该值时，服务器会释放掉其占用的资源，当cache超过limit时,内容不会被cache，而是在使用完后释放掉。
- shared_buffers ：共享缓冲区\
  ##workload优先级相关配置项
- gp_resqueue_priority ：使优先级生效，默认生效。
- gp_resqueue_priority_sweeper_interval：默认值1000ms，每隔这么久去竞争一次cpu
- gp_resqueue_priority_cpucores_per_segment ：每个segment使用的资源

# Workload是如何防止过载的

## 内存限制的工作方式

资源队列有自己的memory_limit，如果该资源队列最大内存限制为2000MB，而activity_statment为10，则该资源队列下的statement默认的占用内存为200MB.设置了statement_mem，可以覆盖默认的占用内存。用户可以通过statement_mem参数设置自己资源队列里的statement_mem，但该值受小于max_statement_mem和memory_limit，max_statement_mem只能由超级管理员设置。如果MEMORY_LIMIT 的资源被耗尽了，那么新进来的请求（哪怕activty_statement没满），请求也只能等待着。\
例如tpc语句中，队列数量为2，队列memroy_limit为1000MB,设置statement_mem为999MB，则在查询语句运行的时候，先后运行3行语句，在查询的时候会出现1行sql语句正在执行，其他语句正在等待。\

```plain
# 查看队列里语句执行状态
SELECT * FROM gp_toolkit.gp_locks_on_resqueue;
```

| lorusename | lorrsqname | lorlocktype | lorobjid | lortransaction | lorpid | lormode | lorgranted | lorwaiting |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| sj | myqueue | resource queue | 132213 | 14918 | 122473 | ExclusiveLock | f | t |
| sj | myqueue | resource queue | 132213 | 14917 | 122427 | ExclusiveLock | f | t |
| sj | myqueue | resource queue | 132213 | 14916 | 122381 | ExclusiveLock | t | f |

## 优先级的工作方式

优先级的设置，是用于抢占cpu而存在的，假设有若干个队列，优先级都不同。当队列里面进来新的查询时候，cpu资源会被重新分配，高优先级资源队列的请求会抢占掉低优先级资源队列里的cpu资源。假设现在有如下3个资源队列：

- adhoc 优先级低
- reporting 优先级中
- executive 优先级最高\
  那么当reporting中有两个查询时，两个查询占用的cpu资源可能是如下图左这样的，因为这两个sql的优先级相同。此时adhoc进来一个查询语句，会占用掉一部分的cpu资源，因为其优先级较低，故其占用cpu资源如下图右所示\
  ![image](http://gpdb.docs.pivotal.io/4380/graphics/gp_query_priority1.png)\
  此时如果再executive队列里进来一行sql语句，因为该队列占据最高优先级，则其将抢占走大量的cpu资源!\
  ![image](http://gpdb.docs.pivotal.io/4380/graphics/gp_query_priority2.png)

# 资源队列resource queue

资源队列只能由超级管理员来创建\

```plain
CREATE RESOURCE QUEUE name WITH (queue_attribute=value [, ... ])
--队列属性有
ACTIVE_STATEMENTS=integer
        [ MAX_COST=float [COST_OVERCOMMIT={TRUE|FALSE}] ]
        [ MIN_COST=float ]
        [ PRIORITY={MIN|LOW|MEDIUM|HIGH|MAX} ]
        [ MEMORY_LIMIT='memory_units' ]

        | MAX_COST=float [ COST_OVERCOMMIT={TRUE|FALSE}
        [ ACTIVE_STATEMENTS=integer ]
        [ MIN_COST=float ]
        [ PRIORITY={MIN|LOW|MEDIUM|HIGH|MAX} ]
        [ MEMORY_LIMIT='memory_units' ]
```

资源队列必须有属性active_statments或max_cost的一个或多个属性

## 资源对列各配置项参数含义

- name:资源队列的名字
- ACTIVE_STATEMENTS:同时运行的最大的指令数
- MAX_COST：指令预估的开销，如果超过该开销，且COST_OVERCOMMIT=FALSE，则不允许执行该指令
- MIN_COST：指令最小的开销，如果低于该值，则指令不计算在active_statements中，例如active_statements=2,而指令select * from table limit 1的开销为1，min_cost值为100，则该指令可直接运行不记录在ACTIVE_STATEMENTS中
- PRIORITY：需要打开优先级设置，设置队列的优先级
- MEMORY_LIMIT：队列能使用的总内存大小，建议为segment的90%以下。
- COST_OVERCOMMIT：cost超过最大值时是否还能允许指令运行。\
  -

## 用户和资源队列

用户提交的指令会出现在对应的资源队列中\
创建用户时候不指定资源队列，会把用户指向默认的资源队列。创建用户命令如下\

```plain
CREATE ROLE name [[WITH] option [ ... ]]
# 例句
CREATE RESOURCE QUEUE executive WITH (ACTIVE_STATEMENTS=3, PRIORITY=MAX);
option属性
 SUPERUSER | NOSUPERUSER
    | CREATEDB | NOCREATEDB
    | CREATEROLE | NOCREATEROLE
    | CREATEEXTTABLE | NOCREATEEXTTABLE
      [ ( attribute='value'[, ...] ) ]
           where attributes and value are:
           type='readable'|'writable'
           protocol='gpfdist'|'http'
    | INHERIT | NOINHERIT
    | LOGIN | NOLOGIN
    | CONNECTION LIMIT connlimit
    | [ ENCRYPTED | UNENCRYPTED ] PASSWORD 'password'
    | VALID UNTIL 'timestamp'
    | IN ROLE rolename [, ...]
    | ROLE rolename [, ...]
    | ADMIN rolename [, ...]
    | RESOURCE QUEUE queue_name
    | [ DENY deny_point ]
    | [ DENY BETWEEN deny_point AND deny_point]
```

可以通过修改用户的资源队列来让用户使用其他资源队列

```plain
ALTER ROLE name RESOURCE QUEUE {queue_name | NONE}
```

## 相关语法

```plain
# 创建资源队列
CREATE RESOURCE QUEUE myqueue WITH (ACTIVE_STATEMENTS=20,
MEMORY_LIMIT='2000MB');
# 设置资源队列优先级
ALTER RESOURCE QUEUE adhoc WITH (PRIORITY=LOW);
# 添加用于到资源队列中，
ALTER ROLE name RESOURCE QUEUE queue_name;
# 创建用户时指定资源队列，未指定的将加入default_queue
CREATE ROLE name WITH LOGIN RESOURCE QUEUE queue_name;
# 将用户移出资源队列
ALTER ROLE role_name RESOURCE QUEUE none;
# 删除资源队列
DROP RESOURCE QUEUE name;
```

# 资源队列监控

```plain
# 查看资源队列状态
 SELECT * FROM gp_toolkit.gp_resqueue_status;
# 视图查看资源队列的统计信息，可以在系统视图pg_stat_resqueues 中看到
stats_queue_level = on
# 查看角色和资源队列关系
 SELECT rolname, rsqname FROM pg_roles,
          gp_toolkit.gp_resqueue_status
   WHERE pg_roles.rolresqueue=gp_toolkit.gp_resqueue_status.queueid;
# 查看处于等待状态的指令
SELECT * FROM gp_toolkit.gp_locks_on_resqueue WHERE lorwaiting='true';
# 清除掉处于等待状态的statement,先查出对应的pid，再运行pg_cancel_backend取消其运行
SELECT rolname, rsqname, pid, granted,
          current_query, datname
   FROM pg_roles, gp_toolkit.gp_resqueue_status, pg_locks,
        pg_stat_activity
   WHERE pg_roles.rolresqueue=pg_locks.objid
   AND pg_locks.objid=gp_toolkit.gp_resqueue_status.queueid
   AND pg_stat_activity.procpid=pg_locks.pid;
   AND pg_stat_activity.usename=pg_roles.rolname;

 pg_cancel_backend(pid)
```
