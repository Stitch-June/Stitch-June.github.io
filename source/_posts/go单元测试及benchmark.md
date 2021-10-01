---
title: Go单元测试及Benchmark
tags: []
id: '69'
categories:
  - - uncategorized
date: 2020-09-23 07:30:04
---

之前在刚开始写了如何编写测试程序。

内置单元测试框架：

```
func TestErrorInCode(t *testing.T) {
    fmt.Println("Start")
    t.Error("Error")
    fmt.Println("End")
    /** 运行结果：
    === RUN   TestErrorInCode
    Start
        TestErrorInCode: functions_test.go:25: Error
    End
    --- FAIL: TestErrorInCode (0.00s)
    */
}

func TestFatalInCode(t *testing.T) {
    fmt.Println("Start")
    t.Fatal("Error")
    fmt.Println("End")
    /** 运行结果：
    === RUN   TestFatalInCode
    Start
        TestFatalInCode: functions_test.go:38: Error
    --- FAIL: TestFatalInCode (0.00s)
    */
}
```

使用断言：

`go get -u github.com/stretchr/testify`

```
func square(op int) int {
    return op * op
}

func TestSquareWithAssert(t *testing.T) {
    inputs := [...]int{1, 2, 3}
    expected := [...]int{1, 4, 9}
    for i := 0; i < len(inputs); i++ {
        ret := square(inputs[i])
        assert.Equal(t, expected[i], ret)
    }
}
```

## Benchmark

文件名以下划线`_benchmark`结尾，方法名以`Benchmark`开头，参数为`b *testing.B`

```
// 利用+=连接
func TestConcatStringByAdd(t *testing.T) {
    assert := assert.New(t)
    elems := []string{"1", "2", "3", "4", "5"}
    ret := ""
    for _, elem := range elems {
        ret += elem
    }
    assert.Equal("12345", ret)
}

// 利用buffer连接
func TestConcatStringBytesBuffer(t *testing.T) {
    assert := assert.New(t)
    var buf bytes.Buffer
    elems := []string{"1", "2", "3", "4", "5"}
    for _, elem := range elems {
        buf.WriteString(elem)
    }
    assert.Equal("12345", buf.String())
}

func BenchmarkConcatStringByAdd(b *testing.B) {
    elems := []string{"1", "2", "3", "4", "5"}
    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        ret := ""
        for _, elem := range elems {
            ret += elem
        }
    }
    b.StopTimer()
}

func BenchmarkConcatStringBytesBuffer(b *testing.B) {
    elems := []string{"1", "2", "3", "4", "5"}
    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        var buf bytes.Buffer
        for _, elem := range elems {
            buf.WriteString(elem)
        }
    }
}
```

在命令行输入 `go test -bench=. -benchmem`

Windows 下使⽤ go test 命令⾏时，-bench=.应写为-bench="."

运行结果：

```
$ go test -bench=. -benchmem
goos: darwin
goarch: amd64
pkg: eighteen/benchmark
BenchmarkConcatStringByAdd-8             8982729               130 ns/op              16 B/op          4 allocs/op
BenchmarkConcatStringBytesBuffer-8      17703706                64.9 ns/op            64 B/op          1 allocs/op
PASS
ok      eighteen/benchmark      2.532s
```

使用 `buffer` 连接字符串的性能比 `+=` 要好很多。

## BDD

BDD in Go：

项⽬⽹站 ：

https://github.com/smartystreets/goconvey

安装：

`go get -u github.com/smartystreets/goconvey/convey`

启动 WEB UI ：

`$GOPATH/bin/goconvey`

```
func TestSpec(t *testing.T) {
    convey.Convey("Given 2 even numbers", t, func() {
        a := 2
        b := 4
        convey.Convey("When add the two numbers", func() {
            c := a + b
            convey.Convey("Then the result is still even", func() {
                convey.So(c%2, convey.ShouldEqual, 0)
            })
        })
    })
}
```

运行结果：

```
$ go test -v  bdd_spec_test.go 
=== RUN   TestSpec

  Given 2 even numbers 
    When add the two numbers 
      Then the result is still even ✔


1 total assertion

--- PASS: TestSpec (0.00s)
PASS
ok      command-line-arguments  0.006s
```

可以看到最后一步为 ✔