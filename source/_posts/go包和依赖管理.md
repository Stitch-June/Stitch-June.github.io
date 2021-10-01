---
title: Go包和依赖管理
tags: []
id: '41'
categories:
  - - uncategorized
date: 2020-09-20 18:38:27
---

package：

*   基本复用模块单元
    
    以首字母大写来表明可被包外代码访问
    
*   代码的 package 可以和所在的目录不一致
*   同一目录里的 Go 代码的 package 要保持一致

需要把包目录加入到GOPATH

目录结构：

```
~/Documents/Go
- learning
    - src
        - fifteen
            - client
                - package_test.go
            - series
                - my_series.go
```

查看 go env

```
$ go env
GOPATH="/Users/gaobinzhan/Documents/Go/learning:/Users/gaobinzhan/Documents/Go"
```

可以看到这个目录已经加入`GOPATH`里了。

`my_series.go`

```
package series

// 首字母必须大写 才可被包外代码访问
func GetFibonacci(n int) ([]int, error) {
    fibList := []int{1, 2}

    for i := 2; i < n; i++ {
        fibList = append(fibList, fibList[i-2]+fibList[i-1])
    }
    return fibList, nil
}
```

`package_test.go`

```
package client

import (
    "fifteen/series"
    "testing"
)

func TestPackage(t *testing.T) {
    t.Log(series.GetFibonacci(5))
    /** 运行结果：
    === RUN   TestPackage
        TestPackage: package_test.go:9: [1 2 3 5 8] 
    --- PASS: TestPackage (0.00s)
    */
}
```

init方法：

*   在 `main` 被执行前，所有依赖的 `package` 的 `init` 方法都会被执行
*   不同包的 `init` 函数按照包导入的依赖关系决定执行顺序
*   每个包可以有多个 `init` 函数
*   包的每个源文件

下面修改文件

`my_series.go`

```
package series

import "fmt"

func init() {
    fmt.Println("init 1")
}

func init() {
    fmt.Println("init 2")
}

func GetFibonacci(n int) ([]int, error) {
    fibList := []int{1, 2}

    for i := 2; i < n; i++ {
        fibList = append(fibList, fibList[i-2]+fibList[i-1])
    }
    return fibList, nil
}
```

`package_test.go`

```
package client

import (
    "fifteen/series"
    "testing"
)

func TestPackage(t *testing.T) {
    t.Log(series.GetFibonacci(5))
    /** 运行结果：
    init 1
    init 2
    === RUN   TestPackage
        TestPackage: package_test.go:10: [1 2 3 5 8] 
    --- PASS: TestPackage (0.00s)
    */
}
```

获取远程package：

*   通过 go get 来获取远程依赖
    
    go get -u 强制从网络更新远程依赖
    
*   注意代码在 Github 上的组织形式，以适应 go get
    
    直接以代码路径开始，不要有 src
    

示例：go get -u https://github.com/easierway/concurrent\_map

代码：

```
package remote_package

import (
    cm "github.com/easierway/concurrent_map"
    "testing"
)

func TestConcurrentMap(t *testing.T) {
    m := cm.CreateConcurrentMap(99)
    m.Set(cm.StrKey("key"), 10)
    t.Log(m.Get(cm.StrKey("key")))
    /** 运行结果：
    === RUN   TestConcurrentMap
        TestConcurrentMap: remote_package_test.go:11: 10 true
    --- PASS: TestConcurrentMap (0.00s)
    */

}
```

## 依赖管理

Go未解决的依赖问题：

*   同一环境下，不同项目使用同一包的不同版本
*   无法管理对包的特定版本的依赖

vendor路径：

随着 Go 1.5 release 版本的发布，vendor ⽬录被添加到除了 GOPATH 和

GOROOT 之外的依赖⽬录查找的解决⽅案。在 Go 1.6 之前，你需要⼿动

的设置环境变量

查找依赖包路径的解决⽅案如下：

*   当前包下的 vendor ⽬录
*   向上级⽬录查找，直到找到 src 下的 vendor ⽬录
*   在 GOPATH 下⾯查找依赖包
*   在 GOROOT ⽬录下查

常用的依赖管理工具：

简单用一下

安装 gilde

`brew install glide`

删除我们刚刚 go get 下来的包 然后执行 glide init

![](http://qiniu.gaobinzhan.com/2020/05/20/2c92c86163d52.png)

然后会在目录下面生成一个 `glide.yaml`文件

执行 `glide install` 会生成 `vendor` 目录 里面就是我们的依赖包

执行原来的测试文件，依然可以执行成功。