---
title: Go典型并发任务
tags: []
id: '61'
categories:
  - - uncategorized
date: 2020-09-20 19:58:51
---

\[toc\]

## 仅运行一次

最容易联想到的单例模式：

```
type Singleton struct {
}

var singleInstance *Singleton
var once sync.Once

func GetSingletonObj() *Singleton {
    once.Do(func() {
        fmt.Println("Create Obj")
        singleInstance = new(Singleton)
    })
    return singleInstance
}

func TestGetSingletonObj(t *testing.T) {
    var wg sync.WaitGroup
    for i := 0; i < 10; i++ {
        wg.Add(1)
        go func() {
            obj := GetSingletonObj()
            fmt.Printf("%x\n", unsafe.Pointer(obj))
            wg.Done()
        }()
    }
    wg.Wait()
    /** 运行结果：
    === RUN   TestGetSingletonObj
    Create Obj
    1269f78
    1269f78
    1269f78
    1269f78
    1269f78
    1269f78
    1269f78
    1269f78
    1269f78
    1269f78
    --- PASS: TestGetSingletonObj (0.00s)
    */
}
```

## 仅需任意任务完成

任务堆里面，只需任务一个完成就返回。

```
func runTask(id int) string {
    time.Sleep(10 * time.Millisecond)
    return fmt.Sprintf("the result is from %d", id)
}

func FirstResponse() string {
    numOfRunner := 10
    ch := make(chan string) // 非缓冲channel
    for i := 0; i < numOfRunner; i++ {
        go func(i int) {
            ret := runTask(i)
            ch
```