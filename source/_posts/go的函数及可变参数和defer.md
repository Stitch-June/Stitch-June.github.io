---
title: Go的函数及可变参数和defer
tags: []
id: '46'
categories:
  - - uncategorized
date: 2020-09-20 18:56:12
---

函数是一等公民：

*   可以有多个返回值
*   所有参数都是值传递：`slice`、`map`、`channel` 会有传引用的错觉
*   函数可以作为变量的值
*   函数可以作为参数和返回值

```
func returnMultiValues() (int, int) {
    // 返回两个值
    return rand.Intn(10), rand.Intn(20)
}

func TestFn(t *testing.T) {
    a, b := returnMultiValues()
    t.Log(a, b) // 1 7
}
```

可变参数：

```
func Sum(ops ...int) int {
    ret := 0
    for _, op := range ops {
        ret += op
    }
    return ret
}

func TestVarParams(t *testing.T) {
    t.Log(Sum(1, 2, 3, 4))
    t.Log(Sum(1, 2, 3, 4, 5))
    /** 运行结果
    === RUN   TestVarParams
        TestVarParams: func_test.go:48: 10
        TestVarParams: func_test.go:49: 15
    --- PASS: TestVarParams (0.00s)
     */
}
```

defer：

在最后执行完执行，通常我们用于释放资源及释放锁

```
func TestDefer(t *testing.T) {
    defer func() {
        t.Log("Clean resources")
    }()
    t.Log("Started")
    // panic 手动触发宕机
    panic("Fatal error") // defer 仍然会执行
    /** 运行结果
    === RUN   TestDefer
        TestDefer: func_test.go:56: Started
        TestDefer: func_test.go:54: Clean resources
    --- FAIL: TestDefer (0.00s)
     */
}
```