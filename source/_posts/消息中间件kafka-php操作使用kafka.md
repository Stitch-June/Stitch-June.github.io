---
title: 消息中间件Kafka - PHP操作使用Kafka
tags: []
id: '75'
categories:
  - - uncategorized
date: 2020-09-23 08:11:08
---

*   博主： gaobinzhan
*   发布时间：2019 年 03 月 21 日
*   1366次浏览
*   暂无评论
*   1527字数
*   分类： 消息中间件

\[TOC\]

> **我们需要安装libkafka和rdkafka**

## 安装libkafka

1.  **下载**
    
    去GitHub上克隆下来
    
    `git clone https://github.com/edenhill/librdkafka.git`
    
2.  **安装**
    
    `cd librdkafka/`
    
    `./configure && make && make install`
    
    安装成功界面 没有报错就是安装成功
    
    ![](http://qiniu.gaobinzhan.com/2019/11/02/0afdf2be1eead.png)
    

## 安装rdkafka

1.  **下载**
    
    `git clone https://github.com/arnaud-lb/php-rdkafka`
    
    `cd php-rdkafka/`
    
2.  **为php安装扩展**
    
    在php-rdkafka这个目录下
    
    `phpize`
    
    然后会生成源代码安装的脚本
    
    把php-config的位置改成自己php-config的位置
    
    `./configure --with-php-config=/usr/local/php/bin/php-config`
    
    编译安装
    
    `make && make install`
    
    成功后会出现一个文件夹
    
    ![](http://qiniu.gaobinzhan.com/2019/11/02/24854ed8d1d47.png)
    
    这个位置就是保存的我们刚刚安装的扩展
    
    进入该目录
    
    `cd /usr/local/php/lib/php/extensions/no-debug-non-zts-20170718/`
    
    会发现出现个rdkafka.so文件  
    ![](http://qiniu.gaobinzhan.com/2019/11/02/194c95c93593c.png)
    
    修改php.ini文件加入 这里的路径就是写自己rdkafka.so文件的路径
    
    `extension=/usr/local/php/lib/php/extensions/no-debug-non-zts-20170718/rdkafka.so`
    
    重启php
    
    `php-m`
    
    出现rdkafka就是安装成功
    
    ![](http://qiniu.gaobinzhan.com/2019/11/02/402b89b50a15e.png)
    

## php操作kafka

> **运行前先开启我们的zookeeper和kafka 上篇文章有如何开启**

1.  **运行producer**  
    kafka默认端口9092
    
    `vim producer.php`