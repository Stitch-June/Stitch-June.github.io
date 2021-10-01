---
title: 'Hello,Go'
tags: []
id: '60'
categories:
  - - uncategorized
date: 2020-09-20 19:53:18
---

Go的特点

*   只有25个关键字
*   强类型语言
*   垃圾回收
*   指针直接访问内存

开发环境构建

*   1.8之前必须设置
*   1.8之后没用设置，将使用默认值
*   扩展名必须为`go`

## 编写第一个go程序

hello.go

```
package main // 包，表明代码所在的模块

import (
    "fmt"
) // 引入代码依赖

// 功能实现
func main() {
    fmt.Println("Hello, World!")
}
```

两种运行方式：

```
$ go run hello.go
Hello, World!
```

```
$ go build hello.go
$ ls
hello           hello.go
$ ./hello
Hello, World!
```

应用程序入口：

*   必须是`main`包 `package main`
*   必须是`main` 方法 `func main()`
*   文件名不一定是`main.go`

退出返回值：

*   Go中`main`函数不支持任何返回值
*   通过`os.Exit`来返回状态

获取命令行参数：

*   Go中`main`函数不支持任何返回值
*   `main`函数不支持传入参数 func main(arg \[\]string)
*   在程序中直接通过`os.Args`获取命令行参数

```
package main // 包，表明代码所在的模块

import (
    "fmt"
    "os"
) // 引入代码依赖

// 功能实现
func main() {
    fmt.Println("Hello, World!")
    fmt.Println(os.Args[0], os.Args[1]) // 默认情况下 参数0返回可执行文件路径
    os.Exit(100)
}
```

```
$ go run hello.go gaobinzhan
Hello, World!
参数0：/var/folders/qr/9vkwk7xn5rzbtnmykx7sxyv00000gn/T/go-build024626496/b001/exe/hello
参数1：gaobinzhan
exit status 100
```

**公众号 : Tinkled**

![](http://qiniu.gaobinzhan.com/2019/12/14/f702d956e8211.jpg?imageView2/2/w/300)