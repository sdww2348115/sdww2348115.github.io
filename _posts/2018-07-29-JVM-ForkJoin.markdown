---
layout: post
title:  "ForkJoin"
date:   2018-07-29 20:42:00 +0800
categories: concurrent
permalink: /concurrent/ForkJoin
---

## Fork/Join框架是什么

Fork/Join框架是由并发大师Doug Lea所开发的一种新增并发编程模型，它提供了一种通过（递归的）把问题划分为子任务，然后并行的执行这些子任务，等所有的子任务都结束的时候，再合并最终结果的这种方式来支持并行计算编程的模型。此模型已经成为了许多JDK功能所依赖的基础，其中包括：JDK8所提供的Stream，ComputableFuture以及JDK9中所提供的FlowAPI等。

该框架所包含的类主要有：
 * ForkJoinPool：Fork/Join框架的调度器
 * ForkJoinTask：Fork/Join框架执行任务的抽象
 * RecursiveAction：ForkJoinTask的子类，用于处理没有返回值的计算
 * RecursiveTask：ForkJoinTask的子类，用于处理有返回值的计算
 
## Fork/Join框架诞生的背景

在Fork/Join框架出现前，对于一个递归任务，我们通常采用的方式是直接调用递归算法处理它，这里以统计某目录下的文件数量为例，代码如下：

``` java
public static Integer countFile(File root) {
    //递归退出条件:root为文件
    if (root.isFile()) {
        return 1;
    //如果root为目录，则需要递归统计该目录下的所有目录与文件
    } else {
        int sum = 0;
        for (File file : root.listFiles()) {
            sum += countFile(file);
        }
        return sum;
    }
}
```

代码将以单线程方式递归处理root目录，直到统计出该目录下含有的文件总个数为止。

如果某个目录下的文件数量特别大，导致该任务执行时间很长，我们应如何优化此类计算？

### 并行化

上述代码执行起来最大的瓶颈在于只能单线程运行，不能充分发挥多线程的优势。考虑文件目录结构如下（不考虑目录树出现回环的情况）：

![fileSystem1](../resources/img/fileSystem1.png)

如上图所示，root目录下含有多个目录，此时如果能够采用并行方式在线程1，线程2...上分别调用countFile(dir1),countFile(dir2)...就能充分利用CPU资源，利用多核的性能优势能够大大减少程序所运行的时间。使用代码表示如下：

```java
public static Integer countFileRoot(File root) {
    if (root.isFile()) {
        return 1;
    }
    AtomicInteger sum = new AtomicInteger(0);
    CountDownLatch countDownLatch = new CountDownLatch(root.listFiles().length);
    for(File file : root.listFiles()) {
        //并发执行
        new Thread(new Runnable() {
            @Override
            public void run() {
                sum.getAndAdd(countFile(file));
                countDownLatch.countDown();
            }
        }).start();
    }
    countDownLatch.await();
    return sum.get();
}
```

### 负载均衡

上面的parallelCount方法存在严重的缺陷：各工作线程所分得的任务不均匀。考虑目录树结构如下所示：

![fileSystem2](../resources/img/fileSystem2.png)

其中dir2，dir3，dir4均是空目录，所有的计算量仍然集中于运行countFile(dir1)的线程上。其执行效率与单线程递归并无多大提升，由于创建了多个线程，效率甚至还不如单线程递归模式。此方案效率不高的最根本原因是：海量任务集中于某一个线程上，其他工作线程处于闲置状态，结果仍然是无法充分利用CPU资源。

![weiguan](../resources/img/weiguan.png)

如果能够将所有的任务`平均`分给许多工作线程， 这样执行起来岂不很棒？以自然语言的方式描述起来如下：假设我们有一个任务池存放着所有的工作任务，每个线程按以下方式进行工作：

 1. 从任务池中获取一个任务，按以下方式进行计算
 2. 如果任务所依赖的子任务全部计算完成，该任务可直接计算得到结果，则直接执行，然后回到1
 3. 如果任务需要被划分为若干个更小的任务，则将这若干个更小的任务全部放入任务池，然后回到1
 4. 反复执行该过程直到所有任务执行完毕
 
这与线程池的语义不是差不多吗？这个简单

```java
//由于要阻塞等待subTask的完成，所以必须使用无界的newCachedThreadPool
private static ExecutorService threadpool = Executors.newCachedThreadPool();
public static Integer countFileByTask(File root) {
    if (root.isFile()) {
        return 1;
    }
    List<Future<Integer>> futures = new LinkedList<>();
    for (File file : root.listFiles()) {
        Future<Integer> future = threadpool.submit(new Callable<Integer>() {
            @Override
            public Integer call() throws Exception {
                return countFileByTask(file);
            }
        });
        futures.add(future);
    }
    int sum = 0;
    for(Future<Integer> future : futures) {
        sum += future.get();
    }
    return sum;
}
```

想法很丰满，现实很骨感。经过试验，这种执行方式的效率是所有执行方式中最慢的。

仔细分析countFileByTask方法：其核心计算逻辑只有一个return 0以及将各子任务的值进行累加，但是却引入了对每一个子目录/文件创建Thread的过程！创建Thread的开销远远大于简单累加的开销，更不用说当Thread实例数量爆炸之后操作系统调度所带来的开销以及GC的时间开销。

事实上，使用这种方式解决问题的隐含着将task转换为了thread的过程，程序分配的事实上是thread而非task。


## Fork/Join框架的实现之道

### 功能实现