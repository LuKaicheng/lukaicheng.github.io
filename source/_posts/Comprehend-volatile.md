---
title: Java并发基础--理解volatile
date: 2017-07-23 23:52:56
tags:
- volatile
- JUC
---

## 引言

众所周知，Java语言是通过共享内存的方式，来实现多线程之间的通信行为。为了保证共享变量的值能在不同线程之间得到一致地更新，可以考虑使用`synchronized`来确保共享变量的可见性。除此之外，Java语言还提供了另一种机制：`volatile`。使用`volatile`同样也提供对共享变量内存可见性的保证，并且它的运行时开销相比`sychronized`会更少，这种行为是由Java内存模型所确保的。下面就以`volatile`为中心，梳理并总结一下对内存可见性、重排序、Java内存模型、happens-before规则、内存屏障等一系列概念的理解。

<!-- more  -->

## 缘由

首先来了解一下什么是内存可见性，以及为什么会发生这个问题。

### 内存可见性

可见性问题是造成很多Java多线程程序错误的根源，其主要是指在多线程应用中，不同线程对同一个共享变量分别进行读写时，如果没有采用正确的同步行为，那么可能会出现读线程无法适时的看到其他线程写入共享变量的值。

比如以下这个程序，一个线程重复的调用方法`one()`，另外一个线程重复调用方法`two()`：

```java
class Test {
    static int i = 0, j = 0;
    static void one() { i++; j++; }
    static void two() {
        System.out.println("i=" + i + " j=" + j);
    }
}
```

那么某些时候方法`two()`可能打印出来变量`j`的值大于另一个变量`i`的值。

而引发这个问题的原因归结起来主要是两个：**CPU高速缓存和重排序行为**

### CPU高速缓存

由于计算机的存储设备与处理器的运算速度有几个数量级的差距，所以现代计算机系统都不得不加入一层读写速度尽可能接近处理器运算速度的高速缓存来作为内存与处理器之间的缓冲：将运算需要使用到的数据复制到缓存中，让运算能快速进行，当运算结束后再从缓存同步回内存之中。因此在某些时刻，CPU上的缓存数据可能与内存中的数据是不一致的，所以极有可能出现某个线程对共享变量设置的新值还保存在高速缓存中且未被写回到内存，而这时候如果有另一个线程读取该共享变量，就会读取到内存中的旧值。

### 重排序行为

从编译源程序到运行时CPU执行指令，出于性能方面的考虑，这个过程中常常会对原始程序指令做出重排序行为，主要包括**编译器重排序**和**处理器重排序**：

- **编译器重排序**：编译器在编译阶段可以在不改变程序语义的情况下，对生成的目标代码进行重新排列。
- **处理器重排序**：现代处理器采用指令级并行技术将多条指令重叠执行，如果不存在数据依赖性，处理器可以改变语句对应机器指令的执行顺序。

正是由于存在这种情况，如果没有正确的同步机制保证，程序可能出现意想不到的行为。

## 方案

明确了可见性问题极其原因，那么是时候了解`volatile`背后的Java内存模型，以及其如何确保`volatile`的内存可见性。

### Java内存模型概述

由于Java倡导”一次编写，到处运行“的理念，为了屏蔽各种硬件和操作系统的内存访问差异，以实现让Java程序在各种平台下都能达到一致的内存访问效果，Java语言为程序员以及JVM实现者提供了一种抽象的内存模型规范，下面是引用自规范的原话：

> A *memory model* describes, given a program and an execution trace of that program, whether the execution trace is a legal execution of the program. The Java programming language memory model works by examining each read in an execution trace and checking that the write observed by that read is valid according to certain rules.
>
> The memory model describes possible behaviors of a program. An implementation is free to produce any code it likes, as long as all resulting executions of a program produce a result that can be predicted by the memory model.

简而言之，Java内存模型主要目标就是**确保程序执行可以产生可预期的行为**。

下图先给出其抽象模型：

![Java内存模型](https://raw.githubusercontent.com/LuKaicheng/lukaicheng.github.io/hexo/source/images/java_memory_model.png)

从抽象角度看，JMM定义了线程和主内存之间的抽象关系：线程之间的共享变量存储在主内存中，每个线程都有一个私有的本地内存，本地内存中存储该线程读写所需的共享变量的副本，线程之间的通信由JMM控制，其决定一个线程对共享变量的写入何时对另一个线程可见。

了解了Java内存模型的基本概念和抽象视图之后，接下来是时候了解**happens-before**概念了。

### Happens-Before

JSR-133使用happens-before的概念来指定两个操作之间的执行顺序，其中两个操作可以在一个线程内，也可以在不同线程之间。其主要定义如下：

1. 如果一个操作happens-before另一个操作，那么第一个操作的执行结果将对第二个操作**可见**，并且第一个操作的执行顺序排在第二个操作之前。(*对程序员的承诺*)
2. 两个操作之间存在happens-before关系，并不意味着Java平台的具体实现必须要按照happens-before关系指定的顺序来执行，如果重排序之后的执行结果与按照happens-before关系执行的结果一致，那么这种重排序并不非法。(*对JMM实现者的约束*)

java内存模型提供了一套happens-before规则来指导程序员如何定义程序的操作顺序以提供正确的内存可见性，其中就有一条关于`volatile`的规则：

**volatile变量规则 - 对一个volatile域的写，happens-before任意后续对这个volatile域的读。**

至此，对于java内存模型如何保证`volatile`变量的内存可见性，我们终于找到了依据。但是每个程序员内心都应该是充满好奇心，所以接下来，我们再稍微了解一下这条happens-before规则是如何保证`volatile`的语义。

### 重排序规则

前面我们已经知道，引发内存可见性的原因主要之一可能是重排序问题，因此为了实现`volatile`的内存语义，Java内存模型指定了`volatile`重排序规则表以禁止某些指令的重排序：

| **是否能重排序** | *第二个操作* | *第二个操作*   | *第二个操作*   |
| ---------- | ------- | --------- | --------- |
| *第一个操作*    | 普通读写    | volatile读 | volatile写 |
| 普通读写       |         |           | No        |
| volatile读  | No      | No        | No        |
| volatile写  |         | No        | No        |

这个规则表明确规定了针对`volatile`域的读写和普通读写之间的重排序行为是否被允许，根据这个规则表，编译器就可以在生成字节码时，通过往指令序列中插入特定的内存屏障来禁止关于`volatile`域的特定类型的重排序行为。

### 内存屏障

内存屏障的含义在维基百科里定义如下：

> **内存屏障**，也称**内存栅栏**，**内存栅障**，**屏障指令**等，是一类[同步屏障](https://zh.wikipedia.org/wiki/%E5%90%8C%E6%AD%A5%E5%B1%8F%E9%9A%9C)指令，是CPU或编译器在对内存随机访问的操作中的一个同步点，使得此点之前的所有读写操作都执行后才可以开始执行此点之后的操作。语义上，内存屏障之前的所有写操作都要写入内存；内存屏障之后的读操作都可以获得同步屏障之前的写操作的结果

Java内存模型把内存屏障分为4类：

- **LoadLoad**(*Load1;LoadLoad;Load2*)：确保Load1数据的装载先于Load2及后续装载指令的装载
- **StoreStore**(*Store1;StoreStore;Store2*)：确保Store1数据对其他处理器可见(刷新到内存)先于Store2及 所有后续存储指令的存储
- **LoadStore**(*Load1;LoadStore;Store2*)：确保Load1数据装载先于Store2及所有后续的存储指令刷新到内存
- **StoreLoad**(*Store1;StoreLoad;Load2*)：确保Store1数据对其他处理变得可见(刷新到内存)先于Load2及所有后续装载指令的装载。使该屏障之前的所有内存访问指令(存储和装载指令)完成之后，才执行该屏障之后的内存访问指令。

针对`volatile`域的读写会依据下表所示插入内存屏障：

| **需要插入的屏障** | *第二个操作*    | *第二个操作*     | *第二个操作*     | *第二个操作*      |
| ----------- | ---------- | ----------- | ----------- | ------------ |
| *第一个操作*     | 普通读        | 普通写         | volatile读   | volatile写    |
| 普通读         |            |             |             | `LoadStore`  |
| 普通写         |            |             |             | `StoreStore` |
| volatile读   | `LoadLoad` | `LoadStore` | `LoadLoad`  | `LoadStore`  |
| volatile写   |            |             | `StoreLoad` | `StoreStore` |

## 总结

至此，从`volatile`角度看去，整个因果线条已经相当明显。首先给出内存可见性的定义，进一步分析其可能引发的原因，从而转到了为解决内存可见性问题而提出的Java内存模型，结合`volatile`引出了该内存模型的核心概念：happens-before，随后分析组成`volatile`这条happens-before规则背后的重排序规则，最终以内存屏障概念收尾。本文并不强求面面俱到，而是以自己理解`volatile`的方式来讲诉其中的因果关系，个人感觉对于学习这一块的相关概念是一种比较好的思路，如果需要更深刻了解每一种概念，可以查看参考中的一些文档或者自行搜索。

## 参考

[volatile Fields](https://docs.oracle.com/javase/specs/jls/se8/html/jls-8.html#jls-8.3.1.4)

[Memory Model](https://docs.oracle.com/javase/specs/jls/se8/html/jls-17.html#jls-17.4)

[The JSR-133 Cookbook for Compiler Writers](http://gee.cs.oswego.edu/dl/jmm/cookbook.html)

[深入理解Java虚拟机](https://book.douban.com/subject/24722612/)

[Java并发编程的艺术](https://book.douban.com/subject/26591326/)

[内存屏障](https://zh.wikipedia.org/wiki/%E5%86%85%E5%AD%98%E5%B1%8F%E9%9A%9C)