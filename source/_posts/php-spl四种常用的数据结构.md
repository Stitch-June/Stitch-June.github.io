---
title: php SPL四种常用的数据结构
tags: []
id: '49'
categories:
  - - uncategorized
date: 2020-09-20 19:10:24
---

```
$stack = new SplStack();
$stack->push('data1');
$stack->push('data2');
$stack->push('data3');
echo $stack->pop();
//输出结果为
//data3


```

```
$queue = new SplQueue();
$queue->enqueue("data1");
$queue->enqueue("data2");
$queue->enqueue("data3");
echo $queue->dequeue();
//输出结果为
//data1

```

```
$heap = new SplMinHeap();
$heap->insert("data1");
$heap->insert("data2");
echo $heap->extract();
//输出结果为
//data1


```

```
$array = new SplFixedArray(5);
$array[0]=1;
$array[3]=3;
$array[2]=2;
var_dump($array);
//输出结果为
// object(SplFixedArray)[1]
// public 0 => int 1
// public 1 => null
// public 2 => int 2
// public 3 => int 3
// public 4 => null
```

原文：https://blog.csdn.net/zhengwish/article/details/51742264