---
title: 消息中间件Kafka - PHP操作使用Kafka
tags: []
id: '19'
categories:
  - - Kafka
date: 2019-08-27 17:40:57
---

# PHP使用Kafka

>**我们需要安装[libkafka](https://github.com/edenhill/librdkafka)和[rdkafka](https://github.com/arnaud-lb/php-rdkafka)**

## 安装libkafka

1. **下载**

   去GitHub上克隆下来

    `git clone https://github.com/edenhill/librdkafka.git`

2. **安装**


   `cd librdkafka/`

    `./configure && make && make install`

   安装成功界面 没有报错就是安装成功
    
    ![14c1ce5d9719d16db1f2de5e9eed9553.png](https://img-blog.csdnimg.cn/20190320100758703.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQxMzIwMjgx,size_16,color_FFFFFF,t_70)
    
    
## 安装rdkafka

1. **下载**

    `git clone https://github.com/arnaud-lb/php-rdkafka`

    `cd php-rdkafka/`
    
2. **为php安装扩展**


    在php-rdkafka这个目录下

    `phpize`

    然后会生成源代码安装的脚本
    
    把php-config的位置改成自己php-config的位置
    
    ` ./configure --with-php-config=/usr/local/php/bin/php-config`

    编译安装
    
    `make && make install`
    
    成功后会出现一个文件夹
    
    ![70c28ed68f37346a84abe96a2fe03aae.png](https://img-blog.csdnimg.cn/20190320100828577.png)
    
    这个位置就是保存的我们刚刚安装的扩展
    
    进入该目录
    
    `cd /usr/local/php/lib/php/extensions/no-debug-non-zts-20170718/`
    
    会发现出现个rdkafka.so文件
    
    ![2bb11d41b5e08b42a3fad4fc181eb791.png](https://img-blog.csdnimg.cn/20190320100957658.png)

    修改php.ini文件加入  这里的路径就是写自己rdkafka.so文件的路径
    
    `extension=/usr/local/php/lib/php/extensions/no-debug-non-zts-20170718/rdkafka.so `
    重启php
    
   ` php-m`
   
   出现rdkafka就是安装成功
   
    ![fe40f168197be21fb00db6d06428df24.png](https://img-blog.csdnimg.cn/20190320101008302.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQxMzIwMjgx,size_16,color_FFFFFF,t_70)

## php操作kafka

>**运行前先开启我们的zookeeper和kafka 上篇文章有如何开启**

1. **运行producer**
    kafka默认端口9092
    
    `vim producer.php`
    
        <?php
           $rk = new RdKafka\Producer();
           $rk->setLogLevel(LOG_DEBUG);
           $rk->addBrokers("ip:9092");       
           $topic = $rk->newTopic("test");
           $topic->produce(RD_KAFKA_PARTITION_UA, 0, "要发送的消息");
                  
                  
  2. **运行consumer**
    `vim consumer.php`
    
          <?php
             $rk = new RdKafka\Consumer();
             $rk->setLogLevel(LOG_DEBUG);
             $rk->addBrokers("ip");
             $topic = $rk->newTopic("test");
             $topic->consumeStart(0, RD_KAFKA_OFFSET_BEGINNING);
             while(true){
                 sleep(1);
                 $msg = $topic->consume(0, 1000);
                 if ($msg) {
                     echo $msg->payload, "\n";
                 }          
             }    
             
             
     开启两个窗口一个运行consumer 一个运行producer 
     
       `php consumer.php`
       `php producer.php`
    
     会发现我们已经简单的会使用kafka了。
