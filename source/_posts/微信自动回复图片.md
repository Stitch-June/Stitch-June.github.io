---
title: 微信自动回复图片
tags: []
id: '53'
categories:
  - - uncategorized
date: 2020-09-20 19:30:36
---

在微信开发的页面上，设置好触发的关键词，及触发后跳转到指定的接口地址，如:http://www.gaobinzhan.com/picture.php  
然后在网站服务器上创建picture.php文件，文件代码如下：

这样，在微信服务号上输入对应的关键字，服务号上就会返回对应的图片。

MediaID的获取方法：登陆微信公众平台->开发者工具->在线接口调试工具

**接口类型选：基础支持**

先获取access\_token

access\_token每次登陆都会变更

获取access\_token后，接口列表选择多媒体文件上传接口  
填入access\_token，type选择image，media选择要回复的图片，图片上传成功后，就会返回一个MediaID，把它填入上面的代码中就可以了。