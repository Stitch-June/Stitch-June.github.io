---
title: Go常用集合
tags: []
id: '90'
categories:
  - - uncategorized
date: 2020-09-27 12:28:48
---

\[Toc\]

## 数组和切片

### 数组

声明：

```
    var a [3]int // 声明并初始化为默认零值
    a[0] = 1
    b := [3]int{1, 2, 3}           // 声明同时初始化
    c := [2][2]int{{1, 2}, {3, 4}} // 多维数组初始化
    t.Log(a[0], a[2])
    t.Log(b[2])
    t.Log(c[1][1])
```

遍历：

```
    arr := [...]int{1, 2, 3, 4, 5, 6} // 自动判断长度

    for i := 0; i < len(arr); i++ { // 典型写法遍历数组
        t.Log(arr[i])
    }

    for idx, e := range arr { // 相当于其它语言的foreach
        t.Log(idx, e)
    }

    for _, e := range arr { // 我们可能用不到 idx 但go语言定义一个值不去使用编译会不通过 使用_代表不关心这个结果，来占位
        t.Log(e)
    }
```

截取：

a\[开始索引(包含):结束索引(不包含)\]

```
    arr := [...]int{1, 2, 3, 4, 5, 6}
    // a[开始索引(包含):结束索引(不包含)]
    t.Log(arr[0:1]) // 1
    t.Log(arr[2:]) // 3 4 5 6
    t.Log(arr[1:len(arr)]) // 2 3 4 5 6
    t.Log(arr[1:3]) // 2 3
```

### 切片

内部结构：

![](http://qiniu.gaobinzhan.com/2020/05/13/7106b2933bdf9.jpeg)

声明：

```
    var s0 []int            // 定义看起来特别像数组，但没有指定长度
    t.Log(len(s0), cap(s0)) // 0 0
    s0 = append(s0, 1)
    t.Log(len(s0), cap(s0)) // 1 1

    s1 := []int{1, 2, 3, 4} // 初始化一个切片
    t.Log(len(s1), cap(s1)) // 4 4

    // []type,len,cap 其中len个元素会被初始化为默认零值，未初始化元素不可以访问
    s2 := make([]int, 3, 5)    // len为3 cap为5
    t.Log(len(s2), cap(s2))    // 3 5
    t.Log(s2[0], s2[1], s2[2]) // 成功被初始化 结果：0 0 0
    //t.Log(s2[0], s2[1], s2[2], s2[3]) // 出现了一个错误 index out of range [3]
    s2 = append(s2, 1)
    t.Log(s2[0], s2[1], s2[2], s2[3]) // 0 0 0 1
    t.Log(len(s2), cap(s2))           // 4 5 len变成了4
```

共享存储结构：

![](http://qiniu.gaobinzhan.com/2020/05/13/775b24bebfd13.jpeg)

```
    s := []int{}
    for i := 0; i < 10; i++ {
        s = append(s, i) // 为什么重新赋值给s,是因为结构体指向的连续存储空间进行了变化,并把原有的连续存储空间拷贝到新的连续存储空间
        t.Log(len(s), cap(s))
    }
    /** 运行结果：当len不够用的时候会增长，cap不够用的时候增长为前一个cap的2倍
        TestSliceGrowing: slice_test.go:28: 1 1
        TestSliceGrowing: slice_test.go:28: 2 2
        TestSliceGrowing: slice_test.go:28: 3 4
        TestSliceGrowing: slice_test.go:28: 4 4
        TestSliceGrowing: slice_test.go:28: 5 8
        TestSliceGrowing: slice_test.go:28: 6 8
        TestSliceGrowing: slice_test.go:28: 7 8
        TestSliceGrowing: slice_test.go:28: 8 8
        TestSliceGrowing: slice_test.go:28: 9 16
        TestSliceGrowing: slice_test.go:28: 10 16
    */
```

## Map声明、元素访问及遍历

声明：

```
    m1 := map[int]int{1: 1, 2: 4, 3: 9} // 初始化一个map
    t.Log(m1[2])                        // 4
    t.Logf("len m1 = %d", len(m1))      // len m1 = 3
    m2 := map[int]int{}                 // 初始化一个空map
    m2[4] = 16                          // 赋值
    t.Logf("len m2 = %d", len(m2))      // len m2 = 1
    m3 := make(map[int]int, 10)         // 使用make初始化map
    t.Logf("len m3 = %d", len(m3))      // len m3 = 0 填的len为10,但打印出来为0
```

元素访问：

```
    // key在map中存在吗？
    // key存在但是对应的值是空值
    m1 := map[int]int{}
    t.Log(m1[1]) // 0 不存在输出0
    m1[2] = 0    // 设置value正好为0
    t.Log(m1[2]) // 0 也是0 该如何判断key是否存在呢
    // 需要主动去判断
    if v, ok := m1[3]; ok {
        t.Logf("key 3's value is %d", v)
    } else {
        t.Log("key 3 is not existing.") // 将输出这句话
    }

    m1[3] = 9
    if v, ok := m1[3]; ok {
        t.Logf("key 3's value is %d", v) // 将输出这句话
    } else {
        t.Log("key 3 is not existing.")
    }
```

遍历：

```
    m1 := map[int]int{1: 1, 2: 4, 3: 9}
    for k, v := range m1 {
        t.Log(k, v)
    }
    /** 运行结果
        TestTravelMap: map_test.go:41: 1 1
        TestTravelMap: map_test.go:41: 2 4
        TestTravelMap: map_test.go:41: 3 9
     */
```

## Map与工厂模式，实现Set

*   Map的value可以是一个方法
*   与Go的Dock type接口方式一起，可以方便的实现单一方法对象的工厂模式

```
    m := map[int]func(op int) int{}
    m[1] = func(op int) int {
        return op
    }
    m[2] = func(op int) int {
        return op * op
    }
    m[3] = func(op int) int {
        return op * op * op
    }
    t.Log(m[1](2), m[2](2), m[3](2)) // 运行结果 2 4 8
```

实现Set：

Go内置集合中没有Set实现，可以`map[type]bool`

*   元素的唯一性
*   基本操作
    

```
    mySet := map[int]bool{}
    mySet[1] = true // 添加元素 value置为true
    n := 1
    if mySet[n] {
        t.Logf("%d is existing.", n) // 输出这句话
    } else {
        t.Logf("%d is not existing.", n)
    }

    delete(mySet, 1) // 把1这个key从map中删除
    
    if mySet[n] {
        t.Logf("%d is existing.", n)
    } else {
        t.Logf("%d is not existing.", n) // 输出这句话
    }
```