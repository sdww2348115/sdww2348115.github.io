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