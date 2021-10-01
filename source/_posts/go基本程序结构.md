---
title: Go基本程序结构
tags: []
id: '45'
categories:
  - - uncategorized
date: 2020-09-20 18:52:44
---

测试程序：

*   源码文件以`_test`结尾：`x x x_test.go`
*   测试方法名以Test开头：`func TestXXX(t *testing.T) {...}`

```
package test

import "testing"

func TestFirstTry(t *testing.T) {
    t.Log("My first try!")
}
```

实现Fibonacci数列

```
package test

import (
    "testing"
)

func TestFibonacciList(t *testing.T) {
    //var a int = 1 // 定义变量
    //var b int = 1
    var (
        a int = 1
        b int = 1
    ) // 这样也可以定义变量
    // a := 1 直接赋值
    // b := 1
    t.Log(a)
    for i := 0; i < 5; i++ {
        t.Log(" ", b)
        tmp := a
        a = b
        b = tmp + a
    }
}
```

## 变量及常量

变量

*   赋值可以进行自动类型推断
*   在一个赋值语句中可以对多个变量进行同时赋值

```
func TestExchange(t *testing.T) {
    a := 1
    b := 2
    //tmp := a
    //a = b
    //b = tmp
    a, b = b, a // 变量交换
    t.Log(a, b)
}
```

常量 进行快速 设置连续值

```
const (
    Monday = iota + 1
    Tuesday // 2
    Wednesday // 3
    Thursday // 4
    Friday // 5
    Saturday // 6
    Sunday // 7
)

const (
    Readable = 1 
```

`检查左边值是否大于右边值` 检查左边值是否大于等于右边值