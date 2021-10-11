---
title: Go常见架构模式的实现
tags: []
id: '83'
categories:
  - - Go
date: 2020-05-31 16:55:36
---

## 实现pipe-filter framework

![](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9xaW5pdS5nYW9iaW56aGFuLmNvbS8yMDIwLzA1LzMxL2I3MWU3ZjRjNDliMTQucG5n?x-oss-process=image/format,png)

Pipe-Filter 模式：

- ⾮常适合与数据处理及数据分析系统
- Filter封装数据处理的功能
- Pipe⽤于连接Filter传递数据或者在异步处理过程中缓冲数据流
- 进程内同步调⽤时，pipe演变为数据在⽅法调⽤间传递
- 松耦合：Filter只跟数据（格式）耦合

Filter和组合模式：

![](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9xaW5pdS5nYW9iaW56aGFuLmNvbS8yMDIwLzA1LzMxL2JjYmI5MWRiNTJhZGQucG5n?x-oss-process=image/format,png)

![](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9xaW5pdS5nYW9iaW56aGFuLmNvbS8yMDIwLzA1LzMxLzY4N2IxNWEzNGU5MmQucG5n?x-oss-process=image/format,png)

示例：

![](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9xaW5pdS5nYW9iaW56aGFuLmNvbS8yMDIwLzA1LzMxL2FjZjMyODk5NWI1NDcucG5n?x-oss-process=image/format,png)

简单示例代码：

`filter.go`

```go
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

```go
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

```go
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

![](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9xaW5pdS5nYW9iaW56aGFuLmNvbS8yMDIwLzA1LzMxLzMzYmU0MjlhMzRjYWEucG5n?x-oss-process=image/format,png)

- 特点

  - 易于扩展
  - 错误隔离
  - 保持架构⼀致性
- 要点
- 内核包含公共流程或通⽤逻辑
  - 将可变或可扩展部分规划为扩展点
- 抽象扩展点⾏为，定义接⼝
  - 利⽤插件进⾏扩展

示例：

![](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9xaW5pdS5nYW9iaW56aGFuLmNvbS8yMDIwLzA1LzMxLzhjZjE1NTllNjNlMDIucG5n?x-oss-process=image/format,png)

简单示例代码：

`agent.go`

```go
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
			case evtSeg[i] = <-agt.evtBuf:
			case <-agt.ctx.Done():
				return
			}
		}
		fmt.Println(evtSeg)
	}

}

func NewAgent(sizeEvtBuf int) *Agent {
	agt := Agent{
		collectors: map[string]Collector{},
		evtBuf:     make(chan Event, sizeEvtBuf),
		state:      Waiting,
	}

	return &agt
}

func (agt *Agent) RegisterCollector(name string, collector Collector) error {
	if agt.state != Waiting {
		return WrongStateError
	}
	agt.collectors[name] = collector
	return collector.Init(agt)
}

func (agt *Agent) startCollectors() error {
	var err error
	var errs CollectorsError
	var mutex sync.Mutex

	for name, collector := range agt.collectors {
		go func(name string, collector Collector, ctx context.Context) {
			defer func() {
				mutex.Unlock()
			}()
			err = collector.Start(ctx)
			mutex.Lock()
			if err != nil {
				errs.CollectorErrors = append(errs.CollectorErrors,
					errors.New(name+":"+err.Error()))
			}
		}(name, collector, agt.ctx)
	}
	if len(errs.CollectorErrors) == 0 {
		return nil
	}
	return errs
}

func (agt *Agent) stopCollectors() error {
	var err error
	var errs CollectorsError
	for name, collector := range agt.collectors {
		if err = collector.Stop(); err != nil {
			errs.CollectorErrors = append(errs.CollectorErrors,
				errors.New(name+":"+err.Error()))
		}
	}
	if len(errs.CollectorErrors) == 0 {
		return nil
	}

	return errs
}

func (agt *Agent) destoryCollectors() error {
	var err error
	var errs CollectorsError
	for name, collector := range agt.collectors {
		if err = collector.Destory(); err != nil {
			errs.CollectorErrors = append(errs.CollectorErrors,
				errors.New(name+":"+err.Error()))
		}
	}
	if len(errs.CollectorErrors) == 0 {
		return nil
	}
	return errs
}

func (agt *Agent) Start() error {
	if agt.state != Waiting {
		return WrongStateError
	}
	agt.state = Running
	agt.ctx, agt.cancel = context.WithCancel(context.Background())
	go agt.EventProcessGroutine()
	return agt.startCollectors()
}

func (agt *Agent) Stop() error {
	if agt.state != Running {
		return WrongStateError
	}
	agt.state = Waiting
	agt.cancel()
	return agt.stopCollectors()
}

func (agt *Agent) Destory() error {
	if agt.state != Waiting {
		return WrongStateError
	}
	return agt.destoryCollectors()
}

func (agt *Agent) OnEvent(evt Event) {
	agt.evtBuf <- evt
}
```

`agent_test.go`

```go
package microkernel

import (
	"context"
	"errors"
	"fmt"
	"testing"
	"time"
)

type DemoCollector struct {
	evtReceiver EventReceiver
	agtCtx      context.Context
	stopChan    chan struct{}
	name        string
	content     string
}

func NewCollect(name string, content string) *DemoCollector {
	return &DemoCollector{
		stopChan: make(chan struct{}),
		name:     name,
		content:  content,
	}
}

func (c *DemoCollector) Init(evtReceiver EventReceiver) error {
	fmt.Println("initialize collector", c.name)
	c.evtReceiver = evtReceiver
	return nil
}

func (c *DemoCollector) Start(agtCtx context.Context) error {
	fmt.Println("start collector", c.name)
	for {
		select {
		case <-agtCtx.Done():
			c.stopChan <- struct{}{}
			break
		default:
			time.Sleep(time.Millisecond * 50)
			c.evtReceiver.OnEvent(Event{c.name, c.content})
		}
	}
}

func (c *DemoCollector) Stop() error {
	fmt.Println("stop collector", c.name)
	select {
	case <-c.stopChan:
		return nil
	case <-time.After(time.Second * 1):
		return errors.New("failed to stop for timeout")
	}
}

func (c *DemoCollector) Destory() error {
	fmt.Println(c.name, "released resources.")
	return nil
}

func TestAgent(t *testing.T) {
	agt := NewAgent(100)
	c1 := NewCollect("c1", "1")
	c2 := NewCollect("c2", "2")
	agt.RegisterCollector("c1", c1)
	agt.RegisterCollector("c2", c2)
	if err := agt.Start(); err != nil {
		fmt.Printf("start error %v\n", err)
	}
	fmt.Println(agt.Start())
	time.Sleep(time.Second * 1)
	agt.Stop()
	agt.Destory()
}
```


