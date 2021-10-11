---
title: Swoole TCP和UDP（同步和异步）
tags: []
id: '31'
categories:
  - - Swoole
date: 2019-12-14 12:23:11
---

>Tcp: 舔狗行为 可靠 先连接然后发消息等待回复

>Udp: 渣男行为 不可靠 不需要建立连接 通信不需要一直保持

* tcp服务端
```
<?php

//创建Server对象，监听 127.0.0.1:9501端口
$serv = new Swoole\Server("127.0.0.1", 9501);

//监听连接进入事件
$serv->on('Connect', function ($serv, $fd) {
    echo "Client: Connect.\n";
});

//监听数据接收事件
$serv->on('Receive', function ($serv, $fd, $from_id, $data) {
    $serv->send($fd, "Server: ".$data);
});

//监听连接关闭事件
$serv->on('Close', function ($serv, $fd) {
    echo "Client: Close.\n";
});

//启动服务器
$serv->start();
```

* tcp同步客户端

```
<?php
$client = new swoole_client(SWOOLE_SOCK_TCP);

//连接到服务器
if (!$client->connect('127.0.0.1', 9501, 0.5))
{
    die("connect failed.");
}
//向服务器发送数据
if (!$client->send("hello world"))
{
    die("send failed.");
}
//从服务器接收数据
$data = $client->recv();
if (!$data)
{
    die("recv failed.");
}
echo $data.PHP_EOL;
//关闭连接
$client->close();

echo '这里是同步客户端！'.PHP_EOL;
```

* tcp异步客户端（只能在cli模式下运行）

```
<?php

$client = new Swoole\Async\Client(SWOOLE_SOCK_TCP);

$client->on("connect", function($cli) {
    $cli->send("GET / HTTP/1.1\r\n\r\n");
});
$client->on("receive", function($cli, $data){
    echo "Receive: $data";
    $cli->send(time()."\n");
    sleep(1);
});
$client->on("error", function($cli){
    echo "error\n";
});
$client->on("close", function($cli){
    echo "Connection close\n";
});
$client->connect('127.0.0.1', 9501);

echo '这里是异步客户端'.PHP_EOL;
```

* udp服务端

```
<?php
//创建Server对象，监听 127.0.0.1:9502端口，类型为SWOOLE_SOCK_UDP
$serv = new Swoole\Server("127.0.0.1", 9502, SWOOLE_PROCESS, SWOOLE_SOCK_UDP);

//监听数据接收事件
$serv->on('Packet', function ($serv, $data, $clientInfo) {
    $serv->sendto($clientInfo['address'], $clientInfo['port'], "Server ".$data);
    var_dump($clientInfo);
});

//启动服务器
$serv->start();
```

* udp客户端

```
<?php
$client = new Swoole\Client(SWOOLE_SOCK_UDP);

$client->sendto('127.0.0.1',9502,'哈哈哈');
$result = $client->recv();
echo $result;
```



>swoole-version 4.4.12

* * *


