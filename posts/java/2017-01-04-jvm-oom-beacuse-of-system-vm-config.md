---
title: "机器有大量剩余内存时但java进程还是报OOM错误"
date: "2017-01-04T05:28:51.000Z"
tags:
  - "java"
source_url: "https://gpdream.github.io/2017/01/04/java/jvm%20oom%20beacuse%20of%20system%20vm%20config/"
recovery_source: "html"
---
# 机器有大量剩余内存时但java进程还是报OOM错误

## 现象

当前环境下有tomcat和ambari-server,zookeeper等java进程在运行，其中tomcat未设置堆大小，ambari-server和zookeeper均在有设置内存大小，大小为几GB。\
但在运行`java -version`的时候出错，提示\

```plain
[root@m1 tmp]# Error occurred during initialization of VM
Could not reserve enough space for object heap
Error: Could not create the Java Virtual Machine.
Error: A fatal exception has occurred. Program will exit.
```

通过`free -h`查看发现机器有很大的空余内存，但还是报内存不够初始化jvm。

```plain
[root@m1 ~]# free -h
              total        used        free      shared  buff/cache   available
Mem:            94G        1.4G         90G        2.2G        2.5G         90G
Swap:          4.0G          0B        4.0G
```

## 问题分析

### jvm参数的影响

首先看看jvm进程内存的分配，jvm进程内存结构由如下部分组成

- 计数器

  大小忽略不计
- Java虚拟机栈

  虚拟机栈是归属于线程的，每个线程对应的这个参数为VMThreadStackSize 默认1mb，也就是每有一个线程会去申请1mb，该值不受堆内存影响，在没有太多线程的情况下也不会占用掉特别多的内存
- 本地方法栈\
  和虚拟机栈差不多，在此次问题定位中占用的空间大小可忽略
- Java堆\
  默认最大值为系统内存的1/4，在当前环境下是23.5g。对应参数`-XX:MaxHeapSize`
- 方法区

  方法去又叫Non-Heap（对应参数非堆），对应参数`-XX:MaxPermSize`。默认值166m，用于存储已被虚拟机加载的类型信息、常量、静态变量、即时编译后的代码等信息。\
  常量池：占用的非堆的内存\
  直接内存：0

综上当前环境下，真正影响jvm内存最大的两个参数分别是堆大小`-XX:MaxHeapSize` 和非堆大小`-XX:MaxPermSize`。\
在测试的时候停止掉系统的ambari-server和tomcat，只留下zookeeper，zk配置了-Xmx 1024m\
依次运行命令\

```plain
[root@m1 ~]# java -Xmx44g -version
java version "1.7.0_91"
OpenJDK Runtime Environment (rhel-2.6.2.3.el7-x86_64 u91-b00)
OpenJDK 64-Bit Server VM (build 24.91-b01, mixed mode)

[root@m1 ~]# java -Xmx45g -version
Error occurred during initialization of VM
Unable to allocate 1479872KB bitmaps for parallel garbage collection for the requested 47355904KB heap.
Error: Could not create the Java Virtual Machine.
Error: A fatal exception has occurred. Program will exit.

[root@m1 ~]# java -Xmx43g -XX:MaxPermSize=1g -version
java version "1.7.0_91"
OpenJDK Runtime Environment (rhel-2.6.2.3.el7-x86_64 u91-b00)
OpenJDK 64-Bit Server VM (build 24.91-b01, mixed mode)
You have mail in /var/spool/mail/root

[root@m1 ~]# java -Xmx43g -XX:MaxPermSize=2g -version
Error occurred during initialization of VM
Unable to allocate 1474560KB bitmaps for parallel garbage collection for the requested 47185920KB heap.
Error: Could not create the Java Virtual Machine.
Error: A fatal exception has occurred. Program will exit.
```

可以看到当堆和非堆大小为44g时候可运行，而超过45g的时候就不能运行。此时jvm去向系统申请的内存会略大于44g。但不足以起到决定性的影响。系统明明有94g的总内存和90g的空闲内存，为何只能申请到44g的内存呢？

### 系统参数的影响

当前环境设置了一些内核参数，其中有`vm.overcommit_memory = 2
vm.overcommit_ratio = 50`这两个参数。其中`vm.overcommit_ratio = 50`和系统默认值相同，系统的会分配的内存为总内存*该值百分比，也就是94*50%=47G。\
其中zk已经申请了1g多一点点的内存，还剩下45+G的内存。由此可见系统剩余的可分配内存为45G左右。此时当jvm的参数`-Xmx43g -XX:MaxPermSize=1g`为44g的时候，可以正常运行，但当达到45g的时候就会报错。

那为什么在启动tomcat后再启动`java -version`就会报错呢？这是因为-Xmx的默认值是系统总内存的1/4，当前环境下是23.5g，`java -version`也同样是默认23.5g，加起来的时候已经超过了系统允许分配的最大内存。因此`java -version`运行出错。

还剩下最后一个问题，其他的机器上同样没有设置-Xmx，那-Xmx理论上也是设置为系统的1/4，为何可以运行多个jvm程序呢？这是因为这个集群中的系统都设置了`vm.overcommit_memory = 2`这个参数。\
该值有3个配置项分别为1,2,3对应作用如下:

- 0:系统在为应用进程分配虚拟地址空间时，会判断当前申请的虚拟地址空间大小是否超过剩余内存大小，如果超过，则虚拟地址空间分配失败。因此，也就是如果进程本身占用的虚拟地址空间比较大或者剩余内存比较小时，fork、malloc等调用可能会失败。
- 1:系统在为应用进程分配虚拟地址空间时，完全不进行限制，这种情况下，避免了fork可能产生的失败，但由于malloc是先分配虚拟地址空间，而后通过异常陷入内核分配真正的物理内存，在内存不足的情况下，这相当于完全屏蔽了应用进程对系统内存状态的感知，即malloc总是能成功,除非出现OOM，进程才会被杀掉
- 2:根据系统内存状态确定了虚拟地址空间的上限，由于很多情况下，进程的虚拟地址空间占用远大小其实际占用的物理内存，这样一旦内存使用量上去以后，对于一些动态产生的进程(需要复制父进程地址空间)则很容易创建失败

该值的系统默认值为0，也就是只有当系统剩余内存不足以为虚拟机申请的时候才会失败。而其他环境中java进程不会立即达到-Xmx的内存值，故可以运行多个。\
在当前环境中设置成了2，虽然系统空闲内存还有，但虚拟地址空间上限已经变化了，故在运行第二个jvm进程的时候便申请失败。

## 结论和解决办法

综上，是因为`vm.overcommit_memory = 2
vm.overcommit_ratio = 50`这两个系统参数影响了操作系统的可分配vm内存，而且有两个java进程没有设置Xmx大小。

`java -version`启动失败，是因为当前环境下tomcat没有设置堆内存大小，申请掉了系统1/4的内存，使剩余可分配内存变为了1/4.而`java -version`同样没有设置-Xmx，启动jvm的时候想要再去申请系统1/4的时候已资源不足，毕竟总共可分配内存为系统的1/2.

## 解决办法

老老实实在java启动时候带上-Xmx参数，当前环境的tomcat用不了多少资源，有2g就差不多够了。
