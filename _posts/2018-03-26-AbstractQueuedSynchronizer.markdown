---
layout: post
title:  "AbstractQueuedSynchronizer剖析"
categories: java juc concurrent
permalink: /concurrent/AQS
description: AbstractQueuedSynchronizer源码剖析
---

## 背景
AbstractQueuedSynchronizer是由并发大师Doug Lea所开发的多线程同步工具框架。该数据结构主要用于解决同步环境下多线程的blocking等待调度机制，包括独占锁的等待唤醒、并发锁的等待唤醒、以及codition队列等。JUC中许多工具都是利用AQS实现：例如ReentrantLock、ReadWriteLock、CountDownLatch、Semaphore等。

## 核心原理

利用AbstractQueuedSynchronizer，线程获取临界资源失败并等待的原理如下：

* 使用原子操作尝试获取临界区访问权限，如果成功，则直接往后运行
* 如果失败，则使用Node将当前线程、需要获取的临界资源以及等待状态以链表Node形式进行封装，并放入当前AQS的队列末尾
* 再次check是否能够获取临界资源（这里也可以理解为手动触发一次唤醒操作），目的是防止在构建Node并添加至队列的过程中丢失唤醒信号
* 检查当前线程是否应该进入block状态：1.有可能前面部分等待的Node已经处于cancel状态了，但仍然未被清理，这时当前线程同样应负责进行清理操作；2.当前线程之前的节点处于condition或者shared模式，这时需要告诉前面的线程后续需要一个唤醒信号
* 线程按照预设逻辑将自己挂起并等待唤醒

挂起线程被唤醒时有以下几种情况：

* 某线程释放了临界区资源，唤起当前线程尝试进行抢夺
* 当前线程收到Interrupted信号

所以当线程被唤醒后，会首先检查自己的状态，如果为interrupted，则将代表自己的节点从等待链表中取下来，再将自己的状态重置为interruptted。
