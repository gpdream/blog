---
title: "由于gpstop -M immediate导致gp reciverseg失败"
date: "2017-01-04T05:28:51.000Z"
tags:
  - "greenplum"
source_url: "https://gpdream.github.io/2017/01/04/greenplum/GP%20recoverseg/"
recovery_source: "html"
---
# 由于gpstop -M immediate导致gp reciverseg失败

## 现象

### 操作

有个主segment挂了，运行了gprecoverseg，在gprecoverseg进程还未结束的时候运行gpstop -M immediate.然后运行gpstart -a启动集群出现问题。

### 现象

gpstart -a 集群启动成功，但带告警\

```plain
20161031:09:56:39:023724 gpstart:mdw1:gpadmin-[INFO]:-----------------------------------------------------
20161031:09:56:39:023724 gpstart:mdw1:gpadmin-[INFO]:-   Successful segment starts                                            = 4
20161031:09:56:39:023724 gpstart:mdw1:gpadmin-[INFO]:-   Failed segment starts                                                = 0
20161031:09:56:39:023724 gpstart:mdw1:gpadmin-[INFO]:-   Skipped segment starts (segments are marked down in configuration)   = 0
20161031:09:56:39:023724 gpstart:mdw1:gpadmin-[INFO]:-----------------------------------------------------
20161031:09:56:39:023724 gpstart:mdw1:gpadmin-[INFO]:-
20161031:09:56:39:023724 gpstart:mdw1:gpadmin-[INFO]:-Successfully started 4 of 4 segment instances
20161031:09:56:39:023724 gpstart:mdw1:gpadmin-[INFO]:-----------------------------------------------------
20161031:09:56:39:023724 gpstart:mdw1:gpadmin-[INFO]:-Starting Master instance mdw1.com directory /home/gpadmin/data/master/gpseg-1
20161031:09:56:40:023724 gpstart:mdw1:gpadmin-[INFO]:-Command pg_ctl reports Master mdw1.com instance active
20161031:09:57:05:023724 gpstart:mdw1:gpadmin-[WARNING]:-FATAL:  DTM initialization: failure during startup recovery, retry failed, check segment status (cdbtm.c:1602)
```

此时集群处于一个无法连接的状态，但通过`gpstate`查看集群状态，集群正常无告警，但有2个segment处于resync状态，这两个segment对应的正是原先主备挂掉了的segment。

### 原因定位

通过命令行工具模式直接psql连接`PGOPTIONS='-c gp_session_role=utility' psql -h192.168.124.32 -p 50000`，连接其他segment正常，但当连接到原先准备做gprecoverseg的那个mirror失败，提示Database is Starting。这个segment没有正常启动。进入这个segment，查看下面的日志，发现有如下内容

```plain
2016-10-31 09:56:38.668411 CST,,,p4738,th-1827370944,,,,0,,,seg-1,,,,,"LOG","00000","'run resync worker', mirroring role 'primary role' mirroring state 'resync' segment state 'transition to resync' filerep state 'up and running' process name(pid) 'primary resync worker #2 process(4738)' 'cdbfilerepresyncworker.c' 'L114' 'FileRepPrimary_RunResyncWorker'",,,,,,,0,,"cdbfilerep.c",1837,
2016-10-31 09:56:38.668460 CST,,,p4736,th-1827370944,,,,0,,,seg-1,,,,,"LOG","00000","'run resync transition, record last lsn in change tracking', mirroring role 'primary role' mirroring state 'resync' segment state 'transition to resync' filerep state 'up and running' process name(pid) 'primary resync manager process(4736)' 'cdbfilerepresyncmanager.c' 'L843' 'FileRepResyncManager_InResyncTransition'",,,,,,,0,,"cdbfilerep.c",1837,
2016-10-31 09:56:38.668492 CST,,,p4736,th-1827370944,,,,0,,,seg-1,,,,,"LOG","00000","'full resync 'false' resync lsn '0/4F44B30(0/83118896)' ', mirroring role 'primary role' mirroring state 'resync' segment state 'transition to resync' filerep state 'up and running' process name(pid) 'primary resync manager process(4736)' 'cdbfilerepresyncmanager.c' 'L350' 'FileRepResyncManager_SetEndResyncLSN'",,,,,,,0,,"cdbfilerep.c",1837,
2016-10-31 09:56:38.668519 CST,,,p4736,th-1827370944,,,,0,,,seg-1,,,,,"LOG","00000","'setting resync lsn ', mirroring role 'primary role' mirroring state 'resync' segment state 'transition to resync' filerep state 'up and running' process name(pid) 'primary resync manager process(4736)' 'cdbresynchronizechangetracking.c' 'L2417' 'ChangeTracking_RecordLastChangeTrackedLoc'",,,,,,,0,,"cdbfilerep.c",1837,
2016-10-31 09:56:38.739675 CST,,,p4736,th-1827370944,,,,0,,,seg-1,,,,,"LOG","00000","'no ct records to flush count '0' size '0' max '32768' offset -1 fileseg '0' ', mirroring role 'primary role' mirroring state 'resync' segment state 'transition to resync' filerep state 'up and running' process name(pid) 'primary resync manager process(4736)' 'cdbresynchronizechangetracking.c' 'L2456' 'ChangeTracking_RecordLastChangeTrackedLoc'",,,,,,,0,,"cdbfilerep.c",1837,
2016-10-31 09:56:38.743869 CST,,,p4736,th-1827370944,,,,0,,,seg-1,,,,,"LOG","00000","'run resync transition, mark scan incremental', mirroring role 'primary role' mirroring state 'resync' segment state 'transition to resync' filerep state 'up and running' process name(pid) 'primary resync manager process(4736)' 'cdbfilerepresyncmanager.c' 'L878' 'FileRepResyncManager_InResyncTransition'",,,,,,,0,,"cdbfilerep.c",1837,
2016-10-31 09:56:38.744573 CST,,,p4736,th-1827370944,,,,0,,,seg-1,,,,,"LOG","00000","'run resync transition, mark page incremental', mirroring role 'primary role' mirroring state 'resync' segment state 'transition to resync' filerep state 'up and running' process name(pid) 'primary resync manager process(4736)' 'cdbfilerepresyncmanager.c' 'L887' 'FileRepResyncManager_InResyncTransition'",,,,,,,0,,"cdbfilerep.c",1837,
2016-10-31 09:56:38.744633 CST,,,p4736,th-1827370944,,,,0,,,seg-1,,,,,"LOG","00000","'running a full round of compacting the logs ', mirroring role 'primary role' mirroring state 'resync' segment state 'transition to resync' filerep state 'up and running' process name(pid) 'primary resync manager process(4736)' 'cdbresynchronizechangetracking.c' 'L3093' 'ChangeTracking_DoFullCompactingRound'",,,,,,,0,,"cdbfilerep.c",1837,
2016-10-31 09:56:38.794314 CST,,,p4736,th0,,,,0,,cmd1,seg-1,,,,,"PANIC","XX000","Unexpected internal error: Segment process received signal SIGSEGV",,,,,,,0,,,,"1    0x944ea3 postgres StandardHandlerForSigillSigsegvSigbus_OnMainThread + 0x163
2    0x7fda90ae4100 libpthread.so.0 <symbol not found> + 0x90ae4100
3    0x4cabd8 postgres heap_compute_data_size + 0x68
4    0x50ecba postgres gp_changetracking_log + 0x10a
5    0x690b51 postgres ExecMakeTableFunctionResult + 0x251
6    0x6ac897 postgres <symbol not found> + 0x6ac897
```

启动进程是crash掉了的，虽然主进程postgres在，但该segment的启动还是crash掉了，日志里可以看出他在尝试再次执行增量的sync。再联想之前的操作，是在继续前面未完成的gprecoverseg。但是该动作被gpstop -M immediate打断了。看看immediate的帮助信息

`immediate shut down. Any transactions in progress are aborted. This
shutdown mode is not recommended, and in some circumstances can cause
database corruption requiring manual recovery.`

immediate强制中断了gprecovreseg的事务，当segment再次起来尝试执行增量recoverseg的时候需要借助segment下的pg_changetracking目录里的文件，但文件内容并不完整，导致segment的增量resync失败。\
 将该文件的内容移走`mv pg_changetracking/* /tmp/pg_changetracking`.

重启集群`gpstop -ar`集群恢复正常

*强制停止集群的时候最好用`gpstop -M fast`来代替`gpstop -M immediate`fast 会中断事务并尝试让事务回滚，虽然有的事务回滚会比较慢，但相比immediate的粗暴，并不需要做太多环境恢复的事情。*
