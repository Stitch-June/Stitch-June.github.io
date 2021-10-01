---
title: redis哨兵故障转移及实现
tags: []
id: '74'
categories:
  - - uncategorized
date: 2020-09-23 08:06:04
---

> 在上篇文章中docker-compose搭建redis-sentinel成功的搭建了1主2从3哨兵。

**sentinel**是一个特殊的**redis**节点，它有自己专属的**api**：

*   `sentinel masters` 显示被监控的所有**master**以及它们的状态。
*   `sentinel master` 显示指定**master**的信息和状态。
*   `sentinel slaves` 显示指定**master**的所有**slave**及它们的状态。
*   `sentinel sentinels` 显示指定**master**的**sentinel**节点集合（不包含当前节点）。
*   `sentinel get-master-addr-by-name` 返回指定**master**的**ip**和**port**，如果正在进行**failover**或者**failover**已经完成，将会显示被提升为**master**的**slave**的**ip**和**port**。
*   `sentinel failover` 强制**sentinel**执行**failover**，并且不需要得到其它**sentinel**的同意。但是**failover**后会将最新的配置发送给其它**sentinel**。

`sentinel masters`

展示所有被监控的主节点状态及相关信息：

```
127.0.0.1:26380> sentinel masters
1)  1) "name"
    2) "mymaster"
    3) "ip"
    4) "192.168.3.2"
    5) "port"
    6) "6379"
…………………………………………………………
```

`sentinel master`

展示指定状态以及相关的信息：

```
127.0.0.1:26380> sentinel master mymaster
 1) "name"
 2) "mymaster"
 3) "ip"
 4) "192.168.3.2"
 5) "port"
 6) "6379"
 ………………………………
```

`sentinel slaves`

展示指定 的从节点状态以及相关的统计信息：

```
127.0.0.1:26380> sentinel slaves mymaster
1)  1) "name"
    2) "192.168.3.4:6379"
    3) "ip"
    4) "192.168.3.4"
    5) "port"
    6) "6379"
…………………………………………
2)  1) "name"
    2) "192.168.3.3:6379"
    3) "ip"
    4) "192.168.3.3"
    5) "port"
    6) "6379"
…………………………………………
```

`sentinel sentinels`

展示指定 的**sentinel**节点集合（不包含当前**sentinel**节点）：

```
127.0.0.1:26380> sentinel sentinels mymaster
1)  1) "name"
    2) "570de1d8085ec8bd7974431c01c589847c857edf"
    3) "ip"
    4) "192.168.3.13"
    5) "port"
    6) "26379"
………………………………………………
```

`sentinel get-master-addr-by-name`

获取主节点信息：

```
127.0.0.1:26380> sentinel get-master-addr-by-name mymaster
1) "192.168.3.2"
2) "6379"
```

`sentinel failover`

对进行强制故障转移：

```
127.0.0.1:26380> sentinel failover mymaster
OK
127.0.0.1:26380> info sentinel
# Sentinel
sentinel_masters:1
sentinel_tilt:0
sentinel_running_scripts:0
sentinel_scripts_queue_length:0
sentinel_simulate_failure_flags:0
master0:name=mymaster,status=ok,address=192.168.3.3:6379,slaves=2,sentinels=3
```

修改配置：

*   添加新的监听：`sentinel monitor test 127.0.0.1 6379 2`
*   放弃对某个**master**监听：`sentinel REMOVE test`
*   设置配置选项：`sentinel set failover-timeout mymaster 180000`

**Master**可能会因为某些情况宕机了，如果客户端是固定一个地址去访问，肯定是不合理的，所以客户端请求是请求哨兵，从哨兵获取主机地址的信息，或者是从机的信息。可以实现一个例子：

*   随机选择一个哨兵连接，获取主机及从机信息。
*   模拟客户端定时访问，实现简单轮询效果，轮询从节点。
*   连接失败重试访问

## Sentinel故障转移

执行`docker-composer up`之后`sentinel.conf`发生了变化，每个配置文件变化如下：

`sentinel\conf\sentinel.conf`

```
user default on nopass ~* +@all
sentinel known-replica mymaster 192.168.3.3 6379
sentinel known-replica mymaster 192.168.3.4 6379
sentinel known-sentinel mymaster 192.168.3.12 26379 497f733919cb5d41651b4a2b5648c4adffae0a73
sentinel known-sentinel mymaster 192.168.3.13 26379 0d0ee41bcb5d765e9ff78ed59de66be049a23a82
sentinel current-epoch 0
```

`sentine2\conf\sentinel.conf`

```
user default on nopass ~* +@all
sentinel known-replica mymaster 192.168.3.3 6379
sentinel known-replica mymaster 192.168.3.4 6379
sentinel known-sentinel mymaster 192.168.3.13 26379 0d0ee41bcb5d765e9ff78ed59de66be049a23a82
sentinel known-sentinel mymaster 192.168.3.11 26379 f5f2a73dc0e60514e4f28c6f40517f48fa409eed
sentinel current-epoch 0
```

`sentine3\conf\sentinel.conf`

```
user default on nopass ~* +@all
sentinel known-replica mymaster 192.168.3.3 6379
sentinel known-replica mymaster 192.168.3.4 6379
sentinel known-sentinel mymaster 192.168.3.12 26379 497f733919cb5d41651b4a2b5648c4adffae0a73
sentinel known-sentinel mymaster 192.168.3.11 26379 f5f2a73dc0e60514e4f28c6f40517f48fa409eed
sentinel current-epoch 0
```

从变化中可以看出每台**Sentinel**分别记录了**slave**的节点信息和其它**Sentinel**节点信息。

在宿主机中随便进入一台**Sentinel**：

```
127.0.0.1:26380> sentinel masters
1)  1) "name"
    2) "mymaster"
    3) "ip"
    4) "192.168.3.2"
    5) "port"
    6) "6379"
```

可以观察到监听的所有**master**，将**192.168.3.2**这台**master**进行宕机

`docker stop redis-master`

宕机完之后等待**Sentinel**检测周期过了之后再对`sentinel.conf`和`redis.conf`进行观察。

3台**Sentinel**的`sentinel monitor mymaster 192.168.3.2 6379 2`变成了`sentinel monitor mymaster 192.168.3.4 6379 2`

其次master对应的slave节点信息也进行更改。

而**192.168.3.3**的`redis.conf`中`replicaof 192.168.3.2 6379`也变成了`replicaof 192.168.3.4 6379`。

**192.168.3.2**的`redis.conf`中`replicaof 192.168.3.2 6379`这行配置被删除掉了。

再次启动**192.168.3.2**的**redis**节点，而这台节点的`redis.conf`中增加了一行`replicaof 192.168.3.4 6379`。

其实就是将我们的操作自动化了。

## Sentinel实现原理

**Sentinel**的实现原理，主要分为三个步骤：

*   检测问题：三个定时任务，这三个内部的执行任务可以保证出现问题马上让**Sentinel**知道。
*   发现问题：主观下线和客观下线，当有一台**Sentinel**机器发现问题时，它就会对它主观下线。但是当多个**Sentinel**都发现问题的时候，才会出现客观下线。
*   找到解决问题的**Sentinel**：进行领导者选举，如何在**Sentinel**内部多台节点做领导者选择。
*   解决问题：就是要进行故障转移。

### 三个定时任务

*   每10s每个**Sentinel**对**Master**和**Slave**执行一次`Info Replication`。
    
    **Redis Sentinel**可以对**Redis**节点做失败判断和故障转移，来`Info Replication`发现**Slave**节点，来确定主从关系。
    
*   每2s每个**Sentinel**通过**Master**节点的**channel**交换信息（pub/sub）。
    
    类似于发布订阅，**Sentinel**会对主从关系进行判断，通过`__sentinel__:hello`频道交互。了解主从关系可以帮助更好的自动化操作**Redis**。然后**Sentinel**会告知系统消息给其它**Sentinel**节点，最终达到共识，同时**Sentinel**节点能够互相感知到对方。
    
*   每1s每个**Sentinel**对其它**Sentinel**和**Redis**执行`ping`。
    
    对每个节点和其它**Sentinel**进行心跳检测，它是失败判断的依据。
    

### 主观下线和客观下线

回顾上一篇文章中**Sentinel**的配置。

```
sentinel monitor mymaster 192.168.3.2 6379 2
sentinel down-after-millseconds mymaster 30000
```

主观下线：每个**Sentinel**节点对**Redis**失败的“偏见”。之所以是偏见，只是因为某一台机器30s内没有得到回复。

客观下线：这个时候需要所以**Sentinel**节点都发现它30s内无回复，才会达到共识。

### 领导者选举方式

*   每个做主观下线的Sentinel节点，会像其它的**Sentinel**节点发送命令，要求将它设置成为领导者。
*   收到命令的**Sentinel**节点，如果没有同意通过其它节点发送的命令，那么就会同意请求，否则就会拒绝。
*   如果**Sentinel**节点发现自己的票数超过半数，同时也超过了`sentinel monitor mymaster 192.168.3.2 6379 2`超过2个的时候，就会成为领导者。
*   进行故障转移操作。

### 如何选择“合适”的Slave节点

​ **Redis**内部其实是有一个优先级配置的，在配置文件中`replica-priority`，这个参数是**slave**节点的优先级配置，如果存在则返回，如果不存在则继续。当上面这个优先级不满足的时候，**Redis**还会选择复制偏移量最大的**Slave**节点，如果存在则返回，如果不存在则继续。之所以选择偏移量最大，这是因为偏移量越小，和**Master**的数据越不接近，现在**Master**挂掉了，说明这个偏移量小的机器数据可能存在问题，这就是为什么选择偏移量最大的**Slave**的原因。如果发现偏移量都一样，这个时候 **Redis** 会默认选择 **runid** 最小的节点。

生产环境部署技巧：

*   **Sentinel**节点不应该部署在一台物理机器上。
    
    这里特意强调物理机是因为一台物理机做成了若干虚拟机或者现今比较流行的容器，它们虽然有不同的**IP**地址，但实际上它们都是同一台物理机，同一台物理机意味着如果这台机器有什么硬件故障，所有的虚拟机都会受到影响，为了实现**Sentinel**节点集合真正的高可用，请勿将**Sentinel**节点部署在同一台物理机器上。
    
*   部署至少三个且奇数个的**Sentinel**节点。通过增加**Sentinel**节点的个数提高对于故障判定的准确性，因为领导者选举需要至少一半加1个节点。

## Sentinel常见问题

哨兵集群在发现**master node**挂掉后会进行故障转移，也就是启动其中一个**slave node**为**master node**。在这过程中，可能会导致数据丢失的情况。

*   异步复制导致数据丢失
    
    因为**master->slave**的复制是异步，所以有可能部分还没来得及复制到**slave**就宕机了，此时这些部分数据就丢失了。
    
*   集群脑裂导致数据丢失
    
    脑裂，也就是说。某个**master**所在机器突然脱离了正常的网络，跟其它**slave**机器不能连接，但是实际上**master**还运行着。
    

造成的问题：

​ 此时哨兵可能就会认为**master**宕机了，然后开始选举，将其它**slave**切换成**master**。这时候集群里就会有2个**master**，也就是所谓的脑裂。此时虽然某个**slave**被切换成**master**，但是可能**client**还没来得及切换成新的**master**，还继续写向旧的**master**的数据可能就丢失了。因此旧**master**再次被恢复的时候，会被作为一个**slave**挂到新的**master**上去，自己的数据会被清空，重新从新的**master**复制数据。

怎么解决：

```
min-slaves-to-write 1
min-slaves-max-lag 10
```

要求至少有一个**slave**，数据复制和同步的延迟不能超过10s。

如果说一旦所有的**slave**，数据复制和同步的延迟都超过了10s，这个时候，**master**就不会再接收任何请求了。

上面两个配置可以减少异步复制和脑裂导致的数据丢失。

异步复制导致的数据丢失：

​ 在异步复制的过程当中，通过`min-slaves-max-lag`这个配置，就可以确保的说，一旦**slave**复制数据和**ack**延迟时间太长，就认为可能**master**宕机后损失的数据太多了，那么就拒绝写请求，这样就可以把**master**宕机时由于部分数据未同步到**slave**导致的数据丢失降低到可控范围内。

集群脑裂导致的数据丢失：

​ 集群脑裂因为**client**还没来得及切换成新的**master**，还继续写向旧的master的数据可能就丢失了通过`min-slaves-to-write`确保必须是有多少个从节点连接，并且延迟时间小于`min-slaves-max-lag`多少秒。

客户端需要怎么做：

​ 对于**client**来讲，就需要做些处理，比如先将数据缓存到内存当中，然后过一段时间处理，或者连接失败，接收到错误切换新的**master**处理。