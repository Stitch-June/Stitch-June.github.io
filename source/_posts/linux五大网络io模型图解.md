---
title: Linux五大网络IO模型图解
tags: []
id: '50'
categories:
  - - uncategorized
date: 2020-09-20 19:19:26
---

\[TOC\]

> 对于一个应用程序即一个操作系统进程来说，它既有内核空间(与其他进程共享),也有用户空间(进程私有)，它们都是处于虚拟地址空间中。用户进程是无法访问内核空间的，它只能访问用户空间，通过用户空间去内核空间复制数据，然后进行处理。

#### 阻塞io(同步io)

发起请求就一直等待，直到数据返回。好比你去商场试衣间，里面有人，那你就一直在门外等着。(全程阻塞)

![](http://imgconvert.csdnimg.cn/aHR0cDovL3Fpbml1Lmdhb2JpbnpoYW4uY29tLzIwMTkvMTIvMjAvZjlmYWY2ZGY3YWQxZS5wbmc?x-oss-process=image/format,png)

#### 非阻塞io(同步io)

不管有没有数据都返回，没有就隔一段时间再来请求，如此循环。好比你要喝水，水还没烧开，你就隔段时间去看一下饮水机，直到水烧开为止。(复制数据时阻塞)

![](http://imgconvert.csdnimg.cn/aHR0cDovL3Fpbml1Lmdhb2JpbnpoYW4uY29tLzIwMTkvMTIvMjAvZGMzMWY5NjE5NWI5YS5wbmc?x-oss-process=image/format,png)

#### io复用(同步io)

I/O是指网络I/O,多路指多个TCP连接(即socket或者channel）,复用指复用一个或几个线程。意思说一个或一组线程处理多个连接。比如课堂上学生做完了作业就举手，老师就下去检查作业。(对一个IO端口，两次调用，两次返回，比阻塞IO并没有什么优越性；关键是能实现同时对多个IO端口进行监听，可以同时对多个读/写操作的IO函数进行轮询检测，直到有数据可读或可写时，才真正调用IO操作函数。)　  
![](http://imgconvert.csdnimg.cn/aHR0cDovL3Fpbml1Lmdhb2JpbnpoYW4uY29tLzIwMTkvMTIvMjAvZDk2YTU2YTJiMjRiYS5wbmc?x-oss-process=image/format,png)

#### 信号驱动io(同步io)

事先发出一个请求，当有数据后会返回一个标识回调，这时你可以去请求数据。好比银行排号，当叫到你的时候，你就可以去处理业务了(复制数据时阻塞)。  
![](http://imgconvert.csdnimg.cn/aHR0cDovL3Fpbml1Lmdhb2JpbnpoYW4uY29tLzIwMTkvMTIvMjAvMWE4NjM3MTg3MjcwMi5wbmc?x-oss-process=image/format,png)

#### 异步io

发出请求就返回，剩下的事情会异步自动完成，不需要做任何处理。好比有事秘书干，自己啥也不用管。　　  
![](http://imgconvert.csdnimg.cn/aHR0cDovL3Fpbml1Lmdhb2JpbnpoYW4uY29tLzIwMTkvMTIvMjAvNzk4MGI4MjlhM2YxMS5wbmc?x-oss-process=image/format,png)

#### 总结

五种IO的模型：阻塞IO、非阻塞IO、多路复用IO、信号驱动IO和异步IO；前四种都是同步IO，在内核数据copy到用户空间时都是阻塞的。

阻塞IO和非阻塞IO的区别在于第一步，发起IO请求是否会被阻塞，如果会那就是传统的阻塞IO，如果不会那就是非阻塞IO。

同步IO和异步IO的区别就在于第二个步骤是否阻塞，如果实际的IO读写阻塞请求进程，那么就是同步IO；如果不阻塞，而是操作系统帮你做完IO操作再将结果返回给你，那么就是异步IO