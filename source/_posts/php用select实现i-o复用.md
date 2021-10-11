---
title: php用select实现I/O复用
tags: []
id: '63'
categories:
  - - Php
date: 2019-12-24 16:20:18
---

### 前言
在Linux Socket服务器短编程时，为了处理大量客户的连接请求，需要使用非阻塞I/O和复用，select、poll和epoll是Linux API提供的I/O复用方式，其实I/O多路复用就是通过一种机制，可以监视多个描述符，一旦某个描述符就绪(一般是读就绪或者写就绪)，能够通知程序进行相应的读写操作。但select，poll，epoll本质上都是同步I/O，因为他们都需要在读写事件就绪后自己负责进行读写，也就是说这个读写过程是阻塞的。现在比较受欢迎的的nginx就是使用epoll来实现I/O复用支持高并发，所以理解好select，poll，epoll对于nginx如何应对高并发还是很有帮助的。

### select调用过程
![select](https://imgconvert.csdnimg.cn/aHR0cDovL3Fpbml1Lmdhb2JpbnpoYW4uY29tLzIwMTkvMTIvMjQvMjkzN2MzMWU2ZGMyMy5wbmc?x-oss-process=image/format,png)

### select缺点

1. 单个进程监控的文件描述符有限，通常为1024*8个文件描述符。

    当然可以改进，由于select采用轮询方式扫描文件描述符。文件描述符数量越多，性能越差。

2. 内核/用户数据拷贝频繁，操作复杂。

    select在调用之前，需要手动在应用程序里将要监控的文件描述符添加到fed_set集合中。然后加载到内核进行监控。用户为了检测时间是否发生，还需要在用户程序手动维护一个数组，存储监控文件描述符。当内核事件发生，在将fed_set集合中没有发生的文件描述符清空，然后拷贝到用户区，和数组中的文件描述符进行比对。再调用selecct也是如此。每次调用，都需要了来回拷贝。

3. 轮回时间效率低

    select返回的是整个数组的句柄。应用程序需要遍历整个数组才知道谁发生了变化。轮询代价大。

4. select是水平触发

    应用程序如果没有完成对一个已经就绪的文件描述符进行IO操作。那么之后select调用还是会将这些文件描述符返回，通知进程。
   
   
### 代码实现

Worker.php
```
<?php
/**
 * @author gaobinzhan <gaobinzhan@gmail.com>
 */

class Worker
{

    private $socket;

    public $onReceive;

    public $onConnect;

    public $onClose;

    private $socketList = [];

    private $events = [
        'connect' => 'onConnect',
        'receive' => 'onReceive',
        'close' => 'onClose'
    ];

    public function __construct($host, $port, $type)
    {
        $this->socket = stream_socket_server("{$type}://{$host}:{$port}");
        stream_set_blocking($this->socket,0); // Set non blocking
        $this->socketList[(int)$this->socket] = $this->socket;
        return $this->socket;
    }


    private function accept()
    {
        while (true) {
            $write = $except = [];
            $read = $this->socketList;
            stream_select($read,$write,$except,60);
            foreach ($read as $socket) $socket === $this->socket ? $this->createSocket() : $this->receive($socket);
        }
    }

    private function createSocket(){
        //Establish a connection with the client
        $client = stream_socket_accept($this->socket);
        (!empty($client && is_callable($this->onConnect))) && call_user_func_array($this->onConnect, [$this->socket, $client]);
        $this->socketList[(int)$client] = $client;
    }

    private function receive($client){

        $buffer = fread($client, 65535);
        if (empty($buffer) && (feof($client) || !is_resource($client))) { fclose($client); unset($this->socketList[(int)$client]); }
        !empty($buffer) && is_callable($this->onReceive) && call_user_func_array($this->onReceive, [$this->socket, $client, $buffer]);


        //because：IO Multiplexing
        /*$close = fclose($v);
        if (!empty($close) && is_callable($this->onClose)) call_user_func_array($this->onClose, [$v]);*/
    }

    public function send($client, $data)
    {
        $response = "HTTP/1.1 200 OK\r\n";
        $response .= "Content-Type: text/html;charset=UTF-8\r\n";
        $response .= "Connection: keep-alive\r\n";
        $response .= "Content-length: ".strlen($data)."\r\n\r\n";
        $response .= $data;
        fwrite($client, $response);
    }

    public function on($event, $callback)
    {
        $event = $this->events[$event] ?? null;
        $this->$event = $callback;
    }

    public function start()
    {
        $this->accept();
    }
}
```

server.php

```
<?php
/**
 * @author gaobinzhan <gaobinzhan@gmail.com>
 */

require 'Worker.php';


class Server{

    public $server;

    public function __construct($host, $port, $type)
    {
        $this->server = new Worker($host, $port, $type);
        $this->server->on('connect',[$this,"onConnect"]);
        $this->server->on('receive',[$this,"onReceive"]);
        $this->server->on('close',[$this,"onClose"]);

        $this->server->start();
    }

    public function onConnect($server,$fd){
        echo 'onConnect '.$fd.PHP_EOL;
    }

    public function onReceive($server,$fd,$data){
        echo 'onReceive '.$fd.PHP_EOL;
        $this->server->send($fd,'this is server !!!');
    }

    public function onClose($fd){
        echo 'onClose '.$fd.PHP_EOL;
    }
}
new Server('0.0.0.0','9501','tcp');
```

cli模式下运行server.php

浏览器第一次访问`http://127.0.0.1:9501`

发现第一次连接符为7

![第一次](https://imgconvert.csdnimg.cn/aHR0cDovL3Fpbml1Lmdhb2JpbnpoYW4uY29tLzIwMTkvMTIvMjQvNTNiZjA4MTYxNDRkMy5wbmc?x-oss-process=image/format,png)

第二访问 发现为7的连接符被复用了

![第二次](https://imgconvert.csdnimg.cn/aHR0cDovL3Fpbml1Lmdhb2JpbnpoYW4uY29tLzIwMTkvMTIvMjQvYjlkMjE1M2IyMjNlOC5wbmc?x-oss-process=image/format,png)

可以用ab测试工具 更能体现出io复用

`ab -c 100 -n 100000 -k http://127.0.0.1:9501/
`

* * *

