---
title: Redis-链表
tags: []
id: '133'
categories:
  - - Redis
date: 2021-04-01 00:48:13
---

链表提供了高效的节点重排能力，以及顺序性的节点访问方式，并且可以通过增删节点来灵活的调整链表的长度。

作为一种常用数据结构，链表内置在很多高级的编程语言里面，因为`Redis`使用的`c`语言并没有内置这种数据结构，所以`Redis`构建了自己的链表实现。

链表在`Redis`中的应用非常广泛，比如列表键的底层实现之一就是链表。当一个列表键包含了数量比较多的元素，又或者列表中包含的元素都是比较长的字符串的时，`Redis`就会使用链表作为列表键的实现。

举个例子，以下展示的`integers`列表键包含了`1-1024`共一千零二十四个整数：

```
redis> LLEN integers
(integer) 1024
redis> LRANGE integers 0 10
1) "1"
2) "2"
3) "3"
4) "4"
5) "5"
6) "6"
7) "7"
8) "8"
9) "9"
10) "10"
11) "11"
```

`integers`列表键的底层实现就是一个链表，链表中的每个节点都保存了一个整数值。
除了链表键之外，发布与订阅、慢查询、监视器等功能也用到了链表，`Redis`服务器本身还是要链表保存多个客户端信息的状态信息，以及使用链表来构建客户端输出缓冲区`（output buffer）`。

## 链表和链表节点的实现

每个链接节点使用一个`adlist.h/listNode`结构来表示：

```c
typedef struct listNode {
		// 前置节点
    struct listNode *prev;
    // 后置节点
    struct listNode *next;
    // 节点的值
    void *value;
} listNode;
```

多个`listNode`可以通过`prev`和`next`指针组成双端列表，如下图：

![](https://img-blog.csdnimg.cn/img_convert/a8921ac3ca23f31e04f086022ed3bfe3.png)

虽然仅仅使用多个`listNode`结构就可以组成链表，但使用`adlist.h/list`来持有链表的话，操作起来会更方便：

```c
typedef struct list {
		// 表头节点
    listNode *head;
    // 表尾节点
    listNode *tail;
    // 节点值复制函数
    void *(*dup)(void *ptr);
    // 节点值释放函数
    void (*free)(void *ptr);
    // 节点值对比函数
    int (*match)(void *ptr, void *key);
    // 链表所包含的节点数量
    unsigned long len;
} list;
```

`list`结构为链表提供了表头指针`head`、表尾指针`tail`，以及链表长度计数器`len`，而`dup`、`free`和`match`成员则是用于实现多态链表所需要的类型特定函数：

- `dup`函数用于复制链表节点所保存的值；
- `free`函数用于释放链表节点所保存的值；
- `match`函数则用于对比链表节点所保存的值和另一个输入值是否相等；

下图是由一个`list`结构和三个`listNode`结构组成的链表。

![](https://img-blog.csdnimg.cn/img_convert/761de7a0b30f3181f4619388fb836872.png)

## 总结

`Redis`的链表实现的特性可以总结如下：

- 双端：链表节点带有`prev`和`next`指针，获取某个节点的前置节点和后置节点的复杂度都是O(1)。
- 无环：表头节点的`prev`指针和表尾节点的`next`指针都指向`NULL`，对链表的访问以`NULL`为终点。
- 带表头指针和表尾指针：通过`list`结构的`head`指针和`tail`指针，程序获取链表的表头节点和表尾节点的复杂度为O(1)。
- 带链表长度计数器：程序使用`list`结构的`len`属性来对`list`持有的链表节点进行计数，程序获取链表中节点数量的的复杂度为O(1)。
- 多态：链表节点使用`void*`指针来保存节点值，并且可以通过`list`结构的`dup`，`free`，`match`三个属性为节点值设置类型特定函数，所以链表可以用于保存各种不同类型的值。