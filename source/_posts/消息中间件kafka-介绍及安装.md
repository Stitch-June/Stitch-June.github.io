---
title: 消息中间件Kafka - 介绍及安装
tags: []
id: '57'
categories:
  - - uncategorized
date: 2020-09-20 19:45:11
---

*   博主： gaobinzhan
*   发布时间：2019 年 03 月 20 日
*   916次浏览
*   暂无评论
*   1815字数
*   分类： 消息中间件

\[TOC\]

#### 优势

*   **高吞吐量**：非常普通的硬件Kafka也可以支持每秒数百万的消息
*   支持通过Kafka服务器和消费机集群来区分消息
*   支持Hadoop并行数据加载

#### 关键概念

*   **Broker**：Kafka集群中的一台或多台服务器统称为broker。
*   **Topic**：Kafka处理的消息源（feeds of messages）的不同分类。
*   **Partition**：Topic物理上的分组，一个topic可以分为多个partition，每个partition是一个有序的队列。partition中的每条消息都会被分配一个有序的id（offset）。
*   **Message**：消息，是通信的基本单位，每个producer可以向一个topic（主题）发布一些消息。
*   **Producers**：消息和数据生产者，向Kafka的一个topic发布消息的过程叫做producers。
*   **Consumers**：消息和数据的消费者，订阅topics并处理其发布的消息的过程叫做consumers。

#### 安装

1.  **下载** 先安装jdk 然后jdk的安装方式在elasticsearch的安装文章中有，这里就不写了
    
    kafka官网
    
    `wget https://www-us.apache.org/dist/kafka/2.1.1/kafka_2.11-2.1.1.tgz`
    
    解压
    
    `tar -xzvf kafka_2.11-2.1.1.tgz`
    
2.  **修改配置文件**
    
    `cd kafka_2.11-2.1.1/config`
    
    `zookeeper.properties` 是zookeeper的配置文件，默认端口号2181，可不做修改
    
    `server.properties` 是kafka配置文件，将 zookeeper.connect 这行 改为自己的zookeeper地址和端口号
    
    修改完成之后 返回kafka主目录
    
    `cd ..`
    
3.  **运行zookeeper和kafka**
    
    运行zookeeper
    
    `bin/zookeeper-server-start.sh config/zookeeper.properties`
    
    不要关闭此窗口 再开一个新窗口 重新进入kafka目录
    
    运行kafka
    
    `bin/kafka-server-start.sh config/server.properties`
    
4.  **运行producer和consumer**
    
    跟上步操作一样 不要关闭窗口 重新开 重新进入kafka目录
    
    创建一个topic为test
    
    把ip和port改为自己zookeeper的
    
    `bin/kafka-topics.sh --create --zookeeper ip:port --replication-factor 1 --partitions 1 --topic test`
    
    运行producer
    
    `bin/kafka-console-producer.sh --broker-list ip:port --topic test`
    
    跟上步操作一样 不要关闭窗口 重新开 重新进入kafka目录
    
    运行consumer
    
    `bin/kafka-console-consumer.sh --bootstrap-server ip:port --topic test --from-beginning`
    
    然后在producer发送信息 会发现 consumer的窗口会出现你发送的消息
    

\[TOC\]

#### 优势

*   **高吞吐量**：非常普通的硬件Kafka也可以支持每秒数百万的消息
*   支持通过Kafka服务器和消费机集群来区分消息
*   支持Hadoop并行数据加载

#### 关键概念

*   **Broker**：Kafka集群中的一台或多台服务器统称为broker。
*   **Topic**：Kafka处理的消息源（feeds of messages）的不同分类。
*   **Partition**：Topic物理上的分组，一个topic可以分为多个partition，每个partition是一个有序的队列。partition中的每条消息都会被分配一个有序的id（offset）。
*   **Message**：消息，是通信的基本单位，每个producer可以向一个topic（主题）发布一些消息。
*   **Producers**：消息和数据生产者，向Kafka的一个topic发布消息的过程叫做producers。
*   **Consumers**：消息和数据的消费者，订阅topics并处理其发布的消息的过程叫做consumers。

#### 安装

1.  **下载** 先安装jdk 然后jdk的安装方式在elasticsearch的安装文章中有，这里就不写了
    
    kafka官网
    
    `wget https://www-us.apache.org/dist/kafka/2.1.1/kafka_2.11-2.1.1.tgz`
    
    解压
    
    `tar -xzvf kafka_2.11-2.1.1.tgz`
    
2.  **修改配置文件**
    
    `cd kafka_2.11-2.1.1/config`
    
    `zookeeper.properties` 是zookeeper的配置文件，默认端口号2181，可不做修改
    
    `server.properties` 是kafka配置文件，将 zookeeper.connect 这行 改为自己的zookeeper地址和端口号
    
    修改完成之后 返回kafka主目录
    
    `cd ..`
    
3.  **运行zookeeper和kafka**
    
    运行zookeeper
    
    `bin/zookeeper-server-start.sh config/zookeeper.properties`
    
    不要关闭此窗口 再开一个新窗口 重新进入kafka目录
    
    运行kafka
    
    `bin/kafka-server-start.sh config/server.properties`
    
4.  **运行producer和consumer**
    
    跟上步操作一样 不要关闭窗口 重新开 重新进入kafka目录
    
    创建一个topic为test
    
    把ip和port改为自己zookeeper的
    
    `bin/kafka-topics.sh --create --zookeeper ip:port --replication-factor 1 --partitions 1 --topic test`
    
    运行producer
    
    `bin/kafka-console-producer.sh --broker-list ip:port --topic test`
    
    跟上步操作一样 不要关闭窗口 重新开 重新进入kafka目录
    
    运行consumer
    
    `bin/kafka-console-consumer.sh --bootstrap-server ip:port --topic test --from-beginning`
    
    然后在producer发送信息 会发现 consumer的窗口会出现你发送的消息