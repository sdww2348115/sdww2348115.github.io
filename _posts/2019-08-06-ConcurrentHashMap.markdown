---
layout: post
title:  "ConcurrentHashMap剖析"
date:   2019-08-06 23:36:00 +0800
categories: java jdk
permalink: /jdk/ConcurrentHashMap
description: ConcurrentHashMap剖析
---

## ConcurrentHashMap
ConcurrentHashMap是Jdk1.7新引入的一种Map类型数据结构，主要目的是实现一种适合在高并发场景下使用的HashMap。为实现该目的，其主要采用了`分段锁`技术,并发resize(),LongAddr计数等方式来提高整个数据结构的吞吐能力。
对该数据结构的学习主要有以下几个目的：

1. 接触并了解分段锁技术
2. 从并发的resie()中学习并发编程的特点与注意事项
3. 分析LongAddr数据结构的特点，并分析其原理

## 分段锁
分段锁中的分而治之思想是当今几乎所有分布式系统的基础，其核心原理为：将一个系统划分为相对独立的数个子模块，从而成倍提高整个系统的吞吐能力。如今的计算机工程中处处可见其思想：Kafka中的partition、redis cluster、solr cluster、数据库分库分表中的横向切分等都是使用的该技术。

HashMap在高并发条件下效率不高的核心原因是Collections.sychronizedMap()方法会将map中的每一个方法都加上synchronized关键字，任何方法都需要经过锁竞争才能得到运行。通过分而治之的思想，将整个HashMap划分为相对独立的多个粒度更小的数据结构，将互斥锁放在粒度更小的数据结构上，就可以成倍提高HashMap在高并发环境中的吞吐速率。

除分段锁技术外，ConcurrentHashMap还大量利用了CAS操作来替代锁，同样节约了许多线程竞争锁资源所带来的开销。

以putVal()方法为例，把一个Entry放入到ConcurrentHashMap中最有可能遇到的有以下两种情况：

1. 该Entry所对应的Slot为null，ConcurrentHashMap将直接通过CAS将Entry置换进入指定位置。
2. 该Entry所对应的Slot不为null, ConcurrentHashMap将锁定Slot位置处的节点，并进行对应的put操作。
