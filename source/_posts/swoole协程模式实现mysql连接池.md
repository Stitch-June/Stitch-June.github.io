---
title: Swoole协程模式实现Mysql连接池
tags: []
id: '51'
categories:
  - - Swoole
date: 2020-03-24 03:41:32
---


# 连接池定义

永不断开，要求我们的这个程序是一个常驻内存的程序。数据库连接池`（Connection pooling）`是程序启 动时建立足够的数据库连接，并将这些连接组成一个连接池，由程序动态地对池中的连接进行申请，使用，释放。

# 为什么需要连接池？
当并发量很低的时候，连接可以临时建立，但当服务吞吐达到几百、几千的时候，频繁 `建立连接 Connect` 和 `销毁连接 Close` 就有可能会成为服务的一个瓶颈，那么当服务启动的时候，先建立好若干个连接并存放于一个队列中，当需要使用时从队列中取出一个并使用，使用完后再反还到队列去，而对这个队列数据结构进行维护的，就是连接池。

# 使用channel实现连接池

> 必须在协程模式下



Pool.php

```php
<?php


class PdoPool
{
    /**
     * @var Swoole\Coroutine\Channel
     */
    protected $channel;

    /**
     * @var int 最大连接数
     */
    protected $maxActive = 30;

    /**
     * @var int 最少连接数
     */
    protected $minActive = 10;

    /**
     * @var int 最大等待连接数
     */
    protected $maxWait = 200;

    /**
     * @var float 最大等待时间 -1 永不超时
     */
    protected $maxWaitTime = 1.2;

    /**
     * @var int 最大空闲时间s
     */
    protected $maxIdleTime = 5;

    /**
     * @var int 自动检查时间ms
     */
    protected $checkTime = 3000;

    /**
     * @var int 当前连接数
     */
    protected $count = 0;

    /**
     * @var self
     */
    protected static $instance = null;


    /**
     * PdoPool constructor.
     */
    private function __construct()
    {
        // 初始化连接池
        $this->init();
        // 定时器进行空闲连接释放
        $this->recovery();
    }

    /**
     * @return PdoPool|null
     */
    public static function getInstance()
    {
        if (!self::$instance instanceof self) {
            self::$instance = new self();
        }
        return self::$instance;
    }

    /**
     * 连接池初始化
     */
    protected function init()
    {
        for ($i = 0; $i < $this->minActive; $i++) {
            $connection = $this->getConnection();
            if ($connection) $this->channel->push($connection);
        }
    }

    public function getConnection(){
        return $this->getConnectionByChannel();
    }

    private function getConnectionByChannel()
    {
        // 创建Channel
        if ($this->channel === null) {
            $this->channel = new Swoole\Coroutine\Channel($this->maxActive);
        }

        // 小于连接池内最小连接数
        if ($this->count < $this->minActive) {
            return $this->createConnection();
        }

        // 取出连接
        $connection = null;
        if (!$this->channel->isEmpty()) {
            $connection = $this->popConnection();
        }

        //检测连接是否正常
        if ($connection !== null && $connection['connection'] instanceof PDO) {
            return $connection;
        }

        //未取出连接 判断是否大于最大连接数进行创建
        if ($this->count < $this->maxActive) {
            return $this->createConnection();
        }


        //查看协程挂起数
        $stats = $this->channel->stats();
        if ($this->maxWait > 0 && $stats['consumer_num'] >= $this->maxWait) {
            echo '协程挂起数已大于最大等待数' . PHP_EOL;
        }

        //重新取出连接
        $connection = $this->channel->pop($this->maxWaitTime);
        if ($connection == false) {
            echo '获取连接失败' . PHP_EOL;
        }
        return $connection;
    }

    private function createConnection()
    {
        // 因为堵塞问题 会造成当前连接数大于最大连接数 先进行++
        $this->count++;
        try {
            $connection = new PDO('mysql:host=localhost;dbname=test', 'root', 'gaobinzhan');
            $connection->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);
            return [
                'connection' => $connection,
                'lastUsedTime' => time()
            ];
        } catch (\Throwable $throwable) {
            //失败--
            $this->count--;
        }
    }


    private function popConnection()
    {
        while (!$this->channel->isEmpty()) {
            $connection = $this->channel->pop();
            return $connection;
        }
        return null;
    }

    public function freeConnection($connection)
    {
        //放入连接池
        $stats = $this->channel->stats();
        if ($stats['queue_num'] < $this->maxActive) {
            $connection['lastUsedTime'] = time();
            $this->channel->push($connection);
        }
    }

    /**
     * 自动回收空闲连接
     */
    private function recovery()
    {
        swoole_timer_tick($this->checkTime, function () {
            while ($this->count > $this->minActive && !$this->channel->isEmpty()) {
                $connection = $this->channel->pop($this->maxWaitTime);
                if (!$connection) {
                    continue;
                }
                if ((time() - $connection['lastUsedTime']) > $this->maxIdleTime) {
                    $this->count--;
                    $connection['connection'] = null;
                    echo "回收成功" . PHP_EOL;
                } else {
                    $this->channel->push($connection);
                }
            }
        });
    }

    private function __clone()
    {

    }

}
```



这里生成的是`Pdo连接池`同理可自行修改`createConnection`方法生成其它连接池

可用`AST语法树`进行多种连接池配置

大佬勿喷 嘿嘿！



* * *


