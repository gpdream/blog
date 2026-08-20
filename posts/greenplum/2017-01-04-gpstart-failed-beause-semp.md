---
title: "信号量问题导致gp启动失败"
date: "2017-01-04T05:28:51.000Z"
tags:
  - "greenplum"
source_url: "https://gpdream.github.io/2017/01/04/greenplum/gpstart%20failed%20beause%20semp/"
recovery_source: "html"
---
# 信号量问题导致gp启动失败

[toc]

## 现象

在执行gpinitsystem的时候，最后执行失败，返回错误日志如下:

> 20160910:21:51:01:017775 gpstart:lin:gpadmin-[INFO]:-Process results…\
> 20160910:21:51:01:017775 gpstart:lin:gpadmin-[ERROR]:-No segment started for content: 1.\
> 20160910:21:51:01:017775 gpstart:lin:gpadmin-[INFO]:-dumping success segments: [‘lin.g3.s1:/gpadmin/data/mirror/gpseg3:content=3:dbid=9:mode=s:status=u’]\
> 20160910:21:51:01:017775 gpstart:lin:gpadmin-[INFO]:—————————————————–\
> 20160910:21:51:01:017775 gpstart:lin:gpadmin-[INFO]:-DBID:4 FAILED host:’lin.g3.s2’ datadir:’/gpadmin/data/primary/gpseg2’ with reason:’PG_CTL failed.’\
> 20160910:21:51:01:017775 gpstart:lin:gpadmin-[INFO]:-DBID:5 FAILED host:’lin.g3.s2’ datadir:’/gpadmin/data/primary/gpseg3’ with reason:’PG_CTL failed.’\
> 20160910:21:51:01:017775 gpstart:lin:gpadmin-[INFO]:-DBID:7 FAILED host:’lin.g3.s2’ datadir:’/gpadmin/data/mirror/gpseg1’ with reason:’PG_CTL failed.’\
> 20160910:21:51:01:017775 gpstart:lin:gpadmin-[INFO]:-DBID:6 FAILED host:’lin.g3.s2’ datadir:’/gpadmin/data/mirror/gpseg0’ with reason:’PG_CTL failed.’\
> 20160910:21:51:01:017775 gpstart:lin:gpadmin-[INFO]:-DBID:8 FAILED host:’lin.g3.s1’ datadir:’/gpadmin/data/mirror/gpseg2’ with reason:’PG_CTL failed.’\
> 20160910:21:51:01:017775 gpstart:lin:gpadmin-[INFO]:-DBID:3 FAILED host:’lin.g3.s1’ datadir:’/gpadmin/data/primary/gpseg1’ with reason:’PG_CTL failed.’\
> 20160910:21:51:01:017775 gpstart:lin:gpadmin-[INFO]:-DBID:2 FAILED host:’lin.g3.s1’ datadir:’/gpadmin/data/primary/gpseg0’ with reason:’Failure in segment mirroring; check segment logfile’\
> 20160910:21:51:01:017775 gpstart:lin:gpadmin-[INFO]:—————————————————–
>
> 20160910:21:51:01:017775 gpstart:lin:gpadmin-[INFO]:—————————————————–\
> 20160910:21:51:01:017775 gpstart:lin:gpadmin-[INFO]:- Successful segment starts = 1\
> 20160910:21:51:01:017775 gpstart:lin:gpadmin-[WARNING]:-Failed segment starts, from mirroring connection between primary and mirror = 1 <<<<<<<<\
> 20160910:21:51:01:017775 gpstart:lin:gpadmin-[WARNING]:-Other failed segment starts = 6 <<<<<<<<\
> 20160910:21:51:01:017775 gpstart:lin:gpadmin-[INFO]:- Skipped segment starts (segments are marked down in configuration) = 0\
> 20160910:21:51:01:017775 gpstart:lin:gpadmin-[INFO]:—————————————————–\
> 20160910:21:51:01:017775 gpstart:lin:gpadmin-[INFO]:-\
> 20160910:21:51:01:017775 gpstart:lin:gpadmin-[INFO]:-Successfully started 1 of 8 segment instances <<<<<<<<\
> 20160910:21:51:01:017775 gpstart:lin:gpadmin-[INFO]:—————————————————–\
> 20160910:21:51:01:017775 gpstart:lin:gpadmin-[WARNING]:-Segment instance startup failures reported\
> 20160910:21:51:01:017775 gpstart:lin:gpadmin-[WARNING]:-Failed start 7 of 8 segment instances <<<<<<<<\
> 20160910:21:51:01:017775 gpstart:lin:gpadmin-[WARNING]:-Review /home/gpadmin/gpAdminLogs/gpstart_20160910.log\
> 20160910:21:51:01:017775 gpstart:lin:gpadmin-[INFO]:—————————————————–\
> 20160910:21:51:01:017775 gpstart:lin:gpadmin-[INFO]:-Commencing parallel segment instance shutdown, please wait…\
> ..
>
> 20160910:21:51:05:017775 gpstart:lin:gpadmin-[ERROR]:-gpstart error: Do not have enough valid segments to start the array.\
>
> ## 问题定位
>
> 按日志提示，initsystem应该已经成功了，只是在做init之后的start 操作失败了。于是我们运行`gpstart -m`只启动master，再运行`gpstop -a -M fast`来保证所有的segment都停止运行。再通过命令`gpstart -a -v`重新启动gp集群，-v参数会打印出详细日志。\
> 结果肯定是和刚才一样肯定失败的。但是我们获取到详细失败日志，是由于该启动的segment未启动成功，导致gp认为启动失败，并停止掉了整个集群。日志内容如下\
> 20160910:21:58:07:013425 gpsegstart.py_lin:gpadmin:lin:gpadmin-[DEBUG]:-[worker2] got cmd: env GPSESSID=0000000000 GPERA=a0ceea5600521f9c_160910215805 $GPHOME/bin/pg_ctl -D /gpadmin/data/mirror/gpseg2 -l /gpadmin/data/mirror/gpseg2/pg_log/startup.log -w -t 600 -o “ -p 50000 -b 8 -z 4 –silent-mode=true -i -M quiescent -C 2 “ start 2>&1\
> 20160910:21:58:07:013425 gpsegstart.py_lin:gpadmin:lin:gpadmin-[DEBUG]:-[worker3] got cmd: env GPSESSID=0000000000 GPERA=a0ceea5600521f9c_160910215805 $GPHOME/bin/pg_ctl -D /gpadmin/data/mirror/gpseg3 -l /gpadmin/data/mirror/gpseg3/pg_log/startup.log -w -t 600 -o “ -p 50001 -b 9 -z 4 –silent-mode=true -i -M quiescent -C 3 “ start 2>&1\
> 20160910:21:58:09:013425 gpsegstart.py_lin:gpadmin:lin:gpadmin-[DEBUG]:-[worker1] finished cmd: Starting seg at dir /gpadmin/data/primary/gpseg1 cmdStr=’env GPSESSID=0000000000 GPERA=a0ceea5600521f9c_160910215805 $GPHOME/bin/pg_ctl -D /gpadmin/data/primary/gpseg1 -l /gpadmin/data/primary/gpseg1/pg_log/startup.log -w -t 600 -o “ -p 40001 -b 3 -z 4 –silent-mode=true -i -M quiescent -C 1 “ start 2>&1’ had result: cmd had rc=1 completed=True halted=False\
>  stdout=’waiting for server to start……pg_ctl: PID file “/gpadmin/data/primary/gpseg1/postmaster.pid” does not exist\
>  stopped waiting\
> pg_ctl: could not start server\
> Examine the log output.\
> ‘\
>  stderr=’’\
> 20160910:21:58:09:013425 gpsegstart.py_lin:gpadmin:lin:gpadmin-[DEBUG]:-[worker0] finished cmd: Starting seg at dir /gpadmin/data/primary/gpseg0 cmdStr=’env GPSESSID=0000000000 GPERA=a0ceea5600521f9c_160910215805 $GPHOME/bin/pg_ctl -D /gpadmin/data/primary/gpseg0 -l /gpadmin/data/primary/gpseg0/pg_log/startup.log -w -t 600 -o “ -p 40000 -b 2 -z 4 –silent-mode=true -i -M quiescent -C 0 “ start 2>&1’ had result: cmd had rc=1 completed=True halted=False\
>  stdout=’waiting for server to start……pg_ctl: PID file “/gpadmin/data/primary/gpseg0/postmaster.pid” does not exist\
>  stopped waiting\
> pg_ctl: could not start server

其中`$GPHOME/bin/pg_ctl -D /gpadmin/data/mirror/gpseg2 -l /gpadmin/data/mirror/gpseg2/pg_log/startup.log -w -t 600 -o " -p 50000 -b 8 -z 4 --silent-mode=true -i -M quiescent -C 2 " start 2>&1`这个命令是下发到gp segment机器上启动segment的命令。

现在我们通手工启动segment的方式查看是否有异常\
先通过gpstart -m，单独启动master，再登录到对应机器上手动启动对应的segment。gpadmin用户先初始化环境变量`source /usr/local/gpdb/greenplum_path.sh`，\
再运行命令启动第一个segment`$GPHOME/bin/pg_ctl -D /gpadmin/data/primary/gpseg0 -l /gpadmin/data/primary/gpseg0/pg_log/startup.log -w -t 600 -o " -p 40000 -b 2 -z 4 --silent-mode=true -i -M quiescent -C 0 " start 2>&1`，启动成功;\
继续运行该机器上的第二个segment`$GPHOME/bin/pg_ctl -D /gpadmin/data/primary/gpseg1 -l /gpadmin/data/primary/gpseg1/pg_log/startup.log -w -t 600 -o " -p 40001 -b 3 -z 4 --silent-mode=true -i -M quiescent -C 1 " start 2>&1`，又成功了;\
运行命令启动第三个segment `$GPHOME/bin/pg_ctl -D /gpadmin/data/mirror/gpseg3 -l /gpadmin/data/mirror/gpseg3/pg_log/startup.log -w -t 600 -o " -p 50001 -b 9 -z 4 --silent-mode=true -i -M quiescent -C 3 " start 2>&1`。这次失败了，终端输出日志如下:

> waiting for server to start……pg_ctl: PID file “/gpadmin/data/mirror/gpseg3/postmaster.pid” does not exist\
>  stopped waiting\
> pg_ctl: could not start server\
> Examine the log output.

报postmaster.pid不存在，启动失败自然就不会有pid文件，都会报这个，去看该segment启动失败的具体错误，进入日志目录`cd /gpadmin/data/mirror/gpseg3/pg_log`。查看最新的日志`vi startup.log`。有日志内容如下

> 2016-09-10 21:58:07.917398 CST,,,p13456,th-1458132928,,,,0,,,seg-1,,,,,”FATAL”,”XX000”,”could not create semaphores: No space left on device (pg_sema.c:129)”,”Failed system call was semget(50001017, 17, 03600).”,”This error does *not* mean that you have run out of disk space.\
> It occurs when either the system limit for the maximum number of semaphore sets (SEMMNI), or the system wide maximum number of semaphores (SEMMNS), would be exceeded. You need to raise the respective kernel parameter. Alternatively, reduce PostgreSQL’s consumption of semaphores by reducing its max_connections parameter (currently 753).

这段日志提示的内容很明显，超过了系统参数设置。建议调大该值SEMMNI或SEMMNS。

## 问题解决

semaphores是信号量的意思，信号量能起到一个进程间锁和通信的作用(具体作用和用法需要另查资料)，此处理解gp segment启动时，会申请一定的信号量，当申请到第三个的时候，便超过限制了。它包含四个内核参数值:\
semmsl,semmns,semopm,semmni。对应的意思分别为:\
semmsl:最大的信号数量\
semmns：系统调用允许的最大信号量个数，至少100，或者等于semmsl。\
semmni:系统信号量set最大个数，也就是日志中的semaphore sets超出的个数.\
semmsl：每个semaphore set中最多包含的信号个数。\
按着刚才的日志，建议调大semmni或semmns。\
通过命令行`sysctl -a|grep sem`查看信号量配置,sem即semaphores的缩写,得到结果对应的分别为semmsl,semns,semopm,semmni。

> kernel.sem = 250 32000 32 128\
> 而按照官网该值的推荐设置，该值为\
> kernel.sem = 250 512000 100 2048\
> 在initsystem之前，未修改过该值。现在按照官网配置修改掉系统内核参数，并重启机器。待机器重启完成后，运行`gpstart -a`，集群启动成功。
