---
layout: post
title:  "distributedlock"
date:   2019-03-10 23:09:00 +0800
categories: jvm concurrent deadlock
permalink: /design/distributedlock
---

## 概述
本文主要内容为分析锁的逻辑结构、JVM中锁的实现，以及自己所设计的一种基于redis的分布式锁。

## 锁的简介

### 锁的定义
> In computer science, a lock or mutex (from mutual exclusion) is a synchronization mechanism for enforcing limits on access to a resource in an environment where there are many threads of execution. A lock is designed to enforce a mutual exclusion concurrency control policy。
---wiki

渣翻：在计算机科学中，锁（互斥量）是一种多线程环境中用于限制资源访问的同步机制，通常代表着并发中的一种互斥访问策略。

### 锁的种类
计算机中有许多种锁，最常见的的便是读写锁和可重入锁。

* 简单锁：资源只能被一个线程持有、加锁、与修改；其他线程对于该资源的加锁和操作必须等待持有锁的线程释放之后才能进行。
* 可重入锁：持有锁资源的线程可以重复的在资源上进行加锁操作，通常持有锁的线程将资源上所有的锁都释放后其他线程才能加锁并访问该资源。
* 读写锁：读写锁通常分为读锁和写锁，当资源被某线程以写锁模式占有时，其他线程均不能访问该资源；当资源被某线程以读锁模式占有时，其他线程可以对该资源加读锁进行读操作，但不能对该资源加写锁执行写操作。

## JVM中锁的实现
Java中常用的锁有两种：synchronized关键字与java.util.concurrent.ReentrantLock.它们都是可重入的排他锁，后者实现时参考了前者的设计。我们这里主要讨论前者，即JVM中自带的锁。
synchronized所实现的锁是一种独占的可重入锁，它还包含了许多优化所带来的新特性。

### 独占锁
锁资源在某一时刻内只能被某一个线程所占有，这样的结构被称为独占锁，独占锁是锁的最基本表现形式。
由于锁涉及到多个线程之间的同步，通常会借用用多个线程同时可见的一片内存空间来实现。这片内存空间通常有两种状态：
* free: 表明锁资源没有被任何线程所占有
* locked：锁资源已被某线程所占有
当一个线程运行到临界区时，它会首先去检查约定的内存空间状态，如果为free，那么就把这里置为locked，标明临界区已经被某线程所占有，其他线程看到locked标志后就不会再向下执行，达到排他运行的目的。

在实现锁的时候，请注意这里的两个要点：
1. 内存区域对所有线程可见。由于JVM内存模型的限制，这里的内存区域通常需要加上一点额外的处理才能保证其"可见"。关于JVM内存模型以及可见性的讨论，请查阅volatile关键字。
2. 查看并设置内存空间这一个过程必须是原子操作。设想这样一种场景：1.线程A查看共享内存空间，此时锁的状态为free；2.线程B查看共享内存空间，此时锁的状态也为free（线程A还没来得及上锁）；3.线程A将锁空间状态置为locked，执行临界区代码；4.线程B将锁空间状态置为locked，也执行临界区代码。此时线程A与线程B都在执行临界区的代码，锁的功能就失效了。因此，查看并修改这一个过程必须要保证原子性。具体实现请参考CAS（CompareAndSwap）