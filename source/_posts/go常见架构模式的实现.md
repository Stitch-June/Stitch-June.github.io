---
title: Go常见架构模式的实现
tags: []
id: '83'
categories:
  - - uncategorized
date: 2020-09-23 09:21:33
---

![](http://qiniu.gaobinzhan.com/2020/05/31/b71e7f4c49b14.png)

Pipe-Filter 模式：

*   ⾮常适合与数据处理及数据分析系统
*   Filter封装数据处理的功能
*   Pipe⽤于连接Filter传递数据或者在异步处理过程中缓冲数据流
*   进程内同步调⽤时，pipe演变为数据在⽅法调⽤间传递
*   松耦合：Filter只跟数据（格式）耦合

Filter和组合模式：

![](http://qiniu.gaobinzhan.com/2020/05/31/bcbb91db52add.png)

![](http://qiniu.gaobinzhan.com/2020/05/31/687b15a34e92d.png)

示例：

![](http://qiniu.gaobinzhan.com/2020/05/31/acf328995b547.png)

简单示例代码：

`filter.go`

```
// Package pipefilter is to define the interfaces and the structures for pipe-filter style implementation
package pipefilter

// Request is the input of the filter
type Request interface{}

// Response is the output of the filter
type Response interface{}

// Filter interface is the definition of the data processing components
// Pipe-Filter structure
type Filter interface {
    Process(data Request) (Response, error)
}
```

`split_filter.go`

```
package pipefilter

import (
    "errors"
    "strings"
)

var SplitFilterWrongFormatError = errors.New("input data should be string")

type SplitFilter struct {
    delimiter string
}

func NewSplitFilter(delimiter string) *SplitFilter {
    return &SplitFilter{delimiter}
}

func (sf *SplitFilter) Process(data Request) (Response, error) {
    str, ok := data.(string) //检查数据格式/类型，是否可以处理
    if !ok {
        return nil, SplitFilterWrongFormatError
    }
    parts := strings.Split(str, sf.delimiter)
    return parts, nil
}
```

`split_filter_test.go`

```
package pipefilter

import (
    "reflect"
    "testing"
)

func TestStringSplit(t *testing.T) {
    sf := NewSplitFilter(",")
    resp, err := sf.Process("1,2,3")
    if err != nil {
        t.Fatal(err)
    }
    parts, ok := resp.([]string)
    if !ok {
        t.Fatalf("Repsonse type is %T, but the expected type is string", parts)
    }
    if !reflect.DeepEqual(parts, []string{"1", "2", "3"}) {
        t.Errorf("Expected value is {\"1\",\"2\",\"3\"}, but actual is %v", parts)
    }
}

func TestWrongInput(t *testing.T) {
    sf := NewSplitFilter(",")
    _, err := sf.Process(123)
    if err == nil {
        t.Fatal("An error is expected.")
    }
}
```

## 实现micro-kernel framework

![](http://qiniu.gaobinzhan.com/2020/05/31/33be429a34caa.png)

*   特点
    
*   要点
*   内核包含公共流程或通⽤逻辑
    
*   抽象扩展点⾏为，定义接⼝
    

示例：

![](http://qiniu.gaobinzhan.com/2020/05/31/8cf1559e63e02.png)

简单示例代码：

`agent.go`

```
package microkernel

import (
    "context"
    "errors"
    "fmt"
    "strings"
    "sync"
)

const (
    Waiting = iota
    Running
)

var WrongStateError = errors.New("can not take the operation in the current state")

type CollectorsError struct {
    CollectorErrors []error
}

func (ce CollectorsError) Error() string {
    var strs []string
    for _, err := range ce.CollectorErrors {
        strs = append(strs, err.Error())
    }
    return strings.Join(strs, ";")
}

type Event struct {
    Source  string
    Content string
}

type EventReceiver interface {
    OnEvent(evt Event)
}

type Collector interface {
    Init(evtReceiver EventReceiver) error
    Start(agtCtx context.Context) error
    Stop() error
    Destory() error
}

type Agent struct {
    collectors map[string]Collector
    evtBuf     chan Event
    cancel     context.CancelFunc
    ctx        context.Context
    state      int
}

func (agt *Agent) EventProcessGroutine() {
    var evtSeg [10]Event
    for {
        for i := 0; i < 10; i++ {
            select {
            case evtSeg[i] =
```