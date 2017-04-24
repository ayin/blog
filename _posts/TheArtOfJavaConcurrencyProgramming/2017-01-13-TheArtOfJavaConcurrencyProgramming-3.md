---
title: Java并发编程的艺术-第3章 Java内存模型
categories:
  - Technology
tags:
  - Java
  - 并发
  - 内存模型
excerpt: 读《Java并发编程的艺术》第3章 Java内存模型  笔记记录
---
{% include toc title="目录" icon="file-text" %}

# Java内存模型的基础
## 并发编程模型的两个关键问题
* 线程之间如何通信
* 线程之间如何同步

在命令式编程中，线程之间的通信机制有两种：

* 共享内存：隐式通信，显示同步
* 消息传递：显示通信，隐式同步

Java的并发主要采用的是共享内存的模型

## Java内存模型的抽象结构

![Java内存模型的抽象结构示意图]({{ site.baseurl }}/assets/image/TheArtOfJavaConcurrencyProgramming/JMM.png)

## 从源代码到指令序列的重排序
* 编译器重排序：编译器优化重排序
* 处理器重排序：指令系统重排序
* 处理器重排序：内存系统的重排序

![指令重排]({{ site.baseurl }}/assets/image/TheArtOfJavaConcurrencyProgramming/instructions-reorder.png)

## 并发编程模型的分类

为了优化下，现代处理器都提供有写缓存区，通过批量写提高性能，因此现代处理器都允许对”写读”操作进行重排序

![常见处理器的重排序规则]({{ site.baseurl }}/assets/image/TheArtOfJavaConcurrencyProgramming/CPU-reorder.png)

为了保证内存可见性，Java编译器在生成指令序列时会在合适位置插入内存屏障指令来禁止特定类型的处理器重排序，JMM把内存指令分为4类：

![内存屏障指令]({{ site.baseurl }}/assets/image/TheArtOfJavaConcurrencyProgramming/memory-barrier-instructions.png)

StoreLoad Barriers是一个“全能型”的屏障，它同时具有其他三个屏障的效果。现代的多处理器大都支持该屏障（其他类型的屏障不一定被所有处理器支持）。执行该屏障开销会很昂贵，因为当前处理器通常要把写缓冲区中的数据全部刷新到内存中（buffer fully flush）

## Happens-Before
* 程序顺序规则：同一个线程中前一个操作happens-before任意后续操作
* 监视器锁规则：对一个锁的解锁happens-before于随后对这个锁的加锁
* volatile变量规则：对一个volatile变量的写happens-before于任意后续对这个volatile变量的读
* 传递性： A happens-before B, B happens-before C，则有 A happens-before C

> happens-before并不意味着执行时间上的约束，而是对执行结果的可见性的约束

# 重排序
## 数据依赖性
* 写后写
* 写后读
* 读后写

对于单个处理器中执行的指令序列和单个线程中执行的操作，编译器和处理器不应该改变存在数据依赖关系的两个操作之间的执行顺序

## as-if-serial语义
as-if-serial语义的意思是：不管怎么重排序，(单线程)程序的执行结果不能被改变。编译器、runtime和处理器都必须遵守as-if-serial语义

## 程序顺序规则

happens-before与程序实际执行时间直接的关系

## 重排序对多线程的影响

控制依赖性

重排序缓冲（Reorder Buffer, ROB）

# 顺序一致性

顺序一致性内存模型是一个理论参考模型

## 数据竞争与顺序一致性
数据竞争的含义：一个线程写一个变量，另一个线程读同一个变量，而且写和读之间没有通过同步来排序

## 顺序一致性内存模型

* 一个线程中的所有操作必须按照程序的顺序执行
* 所有线程只能看到单一的执行顺序，顺序一致性内存模型中的每个操作必须立即对任意线程可见

JMM中没有保证，因为一个线程的写操作放在本地内存中而没有刷新到主内存的时候，这个操作只对当前线程可见，从其他线程的角度来看这个操作根本没有执行。

## 同步程序的顺序一致性效果
## 未同步程序的执行特性
对于未同步或者未正确同步的多线程程序，JMM只提供最小安全性

# volatile的内存语义


# 锁的内存语义
## 锁的释放和获取建立的happens-before关系
锁除了让临界区互斥执行外，还可以让释放锁的线程向获取同一个锁的线程发送消息
## 锁的释放和获取的内存语义
* 线程释放锁时，JMM会把线程对应的本地内存中的共享变量刷新到主内存中
* 线程获取锁时，JMM会把该线程对应的本地内存置为无效

> 锁释放与volatile写有相同的内存语义，锁获取与volatile读有相同的内存语义

* 线程A释放一个锁，实质上是线程A向接下来将要获取这个锁的某个线程发出了(线程A对共享变量所作的修改的)消息
* 线程B获取一个锁，实质上是线程B接收了之前某个线程发出的(在释放这个锁之前对共享变量所作的修改的)消息
* 线程A释放锁，随后线程B获取这个锁，这个过程本质上就是线程A通过主内存向线程B发送消息

## 锁内存语义的实现
