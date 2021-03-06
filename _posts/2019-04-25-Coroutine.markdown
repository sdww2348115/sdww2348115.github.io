---
layout: post
title:  "协程(coroutine)"
date:   2019-04-25 23:31:00 +0800
categories: concurrent thread task
permalink: /concurrent/coroutine
description: 从线程的角度描述了协程的概念;分析了协程的优点、缺点；以及协程的适用场景
---

## 协程是什么

与线程不同，协程是一种非抢占式的子任务调度系统，协程在运行时同样存在挂起与恢复，但是都在用户态进行，不存在内核态/用户态的切换，因此相比于线程调度开销要小得多。

下图为典型的多任务线程执行模型，任务被映射至线程上，不同的任务运行在不同的线程，程序通过线程调度任务。线程（任务）之间通过锁等并发工具保证执行的顺序。线程交替切换自身阻塞/运行状态，避免占用CPU资源又不做任何实质计算工作。

![thread-model](../resources/img/thread-model.png)

下图为协程模型下的多任务调度示意图，与多线程系统不同，多个任务将运行在同一个线程中，任务切换不涉及线程切换。当任务1执行到一定阶段需要任务2继续执行时，任务1会主动释放线程与CPU的控制权，让任务2在同一个线程执行。

![coroutine-model](../resources/img/coroutine-model.png)

综上，协程是一种编程语言级别的逻辑概念，描述的是多个任务在同一个线程内进行非抢占式调度的模型，与传统编程层面上的"进程/线程"调度模型有着本质的不同。协程含有以下几个典型特点：

1. 任务调度不涉及线程上下文切换，切换过程完全在线程内部（用户态）完成，由语言的执行器进行任务的上下文保存等切换工作。
2. 非抢占式调度，依赖于任务自身声明释放CPU控制权。

## 协程的优点与缺点

### 优点
协程最大的优点在于：任务调度所带来的开销远远小于线程(进程)模型。这里以两个任务交替运行的模型为例，分析不同调度模型所带来的时间开销。当线程A（正在执行的线程）发现应由线程B来执行任务时，在传统线程模型中的调度通常有以下几步：

1. 线程A执行系统调用，程序由用户态进入内核态
2. 线程A向操作系统发出唤醒线程B的信号量
3. 线程A退出内核态，进入用户态
4. 线程A执行系统调用，程序由用户态进入内核态
5. 操作系统在内核态下保存线程A当前执行的上下文（现场）
6. 操作系统将线程A挂起（阻塞）
....（为方便描述，这里假设线程A、B运行于单核操作系统中，且无其他线程干扰的理想环境中）
8. 操作系统唤醒线程B，Load 线程B的上下文，退出内核态并继续执行

其中一共涉及到4次内核/用户态的切换，还有一次线程上下文保存与一次线程上下文的载入。这些操作所消耗的时间非常多，在高并发的场景中频繁的线程切换将直接影响程序的吞吐量与运行效率。

协程模型在任务切换时值存在任务上下文的保存/载入工作，避免了多次内核/用户态的切换操作，其任务切换的效率比起线程模型要高很多。

### 缺点
个人认为，协程的缺点主要有两个：1. 协程任务的编写难度要高于线程任务；2. 协程模型下的任务只能并发，不能并行。

协程调度模型的实现通常由程序执行器实现（Windows操作系统层面支持协程），首先，作为一种非抢占式算法，程序员必须自己管理任务对于CPU资源的让出和获取动作；其次，由于调度模型的不同，线程同步工具将无法在协程中得到使用，用户必须自己完成任务之间的同步工作。接管任务调度，缺乏合适同步工具这些问题将导致程序员无论是工作量还是困难度都将远远大于传统的线程模型。除此之外，思想的转化同样是一个难题：由于一个方法中包含了多个入口和出口（每一个获取CPU资源处都是入口，yeild都是出口），造成了类似于"goto"语句的效果：程序的运行将不再是从上至下单方向的，也有可能从下跳到上方的某一处。想想当年计算机学界关于废除"goto"的讨论，大家不难理解这一点对于程序编码所带来的困扰有多大。

受协程模型的限制，相互协作的子任务只能在同一个线程中执行，因此协程天生就失去了并行的能力。多个任务之间并非是完全互斥的，通常也会有一些可以并行的部分，在系统负载较低的情况下，线程模型反而能够得到更好的运行效果。

![parallel-model](../resources/img/thread-parallel.png)

如上图所示，由于充分利用了多个CPU的并行计算能力，线程模型完成任务的速度反而比协程更快。

## 协程的适用场景

由于其模型的特殊性，协程的使用通常只适用于部分特殊的应用场景：

1. 状态机：在任一时刻，状态机只能处于其中一种状态中。当前持有CPU资源的任务代表着状态机本身的状态，协程中任务的切换与状态机状态的切换完全等价。（协程的使用方法与天国的goto果然扯不明白）
2. 生成器(迭代器属于生成器的特例)：程序运行状态为生成/消费交替进行，通过协程可以很方便的在这两个任务之间进行切换。所有对于数据结构的遍历操作都可用协程模型描述。
3. 通信信息流：事件在整个信息流中按照编排的次序流转，从一个handler到另一个handler。从另一个角度看来，每一个handler都可以看作是后续handler的生成器，因此所有的handler都可以采用协程的方式并发协作。（个人认为此项描述有些牵强，一个Engine+编排好的pipeline就能实现，不涉及任何线程切换，这个模型中没有必要引入协程强行扯上关系）

综上，个人认为协程的使用范围很窄，其最合适的模型为`状态机`和`生成器`。而这些模型都有一个显著的特点：程序的执行并不是单向执行，都隐藏着从下至上的反向跳转逻辑。因此，协程的核心竞争力便在于补充了当前计算机课科学编程模型的缺陷，使得程序可以多方面的流动，是对于goto语句的一个补充。