---
title: 'swoole—csp编程模型'
tags: []
id: '60'
categories:
  - - Swoole
date: 2020-04-28 10:21:00
---

# 协程

> 不需要操作系统参与，创建销毁和切换的成本非常低，遇到`io`会自动让出`cpu`执行权，交给其它协程去执行。

![协程执行流程](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9xaW5pdS5nYW9iaW56aGFuLmNvbS8yMDIwLzA0LzI3LzFkZGQ2ODIxY2UwYTkucG5n?x-oss-process=image/format,png)

# Swoole协程

非协程代码：

```php
<?php
$start = time();
for ($i = 0; $i < 500; $i++) {
    file_get_contents('http://www.easyswoole.com/');
    echo '任务' . $i . '完成' . PHP_EOL;
}
echo '非协程总耗时' . (time() - $start) . 's' . PHP_EOL;
```

执行结果：

```bash
任务495完成
任务496完成
任务497完成
任务498完成
任务499完成
非协程总耗时13s
```

协程代码：

```php
<?php
$start = time();
for ($i = 0; $i < 500; $i++) {
    Swoole\Coroutine::create(function () use ($i, $start) {
        $client = new Swoole\Coroutine\Http\Client('www.easyswoole.com', 80);
        $client->get('/');
        echo '任务' . $i . '完成' . '耗时' . (time() - $start) . 's' . PHP_EOL;
    });
}
```

执行结果：

```bash
任务389完成耗时1s
任务395完成耗时1s
任务434完成耗时1s
任务477完成耗时1s
任务469完成耗时1s
任务385完成耗时1s
任务498完成耗时1s
```

可以发现速度相当快，但任务的id，不是顺序执行的，这就是遇到了`io`，`swoole`底层自动切换让出`cpu`执行权。

# Channle

> 用于协程间通讯，支持多生产者协程和多消费者协程。底层自动实现了协程的切换和调度。

执行下面代码：

```php
$start = time();
function task1()
{
    Swoole\Coroutine::sleep(3); // 模拟io阻塞
    echo "发短信\n";
}

function task2()
{
    Swoole\Coroutine::sleep(3); // 模拟io阻塞
    echo "发邮件\n";
}

Swoole\Coroutine::create('task1');
Swoole\Coroutine::create('task2');
echo '总耗时' . (time() - $start) . 's' . PHP_EOL;
```

执行结果：

```bash
总耗时0s
发短信
发邮件
```

却发现以上代码，先执行的`echo '总耗时' . (time() - $start) . 's' . PHP_EOL;`。

 

要等待`task1`及`task2`执行成功后输出，该怎么半呢，这就利用了`channel`,来实现`csp`并发编程。



代码：

```php
<?php
Swoole\Coroutine::create(function (){
    $start = time();
    $channel = new Swoole\Coroutine\Channel();
    function task1($channel)
    {
        /** @var Swoole\Coroutine\Channel $channel */
        Swoole\Coroutine::sleep(3); // 模拟io阻塞
        $channel->push("发短信\n");
    }

    function task2($channel)
    {
        /** @var Swoole\Coroutine\Channel $channel */
        Swoole\Coroutine::sleep(3); // 模拟io阻塞
        $channel->push("发邮件\n");
    }

    Swoole\Coroutine::create('task1', $channel);
    Swoole\Coroutine::create('task2', $channel);

    for ($i = 0; $i < 2; $i++) {
        echo $channel->pop();
    }

    echo '总耗时' . (time() - $start) . 's' . PHP_EOL;
});
```

执行结果：

```bash
发短信
发邮件
总耗时3s
```

可以看到耗时3s，但我们在增加一个任务，`for`里面的`$i`就要修改，使得我们的代码非常繁琐，所以就有了`WaitGroup`。

像`channel`可以实现协程通信，依赖管理，协程同步。

实现连接池功能可以看我之前的文章，[传送门](https://blog.gaobinzhan.com/archives/36.html)

# WaitGroup

> 基于Channel实现的`Golang`的`sync.WaitGrup`功能。

方法：

- `add` 方法增加计数
- `done` 表示任务已完成
- `wait` 等待所有任务完成恢复当前协程的执行
- `WaitGroup` 对象可以复用，`add`、`done`、`wait` 之后可以再次使用

代码：

```php
<?php
Swoole\Coroutine::create(function () {

    $start = time();
    $waitGroup = new Swoole\Coroutine\WaitGroup();
    function task1($waitGroup)
    {
        /** @var WaitGroup $waitGroup */
        Swoole\Coroutine::sleep(3); // 模拟io阻塞
        echo "发短信\n";
        $waitGroup->done();;
    }

    function task2($waitGroup)
    {
        /** @var WaitGroup $waitGroup */
        Swoole\Coroutine::sleep(3); // 模拟io阻塞
        echo "发邮件\n";
        $waitGroup->done();

    }

    $waitGroup->add();
    Swoole\Coroutine::create('task1', $waitGroup);

    $waitGroup->add();
    Swoole\Coroutine::create('task2', $waitGroup);

    $waitGroup->wait();

    echo '总耗时' . (time() - $start) . 's' . PHP_EOL;
});
```

执行结果跟之前一样，也是好时3s，但是不是更简单了呢。



# Context

> 协程原有的异步逻辑同步化，但是在协程切换是隐式发生的，所有协程切换的前后不能保证全局遍历及`static`变量的一致性。

用`context`用协程id做隔离，来保存上下文内容。

代码复现：

```php
<?php
class Email
{
    static $email = null;
}

Swoole\Coroutine::create(function () {

    function task1($email)
    {
        Email::$email = $email;
        Swoole\Coroutine::sleep(3); // 模拟io阻塞
        echo "发邮件：" . Email::$email . PHP_EOL;
    }

    function task2($email)
    {
        Email::$email = $email;
        echo "发邮件：" . Email::$email . PHP_EOL;

    }

    Swoole\Coroutine::create('task1', '975975398@gmail.com');

    Swoole\Coroutine::create('task2', 'gaobinzhan@gmail.com');

});
```

从感觉啥觉得会输出两个邮箱地址，但其实：

```bash
发邮件：gaobinzhan@gmail.com
发邮件：gaobinzhan@gmail.com
```

这就是变量生命周期，需要注意，我们可以封装一个类来保存上下文。

```php
<?php
class Context
{
    /**
     * [
     *    'cid' => [  // 就是协程的id
     *        'key' => 'value' // 保存的全局变量的信息
     *    ]
     * ]
     * @var [type]
     */
    public static $pool = [];

    static function get($key)
    {
        $cid = Swoole\Coroutine::getuid();// 获取当前运行的协程id
        if ($cid < 0) {
            return null;
        }
        if (isset(self::$pool[$cid][$key])) {
            return self::$pool[$cid][$key];
        }
        return null;
    }

    static function put($key, $item)
    {
        $cid = Swoole\Coroutine::getuid();// 获取当前运行的协程id
        if ($cid > 0) {
            self::$pool[$cid][$key] = $item;
        }
    }

    static function delete($key = null)
    {
        $cid = Swoole\Coroutine::getuid();
        if ($cid > 0) {
            if ($key) {
                unset(self::$pool[$cid][$key]);
            } else {
                unset(self::$pool[$cid]);
            }
        }
        var_dump(self::$pool);
    }
}
```

运行以下代码：

```php
<?php
Swoole\Coroutine::create(function () {

    function task1($email)
    {
        Context::put('email',$email);
        Swoole\Coroutine::sleep(3); // 模拟io阻塞
        echo "发邮件：" . Context::get('email') . PHP_EOL;
    }

    function task2($email)
    {
        Context::put('email',$email);
        echo "发邮件：" . Context::get('email') . PHP_EOL;
    }

    Swoole\Coroutine::create('task1', '975975398@gmail.com');

    Swoole\Coroutine::create('task2', 'gaobinzhan@gmail.com');

    var_dump(Context::$pool);
});
```

运行结果：

```bash
发邮件：gaobinzhan@gmail.com
array(2) {
  [2]=>
  array(1) {
    ["email"]=>
    string(19) "975975398@gmail.com"
  }
  [3]=>
  array(1) {
    ["email"]=>
    string(20) "gaobinzhan@gmail.com"
  }
}
发邮件：975975398@gmail.com
```

可以看到，两个邮箱都输出成功了，但是我们的变量没有销毁，如何销毁呢，`Swoole`提供了`defer`方法，在协程关闭之前会调用`defer`。

```php
<?php
Swoole\Coroutine::create(function () {

    function task1($email)
    {
        Context::put('email',$email);
        Swoole\Coroutine::sleep(3); // 模拟io阻塞
        echo "发邮件：" . Context::get('email') . PHP_EOL;
        Swoole\Coroutine::defer(function (){
            Context::delete();
        });
    }

    function task2($email)
    {
        Context::put('email',$email);
        echo "发邮件：" . Context::get('email') . PHP_EOL;
        Swoole\Coroutine::defer(function (){
            Context::delete();
        });
    }

    Swoole\Coroutine::create('task1', '975975398@gmail.com');

    Swoole\Coroutine::create('task2', 'gaobinzhan@gmail.com');


});
```

运行结果：

```bash
发邮件：gaobinzhan@gmail.com
array(1) {
  [2]=>
  array(1) {
    ["email"]=>
    string(19) "975975398@gmail.com"
  }
}
发邮件：975975398@gmail.com
array(0) {
}
```

可以发现到最后为空，已经被清空掉了。

是不是觉得这样写很麻烦，以及不确定在什么时候销毁，然而`Swoole`提供的`Context`可以让协程退出后上下文自动清理 (如无其它协程或全局变量引用)。

代码实现：

```php
<?php
Swoole\Coroutine::create(function () {

    function task1($email)
    {
        $context = Swoole\Coroutine::getContext();
        $context->email = $email;
        Swoole\Coroutine::sleep(3); // 模拟io阻塞
        echo "发邮件：" . $context->email . PHP_EOL;
    }

    function task2($email)
    {
        $context = Swoole\Coroutine::getContext();
        $context->email = $email;
        echo "发邮件：" . $context->email . PHP_EOL;
    }

    Swoole\Coroutine::create('task1', '975975398@gmail.com');

    Swoole\Coroutine::create('task2', 'gaobinzhan@gmail.com');
    
});
```



* * *


