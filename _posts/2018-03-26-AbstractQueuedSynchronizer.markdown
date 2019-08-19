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

当执行取消某个线程等待方法cancelAcquire时。

1. 将Node.thread置为null， ws置为CANCELLED，防止线程被唤醒
2. 从Node开始，向前找到第一个未被取消的节点Node。这里需要分情况讨论：a.如果predNode是head节点或者处于需要传播的模式，就需要将node线程从Block状态唤醒，方便信号在链路上的传播，否则可能导致信号在传播的过程中丢失；b.如果后续节点处于未取消状态，则将后续节点挂到pred节点上，并把pred置为signal。（如果后续节点处于取消状态，则不用作任何事，因为后续节点的cancell会自动将对应的Node挂到正确的位置上去）

## 部分实现细节

* 所有节点所保存的ws是后续节点的等待状态，只有signle表示需要唤醒。
* 如果为共享模式，当线程获取临界资源成功后，会根据自身状态以及后续节点状态进行判断，并在适当时候唤醒后续节点。
* aquire与aquireInterruptibly的区别在于：当线程检查到自己获取到intterrupted信号后，Interruptibly相关接口会抛出线程中断异常。
* 在ReentrantLock中，如果设置锁为fair，则在tryAquire()使用CAS标记占用临界区资源前会首先检查队列中是否存在比自己等待更久的节点。


## Condition相关

Condition相关语意是对Object.wait()与Object.notify()的补全。Condition仅支持排他锁，共享锁模式下不支持。作为一个内部类，ConditionObject实现了condition的所有方法，在其内部同样使用了head与tail两个指针实现了一个condition链表，保存有所有调用condition.wait()方法的线程以及节点。

与Object.wait()类似，condition.wait()方法被调用后主要执行以下几个操作：

1. 检查线程是否处于Intterrupted状态
2. 释放lock相关资源
3. 在Condition队列上创建Node节点
4. 再次检查线程状态并进入Blocking状态

如果线程被唤醒：

1. 检查自己被唤醒的原因，是否需要抛出Intterrupted异常
2. 将condition队列上的节点取下来，并将节点注册至临界区等待线程处
3. 后续与普通线程等待差不多，获取到临界区资源后即可继续执行后续逻辑

主要是通过多次对线程状态的检查实现了对Intterruptd的支持

* Tips：Condition中的signal()仅会唤醒处于第一个位置处的节点，不会像Object.notify()随机唤醒节点。

## Read/Write 额外处理

1. ReadWriteLock中采用一个int记录等待readlock/writelock的线程数量：其中高16位为shared线程数，获取方法为 c >>> SHARED_SHIFT; 低16位为等待独占锁的线程数，获取方式为 c & ((1 << SHARED_SHIFT) - 1)

独占锁tryAquire的逻辑如下：
``` java
protected final boolean tryAcquire(int acquires) {
            /*
             * Walkthrough:
             * 1. If read count nonzero or write count nonzero
             *    and owner is a different thread, fail.
             * 2. If count would saturate, fail. (This can only
             *    happen if count is already nonzero.)
             * 3. Otherwise, this thread is eligible for lock if
             *    it is either a reentrant acquire or
             *    queue policy allows it. If so, update state
             *    and set owner.
             */
            Thread current = Thread.currentThread();
            //获取ReadWriteLock的整体status C
            int c = getState();
            //获取ReadWriteLock的独占锁重入次数 w
            int w = exclusiveCount(c);
            //当ReadWriteLock被独占/共享锁占有时，只有锁被当前线程持有且小于重入次数，此时直接记录重入次数
            if (c != 0) {
                // (Note: if c != 0 and w == 0 then shared count != 0)
                // 此时ReadWriteLock被读锁占有，获取失败
                if (w == 0 || current != getExclusiveOwnerThread())
                    return false;
                if (w + exclusiveCount(acquires) > MAX_COUNT)
                    throw new Error("Maximum lock count exceeded");
                // Reentrant acquire
                setState(c + acquires);
                return true;
            }
            //非公平情况下直接尝试修改status，尝试获取临界资源，随后修改ownerThread为当前线程，返回true
            if (writerShouldBlock() ||
                !compareAndSetState(c, c + acquires))
                return false;
            setExclusiveOwnerThread(current);
            return true;
        }
```

与独占式锁类似，共享锁的获取方式基本一样，只是多了额外两个步骤：

1. 共享锁在获取的时候需要check链表中第一个Node是否为独占锁等待，如果是，则tryAquire返回-1，在独占锁的后续进行排队。
2. 由于持有锁资源的线程可能不止一个，因此不同的线程在处理锁重入次数时有所不同：1.第一个获取读锁的线程将自己的重入次数放入firstReaderHoldCount中，并将firstReader设为当前线程；2.其他线程将重入次数放入ThreadLocal数据结构中。
3. 共享锁在释放时的处理方式同样分为两种：1. 如果当前线程为firstReader，如果重入次数完全释放，直接将firstReader置为null，并把firstReaderHoldCount置为0；2.如果当前线程并非为firstReader，则会将自己热threadlocal变量从threadLocalMap中移除。
