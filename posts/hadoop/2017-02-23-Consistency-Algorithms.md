---
title: "分布式一致性算法"
date: "2017-02-23T05:23:49.000Z"
tags:
  - "hadoop"
source_url: "https://gpdream.github.io/2017/02/23/hadoop/Consistency%20Algorithms/"
recovery_source: "html"
---
# 分布式一致性算法

介绍一致性hash,raft和gossip三种一致性算法\
\
分布式算法几个考虑的点

---

*本部分内容待修改*

- 平衡性\
  当数据分布到不同机器上的时候，数据要尽可能的均衡，避免出现数据倾斜，使所有节点得到充分利用。
- 单调性\
  当系统出现节点变动的时候，比如增加一个节点，哈希的结果应该保证原有已分配的内容可以被映射到新的分片节点中。举例原有3个分片节点，如果增加一个节点为4个的时候，此时已有节点的数据会迁移到新加的节点，而不会移动到其他旧的节点中。
- 分散性（什么鬼）\
  在分布式环境中，终端有可能看不到所有的分片数据，而是只能看到其中的一部分。\
  当终端希望通过哈希过程将内容映射到分片节点上时，由于不同终端所见的分片节点范围有可能不同，从而导致哈希的结果不一致，最终的结果是相同的内容被不同的终端映射到不同的分片节点中。\
  这种情况显然是应该避免的，因为它导致相同内容被存储到不同分片节点中去，降低了系统存储的效率。\
  分散性的定义就是上述情况发生的严重程度。\
  好的哈希算法应能够尽量避免不一致的情况发生，也就是尽量降低分散性。
- 负载(什么鬼)\
  负载问题实际上是从另一个角度看待分散性问题。\
  既然不同的终端可能将相同的内容映射到不同的分片节点中，那么对于一个特定的分片节点而言，也可能被不同的用户映射为不同的内容。\
  与分散性一样，这种情况也是应当避免的，因此好的哈希算法应能够尽量降低分片节点的负荷。

---

## 一致性hash

假设现在有3台机器node(0到3)。要对对象object(3到5)存储在机器上。

### 普通hash的做法

进行hash存储在3台不同机器上的时候。以hash取余为例，mod(hash(object))的数据会落成这样。\

```plain
node0       object3
node1       object4
node2       object5
```

当出现节点变化的时候，比如增加了机器node3，则此时取余的值变成了4.需要数据重分布，数据变成了。\

```plain
node0       object4
node1       object5
node2       object3
node3
```

几乎所有的数据都进行了一次数据重分布，或许有部分数据不需要迁移，但这样的数据单调性是明显不符合的。

### 一致性hash的做法

同样是有3台机器，一致性hash引入了虚拟节点的概念。来更好的保证节点的平衡性.一个物理节点对应若干个虚拟节点，以此让数据更均匀一些落在物理节点上。这么做的话，数据就落在了6个虚拟节点上。\

```plain
node0   node0-1,node0-2
node1   node1-1,node1-2
node2   node2-1,node2-2
```

而3个object的hash结果如下.6个虚拟节点对应的key值组成一个环，其key值落点如下图所示。然后将3个object做hash，橙色的圆为其hash值落在环中。\
![image](http://imgsrc.baidu.com/forum/w%3D580/sign=81475b9d05d79123e0e0947c9d355917/d733941c8701a18bcc1119da972f07082a38fe90.jpg)

根据key值不同，以顺时针的方向把object落在虚拟节点上。\

```plain
node0-1
node0-2 object1
node1-1
node1-2 object2
node2-1 object3
node2-2
```

假设现在出现了节点的变化，比如node2-1这个虚拟节点不见了。则该节点上的数据需要迁移，只需要顺时针迁移到最近的一个虚拟节点上node2-2就行，其他节点上的数据无需迁移。\
假设新增一个节点，新增的节点的key值落在object1和node0-2之间，也只需要把object1的迁移到新加的节点上，不用像hash一样需要全部数据算一遍。

是不是和gp的sengment有些许相似？

## raft算法

使用raft算法的有etcd，Consul (两个的功能都类似zookeeper),\
raft算法有两部分，一部分为选主，另一部分为具体的操作，称为日志复制( log replication)。

### 选举

raft有3种角色，。分别是

- follower：跟随者，能参与选举，接受leader的下发命令
- Candidate: 候选者，当没有leader的时候，任意一个角色都可以把自己变成候选者，并要求集群中其他角色向自己投票。
- leader: 选出来的主，接受客户端的请求，并将请求下发给follower。

#### 只有一个候选者情况下选主

三个节点中，在若干次心跳周期内未发现leader，此时最先确认没有leader的节点会把自己变成候选者，并向其他节点发送请求投票的通知，没有人和他同时成为候选者，所有票都投在了node A节点上。nodeA成为leader。如果有follower宕机，只要存活的follower还大于一半，leader就可以继续工作。\
![image](http://olsznr08y.bkt.clouddn.com/%E6%AD%A3%E5%B8%B8%E9%80%89%E4%B8%BB.png)

当我们把集群的leader （NODE C）给停止掉，则此时集群中便没有了leader，在心跳超时后，需要开始重新选主.\
![image](http://olsznr08y.bkt.clouddn.com/nodec%20leader%E6%8C%82%E4%BA%86.png)

当NODE B成为主后，虽然NODE C已经挂了，但还是会认为NODE C是follower，并向NODE C发送若干次心跳确认节点死亡。\
![image](http://olsznr08y.bkt.clouddn.com/%E5%90%91%E5%B7%B2%E6%8C%82%E7%9A%84%E8%8A%82%E7%82%B9%E5%8F%91%E9%80%81%E5%BF%83%E8%B7%B3.png)

#### 同时有多个候选者选举

现在有4个节点，集群中未发现leader，同时有2个节点把自己变成了候选者，并广播(或组播)给集群中其他角色要求投票。NODE A发现集群没有leader，把自己变成了候选者，把投票请求广播给了（B,C,D）。与此同时，NODE B也成为了候选者，把投票请求发送给了(A,C,D)。\
![image](http://olsznr08y.bkt.clouddn.com/%E5%8F%8C%E5%80%99%E9%80%89%E8%80%85%E7%AB%9E%E4%BA%89%E9%80%89%E4%B8%BB.png)

节点总是同意候选者成为leader，在这种情况下，NODE A（获得B,C,D）和NODE B（获得A,C,D）获得的票数就一样多，NODE A和NODE B在获取到相同票数的时候.都会等待一段时间（时间随机）再进行下一次投票请求，NODE A等待300ms，NODE B等待500ms，则下次投票NODE A先发出去，NODE A成为主。这个投票选举的过程可能需要数次才能完成。如果凑巧若干个候选者等待的时间相同，则需要重新选举，理论上最极限的情况可能无限重复选举。\
![image](http://olsznr08y.bkt.clouddn.com/%E7%9B%B8%E5%90%8C%E7%A5%A8%E6%95%B0%E7%AD%89%E5%BE%85.png)

### 日志复制

leader和follower一直通过心跳来确认node对方是否存活。\
![image](http://olsznr08y.bkt.clouddn.com/%E5%AE%A2%E6%88%B7%E7%AB%AF%E8%AF%B7%E6%B1%82.png)\
此时客户端发出请求，把一个值设置为5。具体会有如下动作:

1. client发送set 5请求给leader（node a）
2. leader判断在上一次心跳中follower NODE B和NODE C是否存活
3. leader把set 5的命令复制给NODE B和NODE C
4. leader确认NODE B和NODE C 执行成功或失败
5. leader把执行结果返回给客户端。

### 集群脑裂

把集群的网络给切断，A,B一个网络，CDE一个网络。此时，集群会重新选主，如图所示，A、B中B选为了leader，广播心跳包给A。C/D/E中D为leader，心跳包广播给C和E.此时便分裂成了两个集群。\
![image](http://olsznr08y.bkt.clouddn.com/%E9%9B%86%E7%BE%A4%E8%84%91%E8%A3%82.png)

此时集群有两个接受请求的leader，客户端分别连接了leader B和D，并发送了不同的请求，set8和set3。两个leader分别把操作发给属于自己的follower\
![image](http://olsznr08y.bkt.clouddn.com/%E8%84%91%E8%A3%82%E6%97%B6%E7%9A%84%E6%93%8D%E4%BD%9C.png)

当两个集群的网络故障修复后，其中一个Leader会成为另外一个leader的follower。B成为D的follower，恢复成一个集群，只有一个Leader，B回滚自己脑裂期间的操作，一切数据以leader D为准。\
![image](http://olsznr08y.bkt.clouddn.com/%E8%84%91%E8%A3%82%E9%9B%86%E7%BE%A4%E4%BF%AE%E5%A4%8D.png)

[这个厉害了，图形并茂的告诉你什么叫raft](http://thesecretlivesofdata.com/raft/)

## gossip 算法

gossip算法不像前两种算法一样，不关注于选主，数据不强一致，关心的是数据传播并保证数据最终一致。\
ElasticSearch,Cassandra，Redis等都使用该算法.\
gossip算法又称为病毒传播算法，弱一致性算法，最终一致性算法。gossip算法不像raft一样是强一致性的，但是他保证数据在一定时间后最终是数据一致的。

### 背景

Gossip的意思是流言，灵感来自办公室八卦，只要一个人八卦一下，在有限的时间内所有的人都会知道该八卦的信息，这种方式也与病毒传播类似。

### 传播方式

在gossip 算法里，数据都有自己的_version和key，每个数据的结构类似于`<key,document,_version>`，_version的维护方式有两种，一种是整体协调，一种是精确协调

- 整体协调\
  整体协调的一个节点上只有一个_version，数据data1,2,3,4,5,…都使用相同的_version。\
  `<key,document,_version>`中的version就全部使用了node的全局version.\
  以nodeA和nodeB通信，发现nodeB的数据_version比自己高的时候，便把所有的数据都以nodeB为准。
- 精确协调\
  精确协调会把每个数据维护上自己的_version。每次发现_version不同就把所有的最新的_version为准是很累的。当_version发生变化的时候，便可以把自己的_version推送给其他节点。如其算法名gossip一样“王大妈，我告诉你一个新的流言，上个版本的说法过时了”.NODEA传给NODEB，NODEB又传给其他人，以达到数据的最终一致。

### 以elasticsearch为例

假设elasticsearch有3个节点，node1,node2,node3.\
集群启动后node1是master node和datanode,node2、node3都是datanode。\
假设现在有数据`<1,data,_version 1>`，1位key，data为具体数据，_version为版本号。客户端发出一个更新请求，要求更新数据为`<1,data2>`。

1. masternode接收到来自客户端的请求，update 1
2. masterNode接受请求，并告诉其中一个node假设为node2做一个这样的插入，在这里就是`<1,data2,_version 2>`
3. node2接收到这个insert 请求后，会push给其他node（node3或者node1），node3再传播给其他node。达成最终的数据一致。
