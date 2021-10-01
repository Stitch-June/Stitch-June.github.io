---
title: 如何搭建redis-cluster
tags: []
id: '32'
categories:
  - - uncategorized
date: 2020-09-20 18:00:14
---

​ 假设在一台主从机器上配置了20G内存，但是业务需求是需要50G内存的时候，主从结构+哨兵可以实现高可用故障切换+冗余备份，但是不能解决数据容量的问题，用哨兵，每个**redis**实例存储的数据也都是完整的数据，浪费内存且有木桶效应。

​ 为了最大化利用内存，可以采用**cluster**，就是分布式存储。即每台**redis**存储不同的内容。

**Redis**分布式方案一般有两种：

*   客户端分区方案：优点是分区逻辑可控，缺点是需要自己处理数据路由，实现高可用、故障转移等问题。比如在**redis2.8**之前通常的做法是获取某个**key**的**hashcode**，然后取余分布到不同节点，不过这种做法无法很好的支持动态伸缩性需求，一旦节点的增或者删操作，都会导致**key**无法在**redis**中命中。
*   代理方案：优点是简化客户端分布式逻辑和升级维护便利，缺点是加重架构部署复杂度和性能损耗。如**twemproxy**、**codis**。

而**redis**官方提供了专有的集群方案：**Redis Cluster**，它非常优雅的解决了**Redis**集群方面的问题，部署方便简单。

## Redis Cluster

​ **Redis Cluster**是**Redis**的分布式解决方案，在**3.0**版本正式推出，有效地解决了**Redis**分布式方面的需求。当遇到单机内存、并发、流量等瓶颈时，可以采用**Cluster**架构方案达到负载均衡的目的。

​ 在**Redis Cluster**，它们任何两个节点之间都是相互连通的。客户端可以与任何一个节点相连接，然后就可以访问集群中的任何一个节点，对其进行存取和其它操作。

**Redis Cluster**提供的好处：

*   将数据自动切分到多个节点的能力
*   当集群中的一部分节点失效或者无法进行通讯时，仍然可以继续处理命令请求的能力，拥有自动故障转移的能力。

**Redis Cluster** 和 **replication + sentinel** 如何选择：

如果数据量很少，主要是承载高并发高性能的场景，比如你的缓存一般就几个G，单机就够了。

*   **Replication**：一个**master**，多个**slave**，要几个**slave**跟你的要求的读吞吐量有关系，结合**sentinel**集群，去保证**redis**主从架构的高可用就行了。
*   **Redis Cluster**：主要是针对海量数据+高并发+高可用的场景，海量数据，如果数据量很大，建议用**Redis Cluster**

数据分布理论：

分布式数据库首先要解决把整个数据集按照分区规则映射到多个节点的问题，即把数据集划分到多个节点上，每个节点负责整体数据的一个子集。

常见的分区规则有哈希分区和顺序分区两种：

*   顺序分布：把一整块数据分散到很多机器中，一般都是平均分配的。
*   哈希分区：通过**hash**的函数，取余产生的数。保证这串数字充分的打散，均匀的分配到各台机器上。

哈希分布和顺序分布只是场景上的适用。哈希分布不能顺序访问，比如你想访问1~100，哈希分布只能遍历全部数据，同时哈希分布因为做了**hash**后导致与业务数据无关了。

分区方式

描述

代表产品

哈希分区

离散度好  
数据分布业务无关  
无法顺序访问

RedisCluster  
Cassanda  
Dynamo

顺序分区

离散度易倾斜  
数据分布业务相关  
可顺序访问

Bigtable  
Hbase  
Hypertable

数据倾斜与数据迁移跟节点伸缩：

顺序分布是会导致数据倾斜的，主要是访问的倾斜。每次点击会重点访问某台机器，这就导致最后数据都到这台机器上了，这就是顺序分布最大的缺点。但哈希分布的时候，假如要扩容机器的时候，称之为“节点伸缩”，这个时候，因为是哈希算法，会导致数据迁移。

哈希分区方式：

*   节点取余分区：
    
    *   使用特点的数据（包括redis的键或用户ID），再根据节点数量N，使用公式：hash(key)%N计算出一个0～（N-1）值，来决定数据映射到哪一个节点上。即哈希值对节点总数取余。
    *   缺点：当节点数量N变化时（扩容或者收缩），数据和节点之间的映射关系需要重新计算，这样的话，按照新的规则映射，要么之前存储的数据找不到，要么之前数据被重新映射到新的节点（导致以前存储的数据发生数据迁移）。
    *   实践：常用于数据库的分库分表规则，一般采用预分区的方式，提前根据数量规划好分区数，比如划分为512或1024张表，保证可支撑未来一段时间的数据量，再根据负载情况将表迁移到其它数据库中。
*   一致性哈希：
    
    *   一致性哈希分区（Distributed Hash Table）实现思路是为系统中每个节点分配一个token，范围一般在0～232，这些token构成一个哈希环。数据读写执行节点查找操作时，先根据key计算hash值，然后顺时针找到第一个大于等于该哈希值的token节点。
        
        ![](http://qiniu.gaobinzhan.com/2020/06/28/9a2f836285ddd.png)
        
        上图就是一个一致性hash的原理解析。
        
        假设有n1～n4这四台机器，我们对每一台机器分配一个唯一token，每次有数据（黄色代表数据），一致性哈希算法规则每次都顺时针漂移数据，也就是图中黄色的数据都指向n3。
        
        这个时候我们需要增加一个节点n5，在n2和n3之间，数据还是会发生漂移（会偏移到大于等于的节点），但是这个时候你是否注意到，其实只有n2～n3这部分的数据被漂移，其它的数据都是不会变的，这种方式相比节点取余最大的好处在于加入和删除节点只影响哈希环中相邻的节点，对其它节点无影响。
        
    *   缺点：每个节点的负载不相同，因为每个节点的hash是根据key计算出来的，换句话说就是假设key足够多，被hash算法打散得非常均匀，但是节点过少，导致每个节点处理的key个数不太一样，甚至相差很大，这就导致某些节点压力很大
    *   实践：加减节点会造成哈希环中部分数据无法命中，需要手动处理或者忽略这部分数据，因此一致性哈希常用于缓存场景。

虚拟槽分区：

虚拟槽分区巧妙地使用了哈希空间，使用分散度良好的哈希函数把所有数据映射到一个固定范围的整数集合中，整数定义为槽（slot）。这个范围一般远远大于节点数，比如Redis Cluster槽范围是0～16383（也就是说有16383个槽）。槽是集群内数据管理和迁移的基本单位。采用大范围槽的主要目的是为了方便数据拆分和集群扩展。每个节点会负责一定数量的槽。

![](http://qiniu.gaobinzhan.com/2020/06/28/1c7845ac0288e.png)

如上图所示，当前集群有5个节点，每个节点平均大约负责3276个槽。由于采用高质量的哈希算法，每个槽所映射的数据通常比较均匀，将数据平均划分到5个节点进行数据分区。Redis Cluster就是采用虚拟槽分区，每当key访问过来，Redis Cluster会计算哈希值是否在这个区间里。它们彼此都知道对应的槽在哪台机器上，这样就能做到平均分配了。

集群限制：批量key操作。

## Docker-compose搭建Redis Cluster

> 因为没有多台机器去部署redis实例，所以这里采用docker来搭建，在生产环境中肯定是多台机器去部署的。

容器名称

ip

端口

redis-cluster1

192.168.3.101

6380->6379  
16380->16379

redis-cluster2

192.168.3.102

6381->6379  
16381->16379

redis-cluster3

192.168.3.103

6382->6379  
16382->16379

redis-cluster4

192.168.3.104

6383->6379  
16383->16379

redis-cluster5

192.168.3.105

6384->6379  
16384->16379

Redis-cluster6

192.168.3.106

6385->6379  
16385->16379

![](http://qiniu.gaobinzhan.com/2020/06/30/2df540d0dcaa0.png)

如上图把redis.conf复制进去，自行去下载，然后分别加入以下代码：

```
bind 0.0.0.0
cluster-enabled yes
cluster-config-file "/redis/conf/nodes.conf"
cluster-node-timeout 5000
protected-mode no
port 6379
daemonize no
```

`docker-compose.yml`

```
version: "3.6"
services:
  redis-cluster1:
    image: redis
    container_name: redis-cluster1
    ports:
      - "6380:6379"
      - "16380:16379"
    volumes:
      - /Users/gaobinzhan/Documents/Redis/cluster/cluster1:/redis
    command: redis-server /redis/conf/redis.conf
    networks:
      redis-test:
        ipv4_address: 192.168.3.101
  redis-cluster2:
    image: redis
    container_name: redis-cluster2
    ports:
      - "6381:6379"
      - "16381:16379"
    volumes:
      - /Users/gaobinzhan/Documents/Redis/cluster/cluster2:/redis
    command: redis-server /redis/conf/redis.conf
    networks:
      redis-test:
        ipv4_address: 192.168.3.102
  redis-cluster3:
    image: redis
    container_name: redis-cluster3
    ports:
      - "6382:6379"
      - "16382:16379"
    volumes:
      - /Users/gaobinzhan/Documents/Redis/cluster/cluster3:/redis
    command: redis-server /redis/conf/redis.conf
    networks:
      redis-test:
        ipv4_address: 192.168.3.103
  redis-cluster4:
    image: redis
    container_name: redis-cluster4
    ports:
      - "6383:6379"
      - "16383:16379"
    volumes:
      - /Users/gaobinzhan/Documents/Redis/cluster/cluster4:/redis
    command: redis-server /redis/conf/redis.conf
    networks:
      redis-test:
        ipv4_address: 192.168.3.104
  redis-cluster5:
    image: redis
    container_name: redis-cluster5
    ports:
      - "6384:6379"
      - "16384:16379"
    volumes:
      - /Users/gaobinzhan/Documents/Redis/cluster/cluster5:/redis
    command: redis-server /redis/conf/redis.conf
    networks:
      redis-test:
        ipv4_address: 192.168.3.105
  redis-cluster6:
    image: redis
    container_name: redis-cluster6
    ports:
      - "6385:6379"
      - "16385:16379"
    volumes:
      - /Users/gaobinzhan/Documents/Redis/cluster/cluster6:/redis
    command: redis-server /redis/conf/redis.conf
    networks:
      redis-test:
        ipv4_address: 192.168.3.106
networks:
  redis-test:
    driver: bridge
    ipam:
      config:
        - subnet: "192.168.3.0/24"
```

然后进行`docker-compose up`

![](http://qiniu.gaobinzhan.com/2020/06/30/22ec0cfb791a0.png)

此刻每个目录下面多了**nodes.conf** 现在文件内容只是单单保存了**redis**实例自身的节点数据。

也可以随便连接一台**redis**，查看集群状态：

```
gaobinzhan-MBP:~ gaobinzhan$ redis-cli -p 6380
127.0.0.1:6380> info Cluster
# Cluster
cluster_enabled:1
```

此刻集群只是开启状态，往里面写入数据会报错：

```
127.0.0.1:6380> set 1 2
(error) CLUSTERDOWN Hash slot not served
```

是因为没有分配槽。

在**redis**之前的版本，需要手动分配槽，非常不方便。现在的版本只需要简单执行下命令就可以了。

随便进入一个容器当中 `docker exec -it redis-cluster1 sh`

然后执行`redis-cli --cluster create 192.168.3.101:6379 192.168.3.102:6379 192.168.3.103:6379 192.168.3.104:6379 192.168.3.105:6379 192.168.3.106:6379 --cluster-replicas 1`

```
gaobinzhan-MBP:~ gaobinzhan$ docker exec -it redis-cluster1 sh
# redis-cli --cluster create 192.168.3.101:6379 192.168.3.102:6379 192.168.3.103:6379 192.168.3.104:6379 192.168.3.105:6379 192.168.3.106:6379 --cluster-replicas 1
>>> Performing hash slots allocation on 6 nodes...
Master[0] -> Slots 0 - 5460
Master[1] -> Slots 5461 - 10922
Master[2] -> Slots 10923 - 16383
Adding replica 192.168.3.105:6379 to 192.168.3.101:6379
Adding replica 192.168.3.106:6379 to 192.168.3.102:6379
Adding replica 192.168.3.104:6379 to 192.168.3.103:6379
M: 2a373fe66cde81d1530584bb86e49694b000d0c2 192.168.3.101:6379
   slots:[0-5460] (5461 slots) master
M: 8de35d3b2439d05de839a2539f68e4833b90679d 192.168.3.102:6379
   slots:[5461-10922] (5462 slots) master
M: caca339b5511d3a14e381d5ffd9434458c7368bd 192.168.3.103:6379
   slots:[10923-16383] (5461 slots) master
S: 97eccf5edf7fb12d4a2e01b16dbf2d9c561896b3 192.168.3.104:6379
   replicates caca339b5511d3a14e381d5ffd9434458c7368bd
S: 64512db80b1a159ca823a25a7e9893154efd555c 192.168.3.105:6379
   replicates 2a373fe66cde81d1530584bb86e49694b000d0c2
S: 1722206f30f3b7c65706d30f4a1ee3b6e0cbca7c 192.168.3.106:6379
   replicates 8de35d3b2439d05de839a2539f68e4833b90679d
Can I set the above configuration? (type 'yes' to accept): 
```

此刻会提示你，是否接受此配置，输入**yes**即可。

```
Can I set the above configuration? (type 'yes' to accept): yes
>>> Nodes configuration updated
>>> Assign a different config epoch to each node
>>> Sending CLUSTER MEET messages to join the cluster
Waiting for the cluster to join
...
>>> Performing Cluster Check (using node 192.168.3.101:6379)
M: 2a373fe66cde81d1530584bb86e49694b000d0c2 192.168.3.101:6379
   slots:[0-5460] (5461 slots) master
   1 additional replica(s)
S: 64512db80b1a159ca823a25a7e9893154efd555c 192.168.3.105:6379
   slots: (0 slots) slave
   replicates 2a373fe66cde81d1530584bb86e49694b000d0c2
S: 1722206f30f3b7c65706d30f4a1ee3b6e0cbca7c 192.168.3.106:6379
   slots: (0 slots) slave
   replicates 8de35d3b2439d05de839a2539f68e4833b90679d
M: 8de35d3b2439d05de839a2539f68e4833b90679d 192.168.3.102:6379
   slots:[5461-10922] (5462 slots) master
   1 additional replica(s)
S: 97eccf5edf7fb12d4a2e01b16dbf2d9c561896b3 192.168.3.104:6379
   slots: (0 slots) slave
   replicates caca339b5511d3a14e381d5ffd9434458c7368bd
M: caca339b5511d3a14e381d5ffd9434458c7368bd 192.168.3.103:6379
   slots:[10923-16383] (5461 slots) master
   1 additional replica(s)
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
```

此时**redis-cluster**已经搭建好了，**nodes.conf**文件也发生了变化，执行**cluster info**：

```
127.0.0.1:6380> cluster info
cluster_state:ok
cluster_slots_assigned:16384
cluster_slots_ok:16384
cluster_slots_pfail:0
cluster_slots_fail:0
cluster_known_nodes:6
cluster_size:3
cluster_current_epoch:6
cluster_my_epoch:1
cluster_stats_messages_ping_sent:234
cluster_stats_messages_pong_sent:242
cluster_stats_messages_sent:476
cluster_stats_messages_ping_received:237
cluster_stats_messages_pong_received:234
cluster_stats_messages_meet_received:5
cluster_stats_messages_received:476
```

状态是**ok**的，来试着写写数据。进入**redis**实例`docker exec -it redis-cluster1 sh`

```
gaobinzhan-MBP:~ gaobinzhan$ docker exec -it redis-cluster1 sh
# redis-cli
127.0.0.1:6379> set 1 2
(error) MOVED 9842 192.168.3.102:6379
```

报错了，因为槽的问题，这个数据需要写入到102那台实例当中，这时可以用集群方式启动`redis-cli -c`

```
# redis-cli -c
127.0.0.1:6379> set 1 2
-> Redirected to slot [9842] located at 192.168.3.102:6379
OK
192.168.3.102:6379> get 1
"2"
```

会发现提示数据移动到102节点了，因为是集群方式所以可以获取的，切换到102节点：

```
# redis-cli -c -h 192.168.3.102
192.168.3.102:6379> get 1
"2"
```

也可以正常的获取。

## 往期方式搭建集群

### 准备节点

**Redis**集群一般由多个节点组成，节点数量至少为6个才能保证组成完整高可用的集群，前面的主从复制跟哨兵共同构成了高可用。每个节点需要开启配置 `cluster-enabled yes` ，让**Redis** 运行在集群模式下。

*   性能：这是Redis赖以生存的看家本领，增加集群功能后当然不能对性能产生太大影响，所以**Redis**采取了**P2P**而非**Proxy**方式、异步复制、客户端重定向等设计，而牺牲了部分的一致性、使用性。
*   水平扩展：集群的最重要能力当然是扩展，文档中称可以线性扩展到1000结点。 可用性：在**Cluster**推出之前，可用性要靠Sentinel保证。有了集群之后也自动具有了**Sentinel**的监控和自动**Failover**能力

集群的相关配置：

```
#节点端口
port 6379
#开启集群模式
cluster-enabled yes
#节点超时时间，单位毫秒
cluster-node-timeout 15000
#集群内部配置文件
cluster-config-file "nodes-6379.conf"
```

其他配置和单机模式一致即可，配置文件命名规则`redis-{port}.conf` ，准备好配置后启动所有节点，第一次启动时如果没有集群配置文件，它会自动创建一份，文件名称采用 cluster-config-file 参数项控制，建议采用`node-{port}.conf`格式定义，也就是说会有两份配置文件。

当集群内节点信息发生变化，如添加节点、节点下线、故障转移等。节点会自动保存集群状态到配置文件中。需要注意的是， **Redis**自动维护集群配置文件，不要手动修改，防止节点重启时产生集群信息错乱。

![](http://qiniu.gaobinzhan.com/2020/06/30/e8f9b47eabdc5.png)

然后就跟上面一样准备准备节点就行了。

### 节点握手

节点握手是指一批运行在集群模式下的节点通过**Gossip**协议彼此通信，达到感知对方的过程。节点握手是集群彼此通信的第一步，由客户端发起命令：`cluster meet{ip}{port}`

通过命令`cluster meet 127.0.0.1 6381`让节点6380和6381节点进行握手通信。`cluster meet`命令是一个异步命令，执行之后立刻返回。内部发起与目标节点进行握手通信 。

*   节点6380本地创建6381节点信息对象，并发送meet消息。
*   节点6381接受到meet消息后，保存6380节点信息并回复pong消息。
*   之后节点6380和6381彼此定期通过ping/pong消息进行正常的节点通信。

通过`cluster nodes`命令确认6个节点都彼此感知并组成集群。

注意：

*   每个**Redis Cluster**节点会占用两个TCP端口，一个监听客户端的请求，默认是**6379**，另外一个在前一个端口加上10000， 比如**16379**，来监听数据的请求，节点和节点之间会监听第二个端口，用一套二进制协议来通信。节点之间会通过套协议来进行失败检测，配置更新，`failover`认证等等。为了保证节点之间正常的访问，需要注意防火墙的配置。
*   节点建立握手之后集群还不能正常工作，这时集群处于下线状态，所有的数据读写都被禁止。
*   设置从节点作为一个完整的集群，需要主从节点，保证当它出现故障时可以自动进行故障转移。集群模式下，Reids 节点角色分为主节点和从节点。
*   首次启动的节点和被分配槽的节点都是主节点，从节点负责复制主节点槽信息和相关的数据。
*   使用 `cluster replicate {nodeId}`命令让一个节点成为从节点。其中命令执行必须在对应的从节点上执行，将当前节点设置为`node_id`指定的节点的从节点。

### 分配槽

Redis 集群把所有的数据映射到16384个槽中。每个key会映射为一个固定的槽，只有当节点分配了槽，才能响应和这些槽关联的键命令。通过cluster addslots命令为节点分配槽利用bash特性批量设置槽（slots），命令如下：

`redis-cli -h 192.168.3.101 cluster addslots {0..5461}`

`redis-cli -h 192.168.3.102 cluster addslots {5462..10922}`

`redis-cli -h 192.168.3.103 cluster addslots {10923..16383}`

然后就可以进行操作了。

集群的命令：

```
CLUSTER nodes： 列出集群当前已知的所有节点（node）的相关信息。 

CLUSTER meet  ： 将ip和port所指定的节点添加到集群当中。 

CLUSTER addslots  [slot ...]： 将一个或多个槽（slot）指派（assign）给当前节点。 

CLUSTER delslots  [slot ...]： 移除一个或多个槽对当前节点的指派。 

CLUSTER slots： 列出槽位、节点信息。 

CLUSTER slaves ： 列出指定节点下面的从节点信息。 

CLUSTER replicate ： 将当前节点设置为指定节点的从节点。 

CLUSTER saveconfig： 手动执行命令保存保存集群的配置文件，集群默认在配置修改的时候会自动保存配置文件。 

CLUSTER keyslot ： 列出key被放置在哪个槽上。 

CLUSTER flushslots： 移除指派给当前节点的所有槽，让当前节点变成一个没有指派任何槽的节点。 

CLUSTER countkeysinslot ： 返回槽目前包含的键值对数量。 

CLUSTER getkeysinslot  ： 返回count个槽中的键。 

CLUSTER setslot  node  将槽指派给指定的节点，如果槽已经指派给另一个节点，那么先让另一个节点删除该槽，然后再进行指派。

CLUSTER setslot  migrating  将本节点的槽迁移到指定的节点中。 

CLUSTER setslot  importing  从 node_id 指定的节点中导入槽 slot 到本节点。 

CLUSTER setslot  stable 取消对槽 slot 的导入（import）或者迁移（migrate）。 

CLUSTER failover： 手动进行故障转移。 

CLUSTER forget ： 从集群中移除指定的节点，这样就无法完成握手，过期时为60s，60s后两节点又会继续完成握手。 

CLUSTER reset [HARDSOFT]： 重置集群信息，soft是清空其他节点的信息，但不修改自己的id，hard还会修改自己的id，不传该参数则使用soft方式。 

CLUSTER count-failure-reports ： 列出某个节点的故障报告的长度。 

CLUSTER SET-CONFIG-EPOCH： 设置节点epoch，只有在节点加入集群前才能设置。
```

注意：在apline系统中不支持{1..10}操作

> 以上理论知识内容为网络整理。。。。