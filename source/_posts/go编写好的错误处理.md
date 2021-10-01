---
title: Go编写好的错误处理
tags: []
id: '56'
categories:
  - - uncategorized
date: 2020-09-20 19:39:29
---

Go的错误机制：

*   没有异常机制
*   `error` 类型实现了 `error` 接口
    
    ```
    type error interface {
      Error() string
    }
    ```
    
*   可以通过 `errors.News` 来快速创建错误实例
    
    ```
    errors.News("n must be in the range [0,10]")
    ```
    

拿Fibonacci举例：

```
func GetFibonacci(n int) []int {
    fibList := []int{1, 2}

    for i := 2; i < n; i++ {
        fibList = append(fibList, fibList[i-2]+fibList[i-1])
    }
    return fibList
}

func TestGetFibonacci(t *testing.T) {
    t.Log(GetFibonacci(10))
    t.Log(GetFibonacci(-10))
    /** 运行结果
    === RUN   TestGetFibonacci
        TestGetFibonacci: err_test.go:15: [1 2 3 5 8 13 21 34 55 89]
        TestGetFibonacci: err_test.go:21: [1 2]
    --- PASS: TestGetFibonacci (0.00s)
    */
}
```

> 可以看到没有对入参进行校验

现在做下校验：

```
func GetFibonacci(n int) ([]int, error) {
    if n < 2  n > 100 {
        return nil, errors.New("n should be in [2,100]")
    }
    fibList := []int{1, 2}

    for i := 2; i < n; i++ {
        fibList = append(fibList, fibList[i-2]+fibList[i-1])
    }
    return fibList, nil
}

func TestGetFibonacci(t *testing.T) {
    // 如果有错误进行错误输出
    if v, err := GetFibonacci(-10); err != nil {
        t.Error(err)
    } else {
        t.Log(v)
    }
    /** 运行结果
    === RUN   TestGetFibonacci
        TestGetFibonacci: err_test.go:22: n should be in [2,100]
    --- FAIL: TestGetFibonacci (0.00s)
    */
}
```

假设现在有个需求，返回的值是太小了还是太大了，返回不同的错误，最简单的方法直接改造`GetFibonacci`：

```
func GetFibonacci(n int) ([]int, error) {
    if n < 2 {
        return nil, errors.New("n should be not less than 2")
    }

    if n > 100 {
        return nil, errors.New("n should be not larger than 100")
    }

    fibList := []int{1, 2}

    for i := 2; i < n; i++ {
        fibList = append(fibList, fibList[i-2]+fibList[i-1])
    }
    return fibList, nil
}
```

如果区分错误类型，依靠字符串去匹配简直太麻烦还容易出错，最常见的解决方法，定义两个预置的错误：

```
var LessThanTwoError = errors.New("n should be not less than 2")
var LargerThenHundredError = errors.New("n should be not larger than 100")

func GetFibonacci(n int) ([]int, error) {
    if n < 2 {
        return nil, LessThanTwoError
    }

    if n > 100 {
        return nil, LargerThenHundredError
    }

    fibList := []int{1, 2}

    for i := 2; i < n; i++ {
        fibList = append(fibList, fibList[i-2]+fibList[i-1])
    }
    return fibList, nil
}

func TestGetFibonacci(t *testing.T) {
    // 如果有错误进行错误输出
    if v, err := GetFibonacci(-10); err != nil {
        // 假如调用者需要判断错误的就比较简单了
        if err == LessThanTwoError {
            fmt.Println("It is less.")
        }
        t.Error(err)
    } else {
        t.Log(v)
    }
    /** 运行结果
    === RUN   TestGetFibonacci
    It is less.
        TestGetFibonacci: err_test.go:36: n should be not less than 2
    --- FAIL: TestGetFibonacci (0.00s)
    */
}
```

总结：

*   定义不同的错误变量，以便于判断错误类型
*   及早失败，避免嵌套，提高代码可读性

## panic和recover

### panic

panic：

*   `panic` 用于不可以恢复的错误
*   `panic` 退出前会执行 `defer` 指定的内容

panic vs. os.Exit：

*   `os.Exit` 退出时不会调用 `defer` 指定的函数
*   `os.Exit` 退出时不输出当前调用栈的信息

```
func TestExit(t *testing.T) {
    fmt.Println("Start")
    os.Exit(-1)
    /** 运行结果
    === RUN   TestExit
    Start

    Process finished with exit code 1
    */
}

func TestPanic(t *testing.T) {
    defer func() {
        fmt.Println("Finally!")
    }()
    fmt.Println("Start")
    panic(errors.New("Something wrong!"))
    /** 运行结果：
    === RUN   TestPanic
    Start
    Finally!
    --- FAIL: TestPanic (0.00s)
    panic: Something wrong! [recovered]
        panic: Something wrong!

    goroutine 6 [running]:
    testing.tRunner.func1.1(0x1119860, 0xc000046510)
        /usr/local/Cellar/go/1.14.2_1/libexec/src/testing/testing.go:940 +0x2f5
    testing.tRunner.func1(0xc00011a120)
        /usr/local/Cellar/go/1.14.2_1/libexec/src/testing/testing.go:943 +0x3f9
    panic(0x1119860, 0xc000046510)
        /usr/local/Cellar/go/1.14.2_1/libexec/src/runtime/panic.go:969 +0x166
    command-line-arguments.TestPanic(0xc00011a120)
        /Users/gaobinzhan/Documents/Go/learning/src/test/err_test.go:65 +0xd7
    testing.tRunner(0xc00011a120, 0x114afa0)
        /usr/local/Cellar/go/1.14.2_1/libexec/src/testing/testing.go:991 +0xdc
    created by testing.(*T).Run
        /usr/local/Cellar/go/1.14.2_1/libexec/src/testing/testing.go:1042 +0x357

    Process finished with exit code 1
    */
}
```

### recover

大家在写c++或者php代码的时候，总有一种习惯不希望这个程序被中断或者退出，用来捕获。

php代码：

```
try {

} catch (\Throwable $throwable) {
    
}
```

c++ 代码：

```
try{
  ...
}catch(...){
  
}
```

go代码：

```
defer func(){
  if err := recover(); err != nil {
    // 恢复错误
  }
}()
```

```
func TestRecover(t *testing.T) {
    defer func() {
        if err := recover(); err != nil {
            // 没有写错误恢复 只是打印出来了
            fmt.Println("recovered from", err)
        }
    }()
    fmt.Println("Start")
    panic(errors.New("Something wrong!"))
    /** 运行结果：
    === RUN   TestRecover
    Start
    recovered from Something wrong!
    --- PASS: TestRecover (0.00s)
    */
}
```

最常见的"错误恢复"：

```
defer func() {
  if err := recover(); err != nil {
    log.Error("recovered panic",err)
  }
}()
```

当心！`recover` 成为恶魔：

*   形成僵尸服务进程，导致 health check 失效。
*   “Let it Crash!” 往往是我们恢复不确定性错误的最好方法。

就如上常见的“错误恢复”只是记录了一下，这样的恢复方式是非常危险的。

一定要当心我们自己 `recover` 在做的事，因为我们 `recover` 的时候并不去检测错误到底发生了什么错误，而是简单的记录了一下或者忽略。

这时候可能是系统里面的某些核心资源已经消耗完了，我们这样把它强制恢复掉，其实系统依然不能够正常地工作的，还是导致我们的一些健康检查程序 health check 没有办法检查出当前系统的问题。

因为很多的这种 health check 只是检查当前的系统进程在还是不在，因为我们的进程是在的，所以就会导致一种僵尸服务进程，它好像活着，但它也不能提供服务。

这种情况下个人认为倒不如采用一种可恢复的设计模式其中的一种叫 `Let it Crash` ，干脆 `Crash`掉，一旦`Crash`掉 守护进程 ，就会帮我们的服务进程重新提起来。