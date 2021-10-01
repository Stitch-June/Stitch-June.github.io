---
title: docker实现redis主从复制
tags: []
id: '59'
categories:
  - - uncategorized
date: 2020-09-20 19:49:58
---

在实际的场景当中单一节点的redis容易面临风险。  
比如:

*   机器故障。我们部署到一台 Redis 服务器，当发生机器故障时，需要迁移到另外一台服务器并且要保证数据是同步的。而数据是最重要的，如果你不在乎， 基本上也就不会使用 Redis 了。

要实现分布式数据库的更大的存储容量和承受高并发访问量，我们会将原来集中式数据库的数据分别存储到其他多个网络节点上。

Redis 为了解决这个单一节点的问题，也会把数据复制多个副本部署到其他节点上进行复制，实现 Redis的高可用，实现对数据的冗余备份，从而保证数据和服务 的高可用。

#### 什么是主从复制

*   主从复制，是指将一台Redis服务器的数据，复制到其他的Redis服务器。前者称为主节点(master)，后者称为从节点(slave),数据的复制是单向的，只能由主节点到 从节点。
*   默认情况下，每台Redis服务器都是主节点，且一个主节点可以有多个从节点(或没有从节点)，但一个从节点只能有一个主节点。

### 主从复制的作用

*   数据冗余:主从复制实现了数据的热备份，是持久化之外的一种数据冗余方式。
*   故障恢复:当主节点出现问题时，可以由从节点提供服务，实现快速的故障恢复;实际上是一种服务的冗余。
*   负载均衡:在主从复制的基础上，配合读写分离，可以由主节点提供写服务，由从节点提供读服务(即写Redis数据时应用连接主节点，读Redis数据时应用连接从节点) 分担服务器负载;尤其是在写少读多的场景下，通过多个从节点分担读负载，可以大大提高Redis服务器的并发量。
*   读写分离:可以用于实现读写分离，主库写、从库读，读写分离不仅可以提高服务器的负载能力，同时可根据需求的变化，改变从库的数量;
*   高可用基石:除了上述作用以外，主从复制还是哨兵和集群能够实施的基础，因此说主从复制是Redis高可用的基础。

#### 主从复制启用

从节点开启主从复制，有3种方式:

*   配置文件
    
    在从服务器的配置文件中加入:slaveof  
    不推荐使用 配置文件可被动态修改
    
*   启动命令
    
    redis-server启动命令后加入 --slaveof
    
*   客户端命令
    
    Redis服务器启动后，直接通过客户端执行命令:slaveof ，则该Redis实例成为从节点。  
    通过 info replication 命令可以看到复制的一些参数信息
    

#### 主从复制原理

主从复制的原理以及过程必须要掌握，这样我们才知道为什么会出现这些问题 主从复制过程大体可以分为3个阶段:连接建立阶段(即准备阶段)、数据同步阶段、命令传播阶段。在从节点执行 slaveof 命令后，复制过程便开始运作，下面图示大概可以看到， 从图中可以看出复制过程大致分为6个过程

![](http://qiniu.gaobinzhan.com/2019/12/19/55f5372ea5837.png)

### 构建

#### dockerfile构建redis镜像

DockerFile

```
FROM alpine
RUN sed -i 's/dl-cdn.alpinelinux.org/mirrors.aliyun.com/g' /etc/apk/repositories \
  && apk add  gcc g++ libc-dev  wget vim  openssl-dev make  linux-headers \
  && rm -rf /var/cache/apk/*

#通过选择更小的镜像，删除不必要文件清理不必要的安装缓存，从而瘦身镜像
#创建相关目录能够看到日志信息跟数据跟配置文件
RUN mkdir -p /usr/src/redis \
      && mkdir -p /usr/src/redis/data \
      && mkdir -p /usr/src/redis/conf \
      && mkdir -p /usr/src/redis/log   \
      && mkdir -p /var/log/redis


RUN wget  -O /usr/src/redis/redis-4.0.11.tar.gz  "http://download.redis.io/releases/redis-4.0.11.tar.gz" \
   && tar -xzf /usr/src/redis/redis-4.0.11.tar.gz  -C /usr/src/redis \
   && rm -rf /usr/src/redis/redis-4.0.11.tar.tgz

RUN cd /usr/src/redis/redis-4.0.11 &&  make && make PREFIX=/usr/local/redis install \
&& ln -s /usr/local/redis/bin/*  /usr/local/bin/  && rm -rf /usr/src/redis/redis-4.0.11

#COPY ./conf/redis.conf  /usr/src/redis/conf

CMD ["/usr/local/bin/redis-server","/usr/src/redis/conf/redis.conf"]
```

切换到当前dockfile文件目录下 执行命令

`docker build -t redis .`

等待构建完成就可以了

#### docker创建自定义网络及redis主从集群规划

执行自定义网络命令  
`docker network create --subnet=192.168.1.0/24 redis-network`

容器名称

容器IP地址

映射端口号

宿主机IP地址

服务运行模式

redis-master

192.168.1.2

6380->6379

127.0.0.1

master

redis-slave

192.168.1.3

6381->6379

127.0.0.1

slave

#### docker启动容器

在 /data/ 下面创建该目录结构

data和log下面文件忽略

![](http://qiniu.gaobinzhan.com/2019/12/19/45619a5aa919b.png)

目录代码

提取码：r8r1

下载完放到 /data 下面

用户可自行定义宿主机目录

master  
`docker run -itd --name redis-master --net redis-network -v /data/redis/master:/usr/src/redis -p 6380:6379 --ip 192.168.1.2 redis`

slave  
`docker run -itd --name redis-slave --net redis-network -v /data/redis/slave:/usr/src/redis -p 6381:6379 --ip 192.168.1.3 redis`

#### 测试主从复制

主从的配置文件 没有设置redis密码

开两个终端分别执行

`docker exec -it redis-master sh`

`docker exec -it redis-slave sh`

进入容器后执行(都要执行)

`redis-cli`

在slave执行

`SLAVEOF 192.168.1.2 6379`

`info replication`

![](http://qiniu.gaobinzhan.com/2019/12/19/1c2dc132f6731.png)

`master_link_status`为up就成功了！

当前是通过内网连接 端口号为6379

如果通过宿主机IP连接 端口号为6380

这时候在master进行操作 就可以看到slave的变化了！

![](http://qiniu.gaobinzhan.com/2019/12/19/639b41e07023f.png)