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

### RedissonExpirable

从名字即可看出该类代表的是可过期的redisson对象，该类主要通过redis的过期策略实现。

RedissonExpirable有两个重要参数:

* CommandAsyncExecutor：用于控制过期，执行对应命令的executor
* name:对象名称，实际上对应的是redis中的key值

## Redisson重点方法分析

作为一个典型排他锁的实现，Redisson中最重要的方法就是lock()与unlock().Redisson的构造方法只含有两个参数：CommandAsyncExecutor与name，其作用与含义与RedissonExpirable类似，这里就不再赘述了。

### lock

#### tryAquire语意实现
RedissonLock利用Redis中的Hash作为分布式锁的核心结构。该Hash结构主要属性如下：

* key：分布式锁名称，以String方式标记
* key-val pair1：key为持有锁的线程id；val为该线程在该锁上的重入次数。
* key的过期时间即为线程持有锁的时间

Tips：

使用不同方式调用lock的逻辑有一定程度区别:

* 设置lock的最长调用时间：Redisson将在Redis中创建一个在指定时间后过期的Hash对象
* 未指定lock最长持有时间：Redisson将定时更新Redis中的对象，保证其在用户程序主动调用前不会过期。Redisson采用HashedWheelTimer作为触发器控制程序的定时更新逻辑，关于HashedWheelTimer的说明请参考文章[HashedWheelTimer]

Redisson中全局线程id的生成方式：每个jvm实例中的UUID + jvm内部线程id。可保证在分布式环境下的线程唯一。

#### lock等待与通知

lock的等待通知机制依赖于Redis的pub/sub实现。当进程获取锁资源失败时，将会注册一个对应key的listen回调：当Redisson收到pub消息的时候，会通过内部的dispatch机制筛选并通知对应的listenner。此时listener将会再次尝试获取锁资源，与其他锁不同的是，Redisson中借助了java中的semaphore对此进行了优化，再次获取锁资源失败后不会再次注册listen，而是转为监听嵌入至listen的semaphore再次等待唤醒。

lock的等待与通知机制集中于方法subscribe()中
```java
    public RFuture<E> subscribe(String entryName, String channelName) {
        AtomicReference<Runnable> listenerHolder = new AtomicReference<Runnable>();
        //pubsubService中有一个简单的无冲突解决的AsyncSemaphore数据结构，这里只是简单从中获取一个
        AsyncSemaphore semaphore = service.getSemaphore(new ChannelName(channelName));
        //构造cancel的callback：当promise被cancell的时候，需要将AsyncSemaphore中的listener remove掉
        RPromise<E> newPromise = new RedissonPromise<E>() {
            @Override
            public boolean cancel(boolean mayInterruptIfRunning) {
                return semaphore.remove(listenerHolder.get());
            }
        };

        //构建一个基础的pub/sub listener并放置进入redis connection中
        Runnable listener = new Runnable() {

            @Override
            public void run() {
                E entry = entries.get(entryName);
                if (entry != null) {
                    entry.aquire();
                    semaphore.release();
                    entry.getPromise().onComplete(new TransferListener<E>(newPromise));
                    return;
                }
                
                E value = createEntry(newPromise);
                value.aquire();
                
                E oldValue = entries.putIfAbsent(entryName, value);
                if (oldValue != null) {
                    oldValue.aquire();
                    semaphore.release();
                    oldValue.getPromise().onComplete(new TransferListener<E>(newPromise));
                    return;
                }
                
                RedisPubSubListener<Object> listener = createListener(channelName, value);
                service.subscribe(LongCodec.INSTANCE, channelName, semaphore, listener);
            }
        };
        semaphore.acquire(listener);
        listenerHolder.set(listener);
        
        return newPromise;
    }
```

### unlock

相比lock,unlock的逻辑非常简单，依次执行以下操作即可：

1. 检查redis中的锁资源是否被当前线程所占有
2. 将redis中锁资源的重入数-1
3. 如果重入数等于0，则删除这个key，并向对应的queue中发送一条消息
4. 最后，删除key的不断renew机制


## 其他
Redisson的RedisClient类底层通信使用netty实现，其所有的命令均是利用netty的promise异步实现，详情清参考netty与源码。

