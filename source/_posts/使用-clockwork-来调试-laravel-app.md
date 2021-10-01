---
title: 使用 Clockwork 来调试 Laravel App
tags: []
id: '52'
categories:
  - - uncategorized
date: 2020-09-20 19:26:39
---

*   博主： gaobinzhan
*   发布时间：2019 年 01 月 06 日
*   895次浏览
*   暂无评论
*   857字数
*   分类： Laravel PHP

Clockwork 由两个部分组成:

Chrome 插件 Clockwork  
服务器端的 Composer Package Github 项目

### 安装

首先安装 Chrome 插件 Clockwork,然后可以根据 Github 的 readme 安装服务端 也可根据 以下操作

##### 1\. 运行命令

`composer require itsgoingd/clockwork`

##### 2.完成上面操作，修改 config/app.php 中 providers 数组

`Clockwork\Support\Laravel\ClockworkServiceProvider::class`

##### 3\. 修改 config/app.php 中 aliases 数组

`'Clockwork' => Clockwork\Support\Laravel\Facade::class,`

#### 效果图：

![](http://blog.gaobinzhan.com/uploads/article/20190106/ca976ed430a91ce56bec5df5ae1593f1.png)

Clockwork 由两个部分组成:

Chrome 插件 Clockwork  
服务器端的 Composer Package Github 项目

### 安装

首先安装 Chrome 插件 Clockwork,然后可以根据 Github 的 readme 安装服务端 也可根据 以下操作

##### 1\. 运行命令

`composer require itsgoingd/clockwork`

##### 2.完成上面操作，修改 config/app.php 中 providers 数组

`Clockwork\Support\Laravel\ClockworkServiceProvider::class`

##### 3\. 修改 config/app.php 中 aliases 数组

`'Clockwork' => Clockwork\Support\Laravel\Facade::class,`

#### 效果图：

![](http://blog.gaobinzhan.com/uploads/article/20190106/ca976ed430a91ce56bec5df5ae1593f1.png)