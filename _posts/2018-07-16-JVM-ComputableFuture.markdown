---
layout: post
title:  "ComputableFuture"
date:   2018-07-16 23:37:01 +0800
categories: concurrent
permalink: /concurrent/ComputableFuture
---

## Future

JDK自1.5起引入了Future接口用于描述异步计算结果。Future提供了多种方法用于获取异步计算结果或强行中断task，典型用法如下：
``` java
public static void FutureExample() {
    Future future = executor.submit(task);
    try {
        future.get();
    } catch (InterruptedException e) {
        ...
    }
}
```

Future孱弱的语义在JDK1.8中得到了增强：通过ComputableStage接口的ComputableFuture类。它帮助我们简化异步编程的复杂性，提供了函数式编程的能力，可以通过回调的方式处理计算结果，并且提供了转换和组合CompletableFuture的方法。

 * ComputableFuture提供了使Future完成的方法。
 * ComputableFuture提供了then、combine等新的语义，使得并发编程更为简单。

## 如何使用

与JDK8所提供的流式计算一样，使用ComputableFuture是一种完全不同的体验。
首先，忘掉Thread、Executor、线程竞争与同步等概念，让我们以一种另外的视角来看待我们的程序。每一段程序都可以被分解为许许多多简单的小任务，把这些小任务按照一定的顺序和逻辑组织编排起来，就能使计算机按照我们想要的方式运行。

以一个简单的程序为例：程序接受一个字符串输入，然后程序会去掉A的前后空格，最后将整个字符串都转换为小写并输出。
程序的运行过程如下：
![programe1](../resources/img/programe1.png)