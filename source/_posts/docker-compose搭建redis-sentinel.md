---
title: docker-compose搭建redis-sentinel
tags: []
id: '44'
categories:
  - - uncategorized
date: 2020-09-20 18:43:56
---

​ 对于上篇文章redis持久化rdb及aof中，**redis**服务器重启时的数据恢复，在新版本中是不符合我画的那个流程图的。

​ **redis**启动的时候会去判断是否开启**aof**，如果开启了，不存在**aof**文件的话，会去判断是否存在**rdb**，但在新的版本中，如果开启**aof**，不存在**aof**文件的时候，**redis**会主动创建**aof**文件并且加载**aof**，这就会导致数据丢失。解决方案如下：

*   关闭**aof**
*   启动**redis**去加载**rdb**文件
*   动态开启**aof**最终达到数据一致性

​ 当主机**master**宕机以后，需要人工解决切换，比如使用`slaveof no one`。实际上主从复制并没有实现高可用。

高可用侧重备份机器，利用集群中系统的冗余，当系统中某台机器发生损坏的时候，其它后备的机器可以迅速的接替它来启动服务。

![](http://qiniu.gaobinzhan.com/2020/06/19/5fae513acdbaa.png)

如何解决：

如果我们有一个监控程序能够监控各个机器的状态并及时调整，手动操作变为自动操作，**Sentinel**的出现就是为了解决这个问题。

## 哨兵机制的原理

​ **Reids Sentinel**一个分布式架构，其中包含若干个**Sentinel**节点和**Redis**数据节点，每个**Sentinel**节点会对数据节点和其余**Sentinel**节点进行监控，当它发现节点不可达的时候，会对节点做下线标识。

​ 如果被标识的是主节点，它还会和其它**Sentinel**节点进行协商，当大多数**Sentinel**节点都认为主节点不可达时，它们会选举出一个**Sentinel**节点来完成自动故障转移的工作，同时会将这个变化实时通知给**Redis**应用方。整个过程完全是自动的，不需要人工来介入，所以这套方案很有效地解决了**Redis**高可用的问题。

![](http://qiniu.gaobinzhan.com/2020/06/19/cf0f4068d4277.png)

基本的故障转移流程：

*   主节点出现故障，此时两个从节点与主节点失去连接，主从复制失败。
*   每个**Sentinel**节点通过定期监控发现主节点出现了故障
*   多个**Sentinel**节点对主节点的故障达成一致会选举出其中一个节点作为领导者负责故障转移。
*   **Sentinel**领导者节点执行了故障转移，整个过程基本是跟我们手动调整一致的，只不过是自动化完成的。
*   故障转移后整个**Redis Sentinel**的结构，重新选举了新的主节点。

Redis Sentinel具有的功能：

*   **监控**：**Sentinel**节点会定期检查Redis数据节点、其余Sentinel节点是否可达。
*   **通知**：**Sentinel**节点会将故障转移的结果通知给应用方。
*   **主节点故障转移**：实现从节点晋升为主节点并维护后续正确的主从关系。
*   **配置提供者**：在**Redis Sentinel**结构中，客户端在初始化的时候连接的是**Sentinel**节点集合，从中获取主节点信息。

同时**Redis Sentinel**包含了若干个**Sentinel**节点，这样做也带了两个好处：

*   对于节点的故障判断是由多个**Sentinel**节点共同完成，这样可以有效地防止误判。
*   **Sentinel**节点集合是由若干个**Sentinel**节点组成的，这样即使个别**Sentinel**节点不可用，整个**Sentinel**节点集合依然是健壮的。

但是**Sentinel**节点本身就是独立的**Reids**节点，只不过它们有一些特殊，不存储数据，只支持部分命令。

## docker-compose 实现 redis-sentinel

容器名称

容器IP

映射端口号

服务运行模式

Redis-master

192.168.3.2

6380->6379

Master

Redis-slave1

192.168.3.3

6381->6379

Slave

Redis-slave2

192.168.3.4

6382->6379

Slave

Redis-sentinel1

192.168.3.11

26380->26379

Sentinel

Redis-sentinel2

192.168.3.12

26381->26379

Sentinel

Redis-sentinel3

192.168.3.13

26382->26379

Sentinel

在这里我用的镜像是redis官方的6.0.5。去网上把配置文件下载下来(`redis.conf`、`sentinel.conf`)

然后开始进行：

![](http://qiniu.gaobinzhan.com/2020/06/20/44959f339649e.png)

创建目录，并且把配置文件拷贝进去。

**sentinel**目录下的所有配置文件进行简单的修改：

搜索`sentinel monitor` 改为 `sentinel monitor mymaster 192.168.3.2 6379 2`

**server**目录下进行修改：

`bind 127.0.0.1`改为`bind 0.0.0.0`

`replicaof` 改为`replicaof 192.168.3.2 6379`(除master目录)

创建`docker-compose.yml`：

```
version: "3.6"
services:
  redis-master:
    image: redis
    container_name: "redis-master"
    ports:
      - "6380:6379"
    volumes:
      - /Users/gaobinzhan/Documents/Redis/server/master:/redis
    command: redis-server /redis/conf/redis.conf
    networks:
      redis-test:
        ipv4_address: 192.168.3.2
  redis-slave1:
    image: redis
    container_name: "redis-slave1"
    ports:
      - "6381:6379"
    volumes:
      - /Users/gaobinzhan/Documents/Redis/server/slave1:/redis
    command: redis-server /redis/conf/redis.conf
    networks:
      redis-test:
        ipv4_address: 192.168.3.3
  redis-slave2:
    image: redis
    container_name: "redis-slave2"
    ports:
      - "6382:6379"
    volumes:
      - /Users/gaobinzhan/Documents/Redis/server/slave2:/redis
    command: redis-server /redis/conf/redis.conf
    networks:
      redis-test:
        ipv4_address: 192.168.3.4
  redis-sentinel1:
    image: redis
    container_name: "redis-sentinel1"
    ports:
      - "26380:26379"
    volumes:
      - /Users/gaobinzhan/Documents/Redis/sentinel/sentinel1:/redis
    command: redis-sentinel /redis/conf/sentinel.conf
    networks:
      redis-test:
        ipv4_address: 192.168.3.11
  redis-sentinel2:
    image: redis
    container_name: "redis-sentinel2"
    ports:
      - "26381:26379"
    volumes:
      - /Users/gaobinzhan/Documents/Redis/sentinel/sentinel2:/redis
    command: redis-sentinel /redis/conf/sentinel.conf
    networks:
      redis-test:
        ipv4_address: 192.168.3.12
  redis-sentinel3:
    image: redis
    container_name: "redis-sentinel3"
    ports:
      - "26382:26379"
    volumes:
      - /Users/gaobinzhan/Documents/Redis/sentinel/sentinel3:/redis
    command: redis-sentinel /redis/conf/sentinel.conf
    networks:
      redis-test:
        ipv4_address: 192.168.3.13
networks:
  redis-test:
    driver: bridge
    ipam:
      config:
        - subnet: "192.168.3.0/24"
```

进行`docker-compose up`，执行完毕后：

redis-cli工具运行`redis-cli -p 26380`输入`info`：

![](http://qiniu.gaobinzhan.com/2020/06/20/6f1b2c36d3e39.png)

出现以上信息即搭建成功。。

**Sentinel**的核心配置：

`sentinel monitor mymaster 192.168.3.2 6379 2`

监控的主节点的名字、IP 和端口，最后一个2的意思是有几台 **Sentinel**发现有问题，就会发生故障转移，例如 配置为2，代表至少有2个 **Sentinel** 节点认为主节点不可达，那么这个不可达的判定才是客观的。对于设置的越小，那么达到下线的条件越宽松，反之越严格。一般建议将其设置为 **Sentinel** 节点的一半加1。最后的参数不可大于**Sentinel**节点数。

`sentinel down-after-millseconds mymaster 30000`

这个是超时的时间（单位为毫米）。打个比方，当你去**ping**一个机器的时候，多长时间后仍**ping**不通，那么就认为它是有问题。

`sentinel parallel-syncs my master 1`

当**Sentinel**节点集合对主节点故障判断达成一致时，**Sentinel**领导者节点会被做故障转移操作，选出新的主节点，原来的从节点会向新的主节点发起复制操作，`paraller-syncs`就是用来限制在一次故障转移之后，每次向新的主节点发起复制操作的从节点个数，指出**Sentinel**属于并发还是串行。1代表每次只能复制一个，可以减轻**Master**的压力。

`sentinel auth-pass`

如果 **Sentinel** 监控的主节点配置了密码，`sentinel auth-pass` 配置通过添加主节点的密码，防止 **Sentinel** 节点对主节点无法监控。

`sentinel failover-timeout mymaster 180000`

表示故障转移的时间。