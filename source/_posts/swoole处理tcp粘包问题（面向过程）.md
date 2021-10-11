---
title: Swoole处理Tcp粘包问题（面向过程）
tags: []
id: '31'
categories:
  - - Swoole
date: 2019-12-15 16:20:41
---

# TCP通信特点

* TCP 是流式协议没有消息边界，客户端向服务器端发送一次数据，可能会被服务器端分成多次收到。客户端向服务器端发送多条数据。服务器端可能一次全部收到。
* 保证传输的可靠性，顺序。
* TCP拥有拥塞控制，所以数据包可能会延后发送。

# TCP粘包

## 介绍

TCP 粘包是指发送方发送的若干包数据 到 接收方接收时粘成一包，从接收缓冲区看，后一包数据的头紧接着前一包数据的尾。

## 原因

发送方：发送方需要等缓冲区满才发送出去，造成粘包
接收方：接收方不及时接收缓冲区的包，造成多个包接收
下图为tcp协议在传输数据的过程：
![Tcp协议传输数据过程.jpg](https://imgconvert.csdnimg.cn/aHR0cDovL3Fpbml1Lmdhb2JpbnpoYW4uY29tLzIwMTkvMTIvMTUvNmU5YmM5M2I4N2RjNC5qcGc?x-oss-process=image/format,png)

## swoole处理粘包

* EOF结束协议：
EOF切割需要遍历整个数据包的内容，查找EOF，因此会消耗大量CPU资源。上手比较简单

* 固定包头+包体协议
长度检测协议，只需要计算一次长度，数据处理仅进行指针偏移，性能非常高


# 重现TCP粘包问题

服务端

```
<?php

//创建Server对象，监听 127.0.0.1:9501端口
$server = new Swoole\Server("127.0.0.1", 9501);

//监听连接进入事件
$server->on('Connect', function ($server, $fd) {
    echo "Client: Connect.\n";
});

//监听数据接收事件
$server->on('Receive', function ($server, $fd, $from_id, $data) {

    echo '接到消息'.$data.PHP_EOL;
    
});

//监听连接关闭事件
$server->on('Close', function ($serv, $fd) {
    echo "Client: Close.\n";
});

//启动服务器
$server->start();
```

客户端

```
<?php
$client = new swoole_client(SWOOLE_SOCK_TCP);
//连接到服务器
if (!$client->connect('127.0.0.1', 9501, 0.5)) die("connect failed.");



//向服务器发送数据


for ($i = 0; $i <= 100; $i++) $client->send('12345678');

//从服务器接收数据
$data = $client->recv();
if (!$data) die("recv failed.");



//关闭连接
$client->close();
```

如下图：此时数据并不是完整的数据
![服务端.png](https://imgconvert.csdnimg.cn/aHR0cDovL3Fpbml1Lmdhb2JpbnpoYW4uY29tLzIwMTkvMTIvMTUvODg1MGU2MWMyMjUwZi5wbmc?x-oss-process=image/format,png)

# 固定包头+包体协议

服务端

```
<?php

//创建Server对象，监听 127.0.0.1:9501端口
$server = new Swoole\Server("127.0.0.1", 9501);

$server->set(
    [
        'open_length_check' => true, // 打开包长检测特性
        'package_max_length' => 1024 * 1024 * 2, // 包最大长度
        'package_length_type' => 'N', // 长度值的类型
        'package_length_offset' => 0, // 从第几个字节开始计算长度
        'package_body_offset' => 4, // 长度值在包头的第几个字节
    ]
);

//监听连接进入事件
$server->on('Connect', function ($server, $fd) {
    echo "Client: Connect.\n";
});

//监听数据接收事件
$server->on('Receive', function ($server, $fd, $from_id, $data) {
    //解包 去掉包头数据
    $body = unpack('N',$data);
    $content = substr($data,4,reset($body)); // 这里的 4 的计算公式  N 为32字节序 1字节等于8bit 32 / 8 = 4
    var_dump($content);

    //回复客户端消息 如果客户端设置了 open_length_check 也需要将数据打包
    $message = 'this is Server !';
    $send = pack('N',strlen($message)).$message;


    $server->send($fd,$send);
});

//监听连接关闭事件
$server->on('Close', function ($serv, $fd) {
    echo "Client: Close.\n";
});

//启动服务器
$server->start();
```

客户端

```
<?php
$client = new swoole_client(SWOOLE_SOCK_TCP);
$client->set(
    [
        'open_length_check' => true, // 打开包长检测特性
        'package_max_length' => 1024 * 1024 * 2, // 包最大长度
        'package_length_type' => 'N', // 长度值的类型
        'package_length_offset' => 0, // 从第几个字节开始计算长度
        'package_body_offset' => 4, // 长度值在包头的第几个字节
    ]
);
//连接到服务器
if (!$client->connect('127.0.0.1', 9501, 0.5)) die("connect failed.");


$content = 'Hello world';
$body = pack('N', strlen($content)) . $content;

//向服务器发送数据
if (!$client->send($body)) die("send failed.");


//从服务器接收数据
$data = $client->recv();
if (!$data) die("recv failed.");


//解包 去掉包头数据
$body = unpack('N',$data);
$content = substr($data,4,reset($body)); // 这里的 4 的计算公式  N 为32字节序 1字节等于8bit 32 / 8 = 4 
var_dump($content);

//关闭连接
$client->close();
```

>swoole-version 4.4.12

* * *

