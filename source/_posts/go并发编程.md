---
title: Go并发编程
tags: []
id: '33'
categories:
  - - uncategorized
date: 2020-09-20 18:05:09
---

**Thead** vs. **Groutine**

*   创建时默认的 **stack** 的大小
    
    *   JDK5 以后的 Java Thread stack 默认为1M
    *   Groutine 的 **Stack** 初始化大小为2k
*   和 KSE（Kernel Space Entity）的对应关系
    
    *   Java Thread 是 1:1
    *   Groutine 是 M:N

![](http://qiniu.gaobinzhan.com/2020/05/21/2e36143a05eb0.png)

**Go**的**GMP**调度：

![](http://qiniu.gaobinzhan.com/2020/05/21/acfb6c6dd2127.png)

**M**：系统线程

**P**：Go实现的协程处理器

**G**：协程

从图中可看出，**Processor** 在不同的系统线程中，每个 **Processor** 都挂着准备运行的协程队列。

**Processor** 依次运行协程队列中的协程。

这时候问题就来了，假如一个协程运行的时间特别长，把整个 **Processor** 都占住了，那么在队列中的协程是不是就会被延迟的很久？

在Go启动的时候，会有一个守护线程来去做一个计数，计每个 **Processor** 运行完成的协程的数量，当一段时间内发现，某个 **Processor** 完成协程的数量没有发生变化的时候，就会往这个正在运行的协程任务栈插入一个特别的标记，协程在运行的时候遇到非内联函数，就会读到这个标记，就会把自己中断下来，然后插到这个等候协程队列的队尾，切换到别的协程进行运行。

当某一个协程被系统中断了，例如说 **io** 需要等待的时候，为了提高整体的并发，**Processor** 会把自己移到另一个可使用的系统线程当中，继续执行它所挂的协程队列，当这个被中断的协程被唤醒完成之后，会把自己加入到其中某个 **Processor** 的队列里，会加入到全局等待队列中。

当一个协程被中断的时候，它在寄存器里的运行状态也会保存到这个协程对象里，当协程再次获得运行状态的时候，重写写入寄存器，继续运行。

话不多说，直接上代码，如何在代码里启动一个协程：

```
func TestGroutine(t *testing.T) {
    for i := 0; i < 10; i++ {
        // int 参数
        go func(i int) {
            fmt.Println(i)
        }(i) // 传入参数
    }
    // 有可能测试程序结束的非常快 加个等待
    time.Sleep(time.Millisecond * 50)
    /** 运行结果
    === RUN   TestGroutine
    1
    4
    5
    2
    6
    3
    0
    8
    9
    7
    --- PASS: TestGroutine (0.05s)
    */
}
```

## 共享内存并发机制

### Lock

如果你是 **Java** 或者 **C++** 程序员，那么以下代码非常常见，使用锁来进行并发控制（可惜我是个Phper）：

```
lock lock = ...;
lock.lock();
try{
  // process (thread-safe)
}catch(Exception ex){
  
}finally{
  lock.unlock();
}
```

同样Go也提供了这样的机 package sync：

**Mutex** 互斥锁

**RWLock** 读写锁

不使用锁的情况：

```
func TestCounter(t *testing.T) {
    counter := 0
    for i := 0; i < 5000; i++ {
        go func(i int) {
            counter++
        }(i)
    }
    time.Sleep(time.Second * 1)
    t.Logf("counter = %d", counter)
    /** 运行结果：
    === RUN   TestCounter
        TestCounter: share_memory_test.go:16: counter = 4627
    --- PASS: TestCounter (1.01s)
    */
}
```

可以发现结果与预期结果不一样，这是因为 `conuter` 变量在不同的协程里面去做自增，导致了一个并发的竞争条件，传统意义来讲就是一个不是线程安全的程序。要保证线程安全，就要对共享的内存进行锁保护。

```
func TestCounterThreadSafe(t *testing.T) {
    var mut sync.Mutex
    counter := 0
    for i := 0; i < 5000; i++ {
        go func(i int) {
            // defer 释放锁
            defer func() {
                mut.Unlock()
            }()
            // 加锁
            mut.Lock()
            counter++
        }(i)
    }
    time.Sleep(time.Second * 1)
    t.Logf("counter = %d", counter)
    /** 运行结果：
    === RUN   TestCounterThreadSafe
        TestCounterThreadSafe: share_memory_test.go:40: counter = 5000
    --- PASS: TestCounterThreadSafe (1.01s)
    */
}
```

这次就得到了预期结果。

### WaitGroup

等待所有协程完成，才能往下执行操作。

上面代码中，怕代码执行太快，所以加了 **sleep**。

但我们无法控制这个 **sleep** 需要睡眠时间。

下面来用 `WaitGroup`：

```
func TestCounterWaitGroup(t *testing.T) {
    var mut sync.Mutex
    var wg sync.WaitGroup
    counter := 0
    for i := 0; i < 5000; i++ {
        wg.Add(1) // 增加一个要等待的协程
        go func(i int) {
            // defer 释放锁
            defer func() {
                mut.Unlock()
            }()
            // 加锁
            mut.Lock()
            counter++
            wg.Done() // 一个协程完成了
        }(i)
    }
    wg.Wait() // 等待所有添加的协程完成 才继续向下运行
    t.Logf("counter = %d", counter)
    /** 运行结果：
    === RUN   TestCounterWaitGroup
        TestCounterWaitGroup: share_memory_test.go:66: counter = 5000
    --- PASS: TestCounterWaitGroup (0.00s)
    */
}
```

## CSP并发机制

有人可能会说不就是 **Actor Model** 嘛

![](http://qiniu.gaobinzhan.com/2020/05/21/f5c13d22f5cdf.png)

**CSP** vs. **Actor**

*   和 **Actor** 的直接通讯不同，**CSP**模式则是通过**Channel**进行通讯的，更松耦合一些。
*   **Go**中的**channel**是有容量限制并且独立于处理**Groutine**，而如**Erlang**，**Actor**模式中的**mailbox**容量是无限的，接收进程也总是被动地处理消息。

**Channel**

![](http://qiniu.gaobinzhan.com/2020/05/21/7b78643d8ddb1.png)

**Go**中**Channel**的基本机制：

*   上图左边（非缓冲**channel**）：
    
    通讯的两方必须同时在**channel**的两边，才能完成这次交互。任何一方不在，另一方就会被阻塞在那里等待，直到等到另一方才能完成这次交互。
    
*   上图右边（缓冲**channel**）：
    
    就是对这个**channel**设置容量，在未满的情况下，放消息的人就能放进去，如果满了，就会发生阻塞等待。
    
    等待拿消息的人去拿，空出来容量。反之，拿消息一样。
    

```
func service() string {
    time.Sleep(time.Millisecond * 50) // 模拟阻塞
    return "Done"
}

func otherTask() {
    fmt.Println("working on something else")
    time.Sleep(time.Millisecond * 100) // 模拟阻塞
    fmt.Println("Task is done.")
}

func TestService(t *testing.T) {
    fmt.Println(service())
    otherTask()
    /** 运行结果：
    === RUN   TestService
    Done
    working on something else
    Task is done.
    --- PASS: TestService (0.16s)
    */
}
```

由运行结果可知，完全是串行的，耗时为 0.16s

对 `service`进行改造，在调用的时候启动另外一个协程去执行，而不是阻塞当前写的协程。

```
func service() string {
    time.Sleep(time.Millisecond * 50) // 模拟阻塞
    return "Done"
}

func otherTask() {
    fmt.Println("working on something else")
    time.Sleep(time.Millisecond * 100) // 模拟阻塞
    fmt.Println("Task is done.")
}

func AsyncService() chan string {
    retCh := make(chan string) // 创建一个非缓冲string类型的channel
    //retCh := make(chan string, 1) // 创建一个容量为1 string类型的缓冲channel
    go func() {
        ret := service()
        fmt.Println("returned result.")
        retCh
```