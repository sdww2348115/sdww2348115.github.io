---
layout: post
title:  "deadlock"
date:   2019-03-08 23:09:00 +0800
categories: jvm concurrent deadlock
permalink: /jvm/deadlock
---

## 概述
最近在实现一个分布式锁的过程中遇到了一些关于死锁的有趣问题，发现JVM中的死锁比我之前想的要复杂得多。这里简单做一点记录，希望大家以后不要踩到同样的坑。

## 背景知识

### 死锁的定义
>Deadlock: In concurrent computing, a deadlock is a state in which each member of a group is waiting for another member,  including itself, to take action, such as sending a message or more commonly releasing a lock.

个人翻译：在并发编程的环境中，多个成员之间相互等待，导致大家都陷入一种无法继续进行下去的状态。
如下图，线程X与Y均需要同时获取临界资源A,B才能继续往下执行。在程序运行的过程中，线程X获取到了临界资源A，线程Y获取到了临界资源B。此时，线程X需要临界资源B才能向下运行，线程Y需要临界资源A才能向下运行，整个系统处于一种无法继续运行的状态，这个状态就被称为死锁。

![deadlock](../resources/img/deadlock.png)

### 死锁的原因

通常是由于程序不够严谨，在重载（竞争严重）的环境中偶然出现。死锁生成的必然条件：

 - 互斥锁
 - 持有等待
 - 主动释放
 - 环形等待

关于以上条件的说明请参考各个文档，这里就不再细讲了。
 
### 死锁的处理方式

首先介绍两种教科书上的解法：

 - 程序获取锁的时候，必须按照一定顺序进行。以上图为例，无论是线程X还是线程Y，都必须按照同样的顺序（A->B 或者 B -> A）依次获取锁。这样在出现竞争的情况下，其他的线程必然阻塞在这条阻塞链的起始处，死锁也就不会存在了。（某位码农写出了A -> B -> ... -> A的代码除外)。在个人有限的人生的中，目前还没有发现采用这种方式的代码。因为在实际的使用过程中，不同锁控制的临界资源通常是不相关的；而且随着代码的开发，程序所使用锁的数量通常是不断变化的，把所有的锁进行一个编排这一点基本无法实现。
 - 银行家算法：在程序申请锁的同时