---
title: php yield关键字及协程实现
tags: []
id: '67'
categories:
  - - Php
date: 2020-04-27 11:35:17
---

# 迭代器

> 迭代是指反复执行一个过程，每执行一次叫做迭代一次

php提供了统一的迭代器接口，之前文章我已经写过了。
通过实现`Iterator`接口，可以自行决定如何遍历。

# 生成器

> 相比迭代器，生成器提供了更容易的方法来简单实现对象的迭代，性能开销和复杂性大大降低。

一个生成器函数看起来更像一个普通的函数，不同的是普通函数返回的是一个值，而生成器可以`yield`生成许多个值。

生成器`yield`关键字不是返回值，而是返回`Generator`对象，不能被实力化，且继承了`Iterator`接口。

生成器优点：

- 生成器会对php应用的性能有非常大的影响。

- 代码运行时，节省大量内存。

- 适合计算大量的数据。
  

#  颠覆常识的yield

大家都知道`range`函数创建一个包含指定范围的元素的数组。

```php
<?php

$start = 1;
$end = 99999999999;

range($start,$end); // PHP Warning:  range(): The supplied range exceeds the maximum array size: start=1 end=99999999999

function xrange($start, $end)
{
    $result = [];
    for ($i = $start; $i <= $end; $i++) {
        $result[] = $i;
    }
}
xrange(1, 99999999999);// Fatal error: Allowed memory size of 134217728 bytes exhausted (tried to allocate 134217736 bytes)
```

以上代码，创建1-99999999999的数组，`range`报错，用`for`来创建，会内存溢出。

接下来看个好玩的！！！

```php
<?php

$start = 1;
$end = 99999999999;


function xrange($start, $end)
{
    for ($i = $start; $i < $end; $i++) {
        yield $i;
    }
}

$result = xrange($start, $end);

echo $result->current(); // 输出 1

$result->next();

echo $result->current(); // 输出 2
```

却发现能够正常输出数值！

那下面这样呢？？？

```php
<?php

$start = 1;
$end = 3;


function yrange($start, $end)
{
    for ($i = $start; $i < $end; $i++) {
        echo '输出1' . PHP_EOL;
        yield $i;
        echo '输出2' . PHP_EOL;
    }
}

echo '输出这个对象！！！！' . PHP_EOL;
var_dump(yrange($start, $end));
echo '对象输出结束！！！！'.PHP_EOL.PHP_EOL;

echo '遍历一次开始！！！！'.PHP_EOL;
foreach (yrange($start, $end) as $value) {
    echo '遍历的数据'.$value . PHP_EOL;
    break; // 我们遍历一次 就停止循环
};
echo '遍历一次结束！！！！'.PHP_EOL.PHP_EOL;


echo '一直遍历开始！！！！'.PHP_EOL;
foreach (yrange($start, $end) as $value) {
    echo '遍历的数据'.$value . PHP_EOL;
};
echo '一直遍历结束！！！！'.PHP_EOL.PHP_EOL;
```

来看下运行结果：

```bash
输出这个对象！！！！
object(Generator)#1 (0) {
}
对象输出结束！！！！

遍历一次开始！！！！
输出1
遍历的数据1
遍历一次结束！！！！

一直遍历开始！！！！
输出1
遍历的数据1
输出2
输出1
遍历的数据2
输出2
一直遍历结束！！！！
```

是不是懵逼了！！

- 调用函数返回，却发现`for`竟然没有执行。
- 就遍历一次，发现只执行了`echo '输出1' . PHP_EOL;`，而且也没有循环3次。

`yield`就是这样，有`yield`的函数被称为生成器函数。



# yield实现协程

```php
<?php
function task1()
{
    for ($i = 0; $i < 3; $i++) {
        sleep(1); // 模拟耗时
        echo "发短信{$i}\n";
    }
}

function task2()
{
    for ($i = 0; $i < 3; $i++) {
        sleep(1); // 模拟阻塞
        echo "发邮件{$i}\n";
    }
}

task1();
task2();
```

以上代码可以看出，短信发完之后，才会发邮件，如果交替执行或者再添加任务应该怎么做呢。



## 多任务协作及调度器实现

为了实现我们的多任务调度，首先实现“任务”--一个用轻量级的包装的协程函数：

```php
<?php
class Task
{
    protected $taskId; // 任务id
    protected $coroutine; // 生成器
    protected $sendValue = null; // 生成器send值
    protected $beforeFirstYield = true; // 迭代的指针是否为第一个

    public function __construct($taskId, Generator $coroutine)
    {
        $this->taskId = $taskId;
        $this->coroutine = $coroutine;
    }

    public function getTaskId()
    {
        return $this->taskId;
    }

    public function setSendValue($sendValue)
    {
        $this->sendValue = $sendValue;
    }

    public function run()
    {
        if ($this->beforeFirstYield) {
            $this->beforeFirstYield = false;
            return $this->coroutine->current();
        } else {
            $retval = $this->coroutine->send($this->sendValue);
            $this->sendValue = null;
            return $retval;
        }
    }

    public function isFinished()
    {
        return !$this->coroutine->valid();
    }
}
```

如代码，一个任务就是用任务ID标记的一个协程(函数)。

 使用`setSendValue()`方法，你可以指定哪些值将被发送到下次的恢复(在之后你会了解到我们需要这个)。

`run()`函数确实没有做什么，除了调用`send()`方法的协同程序, 要理解为什么添加了一个 `beforeFirstYieldflag`变量， 需要考虑下面的代码片段：

```php
<?php
function gen() {
    yield 'foo';
    yield 'bar';
}
$gen = gen();
var_dump($gen->send('something'));
// 如之前提到的在send之前, 当$gen迭代器被创建的时候一个renwind()方法已经被隐式调用
// 所以实际上发生的应该类似:
//$gen->rewind();
//var_dump($gen->send('something'));
//这样renwind的执行将会导致第一个yield被执行, 并且忽略了他的返回值.
//真正当我们调用yield的时候, 我们得到的是第二个yield的值! 导致第一个yield的值被忽略.
//string(3) "bar"
```

通过添加 `beforeFirstYield`我们可以确定第一个yield的值能被正确返回。

调度器现在不得不比多任务循环要做稍微多点了，然后才运行多任务：

```php
<?php
class Scheduler {
    protected $maxTaskId = 0;
    protected $taskMap = []; // taskId => task
    protected $taskQueue;
    public function __construct() {
        $this->taskQueue = new SplQueue();
    }
    public function newTask(Generator $coroutine) {
        $tid = ++$this->maxTaskId;
        $task = new Task($tid, $coroutine);
        $this->taskMap[$tid] = $task;
        $this->schedule($task);
        return $tid;
    }
    public function schedule(Task $task) {
        $this->taskQueue->enqueue($task);
    }
    public function run() {
        while (!$this->taskQueue->isEmpty()) {
            $task = $this->taskQueue->dequeue();
            $task->run();
            if ($task->isFinished()) {
                unset($this->taskMap[$task->getTaskId()]);
            } else {
                $this->schedule($task);
            }
        }
    }
}
```

`newTask()`方法（使用下一个空闲的任务id）创建一个新任务，然后把这个任务放入任务`map`数组里，接着它通过把任务放入任务队列里来实现对任务的调度，接着`run()`方法扫描任务队列，运行任务。

如果一个任务结束了， 那么它将从队列里删除，否则它将在队列的末尾再次被调度。

让我们看看下面具有两个简单任务的调度器：

```php
<?php
function task1()
{
    for ($i = 0; $i < 3; $i++) {
        sleep(1); // 模拟耗时
        yield;
        echo "发短信{$i}\n";
    }
}

function task2()
{
    for ($i = 0; $i < 3; $i++) {
        sleep(1); // 模拟耗时
        yield;
        echo "发邮件{$i}\n";
    }
}

$scheduler = new Scheduler;
$scheduler->newTask(task1());
$scheduler->newTask(task2());
$scheduler->run();
```

两个任务都仅仅回显一条信息，然后使用yield把控制回传给调度器。输出结果如下：

```bash
发短信0
发邮件0
发短信1
发邮件1
发短信2
发邮件2
```

输出确实如我们所期望的：对前五个迭代来说,两个任务是交替运行的。

## 调度器通信

上面实现了协程封装，但是调度器和任务直接缺少了通信，进行重新封装，使协程当中能够获取当前的任务id,新增任务,以及杀死任务。

### 系统调用

先封装系统调用：

```php
<?php
class SystemCall {
    protected $callback;
    public function __construct(callable $callback) {
        $this->callback = $callback;
    }
    public function __invoke(Task $task, Scheduler $scheduler) {
        $callback = $this->callback;
        return $callback($task, $scheduler);
    }
}
```

它和其他任何可调用的对象(使用_invoke)一样的运行， 不过它要求调度器把正在调用的任务和自身传递给这个函数。
为了解决这个问题我们不得不微微的修改调度器的run方法：

```php
<?php
public function run() {
    while (!$this->taskQueue->isEmpty()) {
        $task = $this->taskQueue->dequeue();
        $retval = $task->run();
        if ($retval instanceof SystemCall) {
            $retval($task, $this);
            continue;
        }
        if ($task->isFinished()) {
            unset($this->taskMap[$task->getTaskId()]);
        } else {
            $this->schedule($task);
        }
    }
}
```

### 获取任务

编写获取任务id函数：

```php
<?php
function getTaskId() {
        return new SystemCall(function(Task $task, Scheduler $scheduler) {
            $task->setSendValue($task->getTaskId());
            $scheduler->schedule($task);
        });
    }
```

重新编写我们的例子：

```php
<?php
function task1()
{
    $tid = (yield getTaskId()); // <-- here's the syscall!
    for ($i = 0; $i < 3; $i++) {
        sleep(1); // 模拟耗时
        yield;
        echo "发短信{$i}\n";
    }
}

function task2()
{
    $tid = (yield getTaskId()); // <-- here's the syscall!
    for ($i = 0; $i < 3; $i++) {
        sleep(1); // 模拟耗时
        yield;
        echo "发邮件{$i}\n";
    }
}

$scheduler = new Scheduler;
$scheduler->newTask(task1());
$scheduler->newTask(task2());
$scheduler->run();
```

这段代码将给出与前一个例子相同的输出。请注意系统调用如何同其他任何调用一样正常地运行，只不过预先增加了yield。

### 新增任务

编写新增任务函数：

```php
<?php
function newTask(Generator $coroutine) {
    return new SystemCall(
        function(Task $task, Scheduler $scheduler) use ($coroutine) {
            $task->setSendValue($scheduler->newTask($coroutine));
            $scheduler->schedule($task);
        }
    );
}
```

### 杀死任务

编写杀死任务函数：

```php
<?php
function killTask($tid) {
    return new SystemCall(
        function(Task $task, Scheduler $scheduler) use ($tid) {
            $task->setSendValue($scheduler->killTask($tid));
            $scheduler->schedule($task);
        }
    );
}
```

同样我们也需要往调度器里面，增加一个方法：

```php
<?php
public function killTask($tid) {
    if (!isset($this->taskMap[$tid])) {
        return false;
    }
    unset($this->taskMap[$tid]);
    // This is a bit ugly and could be optimized so it does not have to walk the queue,
    // but assuming that killing tasks is rather rare I won't bother with it now
    foreach ($this->taskQueue as $i => $task) {
        if ($task->getTaskId() === $tid) {
            unset($this->taskQueue[$i]);
            break;
        }
    }
  	echo "任务 $tid 被杀死\n";
    return true;
}
```

运行结果：

```php
<?php
function childTask()
{
    $tid = (yield getTaskId());
    while (true) {
        echo "任务 $tid 执行\n";
        yield;
    }
}

function task1()
{
    $tid = (yield getTaskId()); // <-- here's the syscall!
    for ($i = 0; $i < 3; $i++) {
        echo "发短信{$i}\n";
        yield;
    }
}

function task2()
{
    $tid = (yield getTaskId()); // <-- here's the syscall!
    $childTId = (yield newTask(childTask()));
    for ($i = 0; $i < 3; $i++) {
        echo "发邮件{$i}\n";
        yield;
        if ($i == 2) {
            yield killTask($childTId);
        }
    }
}

$scheduler = new Scheduler;
$scheduler->newTask(task1());
$scheduler->newTask(task2());
$scheduler->run();
```

# swoole实现协程

```php
<?php
Swoole\Coroutine::create(function (){
    $start = time();
    $wait = new Swoole\Coroutine\WaitGroup();
    function task1()
    {
        for ($i = 0; $i < 3; $i++) {
            echo "发短信{$i}\n";
            Swoole\Coroutine::sleep(1); // 协程切换
        }
    }

    function task2()
    {
        for ($i = 0; $i < 3; $i++) {
            echo "发邮件{$i}\n";
            Swoole\Coroutine::sleep(1); // 协程切换
        }
    }
    $wait->add();
    Swoole\Coroutine::create('task1');
    $wait->add();
    Swoole\Coroutine::create('task2');
    $wait->wait();
    echo '耗时' . (time() - $start);
});
```

以上代码可看出，耗时为6s，但运行结果确实3s，这就体现了协程的好处。

下一篇文章会具体写出swoole协程的用法。

> 以上yield实现协程 部分内容来自于 https://www.laruence.com/2015/05/28/3038.html


* * *




