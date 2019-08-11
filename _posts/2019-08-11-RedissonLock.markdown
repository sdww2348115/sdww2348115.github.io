---
layout: post
title:  "RedissonLock剖析"
date:   2019-08-11 23:44:00 +0800
categories: java redis distributed
permalink: /datastructure/RedissonLock
description: RedissonLock剖析
---

## 背景
RedissonLock全限名为`org.redisson.RedissonLock`，属于Redisson包中一个重要的工具类，作用是利用Redis提供一整套分布式锁解决方案。

## 继承关系与接口分析
Redisson继承关系如下图所示：

![redisson](../resources/img/RedissonLock.png)

从上图可以看出，Reddison主要继承了两个类/接口：

* RLock为锁相关的接口，这一分支主要约定了Redison具体实现锁的语义
* RedissonExpirable代表该实例属于一种会expire的redis对象，即redisson锁是带有expire属性的。

### RLock
Rlock继承了`java.util.Lock`接口以及`RLockAsync`接口，其中RLockAsync接口中所规定的方法都是RLock接口方法的异步版本，这里就不再赘述与分析了。本节将主要分析RLock对于Lock语义的补充。

```java
/*
属性相关接口
*/

//用于返回标记redissonlock的名字
String getName();

/*
方法增强：增加过期时间属性
*/

//提供了一种带有过期时间的lockInterruptibly方法
void lockInterruptibly(long leaseTime, TimeUnit unit) throws InterruptedException;

//提供一种带有过期时间属性的tryLock方法
boolean tryLock(long waitTime, long leaseTime, TimeUnit unit) throws InterruptedException;

//提供带有过期时间属性的lock方法
void lock(long leaseTime, TimeUnit unit);

/*
分布式锁额外添加方法
*/

//强制解锁
boolean forceUnlock();

//用于检查这个锁是否被占有
boolean isLocked();

//检查该锁是否被指定线程占有
boolean isHeldByThread(long threadId);

//检查该锁是否被当前线程占有
boolean isHeldByCurrentThread();

//获取该lock被当前线程所重入的次数
int getHoldCount();

//获取当前锁的剩余过期时间
long remainTimeToLive();
```

另：需要注意的是RedissonLock并未实现Lock中的newCondition()方法，该方法被调用后将会直接抛出UnsupportedOperationException.

