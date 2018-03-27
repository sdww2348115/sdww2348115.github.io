---
layout: post
title:  "AbstractQueuedSynchronizer"
categories: source
permalink: /source/AQS
---

# AbstractQueuedSynchronizer是什么
AbstractQueuedSynchronizer(下文简称AQS)是Doug Lea大师所写的并发工具类之一，它通过queue的方式提供了一整套线程资源竞争/同步的基本模式，是各种Lock接口实现类/并发工具类的基石。

# AQS的诞生背景及目的

在ReentrantLock等各种并发工具类被开发出来之前，JAVA的多线程同步方式为：synchronized关键字，Object.wait()方法，Object.notify()方法以及Object.notifyAll()方法。其中synchronized关键字解决的是多线程环境下资源的竞争问题（Lock语义），wait/notify系方法解决的是线程之间同步的问题。(这里不讨论volatile关键字以及CAS方法，因为volatile关键字在JDK1.5之前其内存语义并不能保证happens-before语义，仅提供了内存的可见性保证；而CAS关键字也无法考证其被正式大规模应用的版本)

针对最基本的synchronized关键字而言，它能够解决大多数情况下的临界区的问题，但是功能相对孱弱：
1. 一旦进入synchronized关键字逻辑，线程将进入blocking等待的过程，无法响应中断。在某些条件下，在长时间获取不到锁的情况下我们可能需要手动中断线程尝试获取临界区资源的行为。
2. 无法实现异步获取锁的行为，即tryLock然后立即返回true \|\| false；也无法实现获取锁时超时中断的功能。（此条可认为是1的补充）
3. synchronized关键字在多线程竞争临界区资源时不可控。通俗来说，synchronized关键字无法提供AQS的fair语义。

解决了以上诸多问题的"加强版"资源竞争工具就是Lock接口类的各种实现，它们所共有的实现思路是：将JVM中Object Monitor相关的逻辑用JAVA代码实现，在这基础上为源代码增添了可以中断，异步获取，超时获取等功能。其中最典型的即为ReentrantLock类，它在JAVA语言的层面上完整实现了JVM层次的synchronized语义。再将所有Lock所共有的行为模式抽象出来，就是本文的中心：AbstractQueuedSynchronizer类。
    
将运行着的JAVA程序分为两层：底层为JVM虚拟机，上层为可控JAVA代码的话，可以认为AQS就是JAVA代码层的synchronized关键字。JVM所提供的wait/notify等方法与synchronized关键字息息相关：1.wait/notify方法必须被包含在synchronized语法块中；2.wait方法涉及到线程释放对应的锁资源。所以，要让AQS能够替代synchronize，必须实现线程之间的同步解决方案，即AQS的Condition模块。
    
AQS的功能大致可以分为这么两块：1.多线程竞争临界区资源；2.多线程同步方案。本文将分别从这两个方面来介绍AQS。

# 多线程竞争临界区资源
AQS对于多线程竞争临界区资源的实现主要依靠队列``sync queue``实现。它是一种CLH队列的变种，只有队列头节点的线程能够进入临界区，CLH队列通过自旋的方式等待前一个节点释放资源，而sync queue则是通过blocking的方式等待。当线程退出临界区的时候会唤醒后一个节点，让后续的线程依次进入临界区执行。sync queue中Node的数据结构如下（包括Condition部分代码）
```java
static final class Node {

		/*********** waitStatus 相关 *************/
        //waitStatus 状态，代表该Node被取消
        static final int CANCELLED =  1;
        //waitStatus 状态，代表该节点的后续节点等待该节点完成
        static final int SIGNAL    = -1;
        //waitStatus 状态，代表该节点为Condition节点，应处于Condition Queue中
        static final int CONDITION = -2;
        //waitStatus 状态，代表当唤醒该节点时，还要唤醒后续节点
        static final int PROPAGATE = -3;
		//标识当前Node所处状态
        volatile int waitStatus;

		/*********** prev/next 相关 *************/
        //指向queue中前一个节点
        volatile Node prev;
		//指向queue中后一个节点
        volatile Node next;
		
		//返回该节点的前驱节点
		final Node predecessor() throws NullPointerException {
            Node p = prev;
            if (p == null)
                throw new NullPointerException();
            else
                return p;
        }

		/*********** 该Node所包装的thread *************/
        volatile Thread thread;

		/*********** nextWaiter 相关 *************/
		//仅用于sync queue，代表Node为共享式
        static final Node SHARED = new Node();
        //仅用于sync queue，代表Node为独占式
        static final Node EXCLUSIVE = null;
		//nextWaiter在不同的queue中有不同的含义
		// 1.在sync queue中，它保存的是Node的状态(独占/共享)
		// 2.在condition queue中，它保存的是下一个Node的引用
        Node nextWaiter;

        //返回sync queue中该Node是否为共享模式
        final boolean isShared() {
            return nextWaiter == SHARED;
        }

    }
```
