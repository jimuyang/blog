---
title: 几种IO
date: 2019-11-10 17:36:47
tags:
---
# 学习Linux环境下的几种IO模型和一些概念

## 先列学习和引用资料

* Richard Stevens的《UNIX® Network Programming Volume 1, Third Edition: The Sockets Networking》6.2 IO Models
* 博客：IO - 同步，异步，阻塞，非阻塞 （亡羊补牢篇） https://blog.csdn.net/historyasamirror/article/details/5778378
* Java NIO Tutorial: http://tutorials.jenkov.com/java-nio/index.html

## Linux IO Model （注意和Java中的概念区分
> 引用自 Richard Stevens "UNIX® Network Programming Volume 1, Third Edition: The Sockets Networking" 
* Blocking IO       阻塞IO
* NoneBlocking IO   非阻塞IO
* IO Multiplexing   IO多路复用
* Signal Driven IO  信号驱动IO（少见 不做了解
* Asynchronous IO   异步IO

### 做些说明：
1. 缓存IO/直接IO
这是一种对IO的分类 缓存IO又称标准IO，大多数文件系统的默认I/O操作都是缓存I/O。在Linux的缓存I/O机制中，数据先从磁盘复制到内核空间的缓冲区，然后从内核空间缓冲区复制到应用程序的地址空间。 

2. IO相关对象和阶段
对于一个network IO而言，一般会涉及到2个系统对象，一个是使用这个IO的进程/线程，一个是系统内核（kernel） 
当一个read操作发生时，会经历2个阶段：
* 等待数据准备好 wait for the data to be ready
* 将数据从kernel复制到用户进程中 copy data from kernel to process
不同的IO模型的区分正是在这2个阶段的行为不同  

### Blocking IO 阻塞IO
![](/images/BlockingIOModel.gif)
最大的特点 用户进程从read调用开始阻塞，直到**数据准备好**并**复制到用户进程中**（即read调用返回）之后，阻塞才会结束。

### NoneBlocking IO 非阻塞IO
linux下，可以通过设置socket使其变为non-blocking。当对一个non-blocking socket执行读操作时，流程是这个样子：
![](/images/NonblockingIOModel.gif)
最大的特点 在**数据准备好**之前的read调用会立即返回一个error（类似数据还没有准备好），直到**数据准备好**之后的read调用会阻塞并将数据**复制到用户进程中**，read结束

### IO Multiplexing IO多路复用 
使用select/poll/epoll实现 本质上和阻塞IO一样，但是通过维护一个fd集合来一次监控多个socket，从而大大提高系统并发数，注意到获取到就绪的fd之后还是需要阻塞的将数据**复制到用户进程中**
* select: fd数组 轮询 O(n) 限连接数
* poll: fd链表 轮询 O(n) 无连接数限制 
* epoll: event poll 事件驱动 O(1) 只关心活跃的fd
![](/images/IOMultiplexing.gif)

### Asynchronous IO 异步IO
真正的异步IO是没有环节阻塞的，直到**数据准备好**并**复制到用户进程中**之后才会通知用户进程直接使用
![](/images/AsyncIO.gif)