---
title: Go典型并发任务
tags: []
id: '61'
categories:
  - - Go
date: 2020-05-23 19:44:28
---

## 仅运行一次

最容易联想到的单例模式：

```go
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

```go
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
			ch <- ret
		}(i)
	}
	return <-ch
}

func TestFirstResponse(t *testing.T) {
	t.Log(FirstResponse())
	/** 第一次运行结果：
	=== RUN   TestFirstResponse
	    TestFirstResponse: first_response_test.go:27: the result is from 0
	--- PASS: TestFirstResponse (0.01s)
	*/
	/** 第二次运行结果：
	=== RUN   TestFirstResponse
	    TestFirstResponse: first_response_test.go:27: the result is from 3
	--- PASS: TestFirstResponse (0.01s)
	*/
}
```

因为协程的调度机制，所以返回结果不一样。

但这样是存在很大的问题，修改`TestFirstResponse`：

```go
func TestFirstResponse(t *testing.T) {
	t.Log("Before:", runtime.NumGoroutine()) // 获取协程数量
	t.Log(FirstResponse())
	time.Sleep(time.Second * 1)
	t.Log("After:", runtime.NumGoroutine()) // 获取协程数量
	/** 运行结果：
	=== RUN   TestFirstResponse
	    TestFirstResponse: first_response_test.go:28: Before: 2
	    TestFirstResponse: first_response_test.go:29: the result is from 6
	    TestFirstResponse: first_response_test.go:30: After: 11
	--- PASS: TestFirstResponse (0.01s)
	*/
}
```

因为使用的是非缓冲**channel**，`FirstResponse`方法只取走了一次，往**channel**放入数据的时候，没有被取走，会造成阻塞。

修改非缓冲**channel** 为缓冲**channel**就行，否则会造成资源耗尽。

## 所有任务完成

之前都是用`sync.waitGroup`实现，这次利用csp机制实现：

```go
func runTask(id int) string {
	time.Sleep(10 * time.Millisecond)
	return fmt.Sprintf("the result is from %d", id)
}

func AllResponse() string {
	numOfRunner := 10
	ch := make(chan string) // 非缓冲channel
	for i := 0; i < numOfRunner; i++ {
		go func(i int) {
			ret := runTask(i)
			ch <- ret
		}(i)
	}

	finalRet := ""
	for i := 0; i < numOfRunner; i++ {
		finalRet += <-ch + "\n"
	}

	return finalRet
}

func TestFirstResponse(t *testing.T) {
	t.Log(AllResponse())
	/** 运行结果：
	=== RUN   TestFirstResponse
	    TestFirstResponse: all_done_test.go:33: the result is from 9
	        the result is from 0
	        the result is from 2
	        the result is from 7
	        the result is from 4
	        the result is from 6
	        the result is from 1
	        the result is from 5
	        the result is from 8
	        the result is from 3

	--- PASS: TestFirstResponse (0.01s)
	*/
}
```

## 对象池

使用 buffered channel 实现对象池

![](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9xaW5pdS5nYW9iaW56aGFuLmNvbS8yMDIwLzA1LzIzL2Y2ZGE0NTcyZTQxOWEucG5n?x-oss-process=image/format,png)



```go
type ReusableObj struct {
}

type ObjPool struct {
	bufChan chan *ReusableObj // 用于缓冲可重用对象
}

func NewObjPool(numOfObj int) *ObjPool {
	objPool := ObjPool{}
	objPool.bufChan = make(chan *ReusableObj, numOfObj)
	// 提前建立好连接
	for i := 0; i < numOfObj; i++ {
		objPool.bufChan <- &ReusableObj{}
	}
	return &objPool
}

// 获取连接
func (p *ObjPool) GetObj(timeout time.Duration) (*ReusableObj, error) {
	select {
	case ret := <-p.bufChan:
		return ret, nil
	case <-time.After(timeout): // 超时控制
		return nil, errors.New("time out")
	}
}

// 放入连接
func (p *ObjPool) ReleaseObj(obj *ReusableObj) error {
	select {
	case p.bufChan <- obj:
		return nil
	default:
		return errors.New("overflow")
	}
}

func TestObjPool(t *testing.T) {
	pool := NewObjPool(10) // 创建对象池

	for i := 0; i < 11; i++ {
		// 从对象池中获取
		if v, err := pool.GetObj(time.Second * 1); err != nil {
			t.Error(err)
		} else {
			fmt.Println(v)
			// 放入对象池
			if err := pool.ReleaseObj(v); err != nil {
				t.Error(err)
			}
		}
	}

	fmt.Println("Done")
}
```

## sync.Pool对象缓存

![](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9xaW5pdS5nYW9iaW56aGFuLmNvbS8yMDIwLzA1LzIzLzI3OGQ1YzgzOTg5YWEucG5n?x-oss-process=image/format,png)

**sync.Pool** 对象获取：

- 尝试从私有对象获取
- 私有对象不存在，尝试从当前 **Processor** 的共享池获取
- 如果当前 **Processor** 共享池也是空的，那么就尝试去其他 **Processor** 的共享池获取
-  如果所有⼦池都是空的，最后就⽤⽤户指定的 New 函数，产⽣⼀个新的对象返回




**sync.Pool** 对象放回：

- 如果私有对象不存在则保存为私有对象
- 如果私有对象存在，放⼊当前 **Processor** ⼦池的共享池中




**sync.Pool** 对象生命周期：

- **GC** 会清除 **sync.Pool** 缓存的对象
- 对象的缓存有效期为下⼀次 **GC** 之前



```go
func TestSyncPool(t *testing.T) {
	pool := &sync.Pool{
		New: func() interface{} {
			fmt.Println("Create a new object.")
			return 100
		},
	}

	v := pool.Get().(int) // 从池中获取并断言类型
	fmt.Println(v)        // 100

	pool.Put(3)
	v1, _ := pool.Get().(int)
	fmt.Println(v1) // 3

	//在放进去个 2
	pool.Put(2)
	//为了验证生命周期 这里GC一下
	runtime.GC()
	v3, _ := pool.Get().(int)
	fmt.Println(v3) // 100 而不是 2
	/** 运行结果：
	=== RUN   TestSyncPool
	Create a new object.
	100
	3
	Create a new object.
	100
	--- PASS: TestSyncPool (0.00s)
	*/
}
```

```go
func TestSyncPoolMultiGoroutine(t *testing.T) {
	pool := sync.Pool{
		New: func() interface{} {
			fmt.Println("Create a new object.")
			return 10
		},
	}

	pool.Put(100)
	pool.Put(100)
	pool.Put(100)

	var wg sync.WaitGroup

	for i := 0; i < 10; i++ {
		wg.Add(1)
		go func() {
			t.Log(pool.Get())
			wg.Done()
		}()
	}
	wg.Wait()
	/** 运行结果：
	=== RUN   TestSyncPoolMultiGoroutine
	    TestSyncPoolMultiGoroutine: sync_pool_test.go:59: 100
	Create a new object.
	    TestSyncPoolMultiGoroutine: sync_pool_test.go:59: 10
	Create a new object.
	    TestSyncPoolMultiGoroutine: sync_pool_test.go:59: 10
	    TestSyncPoolMultiGoroutine: sync_pool_test.go:59: 100
	Create a new object.
	Create a new object.
	    TestSyncPoolMultiGoroutine: sync_pool_test.go:59: 10
	Create a new object.
	Create a new object.
	Create a new object.
	    TestSyncPoolMultiGoroutine: sync_pool_test.go:59: 100
	    TestSyncPoolMultiGoroutine: sync_pool_test.go:59: 10
	    TestSyncPoolMultiGoroutine: sync_pool_test.go:59: 10
	    TestSyncPoolMultiGoroutine: sync_pool_test.go:59: 10
	    TestSyncPoolMultiGoroutine: sync_pool_test.go:59: 10
	--- PASS: TestSyncPoolMultiGoroutine (0.00s)
	*/
}
```



**sync.Pool** 总结：

- 适合于通过复用，降低复杂对象的创建和GC代价
- 协程安全，会有锁的开销
- 生命周期受GC影响，不适合于做连接池等，需自己管理生命周期的资源的池化