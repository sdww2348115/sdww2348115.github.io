---
layout: post
title:  "Incremental update vs. SATB"
date:   2019-04-18 23:39:01 +0800
categories: jvm
permalink: /jvm/IncrementalUpdateVsSATB
description: G1算法的SATB与CMS中的IncrementalUpdate之间的区别
---

## 引言
作为JVM常用的垃圾回收算法，G1与CMS有许多相似之处，最大的共同点便是它们的主要步骤都是：

1. 初始标记（inital marking）
2. 并发标记（concurrent marking）
3. 最终标记（final marking/remarking）
4. 清理（cleanup）

虽然两者标记过程类似，但实际对于并发标记过程中对象引用状态改变的处理却完全不同。G1采用SATB模型，而CMS采用的是incrimental update模型。

## 并发标记阶段的问题
无论CMS还是G1，对于比较耗时的堆扫描阶段都采用了并发扫描的方式，即不产生StopTheWorld，GC线程与工作线程并发执行。当GC线程对整个堆进行三色标记时，工作线程可能会修改其中某些对象的引用关系，当遇到如下情况时，如不加入额外处理将会导致不该被回收的对象被错误回收。

并发标记阶段会导致对象被误扫描有两个条件：
1. 将白对象的引用存到黑对象的字段中
2. 某白对象失去所有能从灰对象到达它的引用路径（直接或间接）

其中条件1是充要条件，一旦发生一定会导致漏标情况发生；条件2是条件1的前提条件之一，出现条件2不一定会导致漏标。因为此时该对象确实已经无法到达，可以进行回收了。

CMS的Incremental update主要针对条件1进行处理；SATB主要针对条件2进行处理。因此SATB的精确度低于Incremental update。

SATB的处理方式为：借助write barrier当一个白对象与持有其引用的对象之间关系断裂时，将该白对象标记为灰色。待下一阶段最终标记时，再以这些灰色对象为根进行一次扫描，即可保证所有存活的对象都不会被漏扫。

CMS的Incremental update处理方式则为：当一个黑对象的字段与一个白对象建立引用关系时，将该白对象置灰。由于存在着将白色对象放置于mutator的行为，仅扫描这些新增灰对象仍然有可能导致对象漏扫。

假设灰对象A有3个字段a,b,c；其中a与b字段的值均已得到了标记，此时mutator将白对象B放置于对象A的a字段中，GC线程执行完标记c后就会把A对象由灰变为黑（已完成标记）。此时黑对象A却持有了白对象B的引用，B以及其相关联的一系列对象都将被错误回收。

因此，CMS的Incremental update在执行完成并发标记过程后，在final mark/remark 阶段将再次对整个堆空间进行重新扫描。

## SATB

SATB即snapshot-at-the-beginning，G1会在一次GC开始时对堆中所有对象做一个抽象的"snapshot"，snapshot中存活的对象在整个GC过程中都是存活的。SATB的实现主要有两点：

1. 在GC过程中所有新增的对象都是存活的
2. 持有自身引用的对象变化后，自己一定是存活的

### 新增对象存活性保证
G1 垃圾回收器使用两个bitmap以copyOnWrite的方式记录当前Region中所有对象的存活状态：

* prevBitmap：用于记录第n-1轮concurrent marking所得的对象存活状态。由于第n-1轮concurrent marking已经完成，该bitmap的信息可直接使用
* nextBitmap：用于记录第n轮concurrent marking的结果。该bitmap是当前将要或正在进行concurrent marking的结果，尚未完成，不能使用。

对应于此，每个region都有如下指针：

|<-- (1) -->|<-- (2) -->|<-- (3) -->|<-- (4) -->|
bottom    prevTAMS    nextTAMS     top         end

其中top是该region的当前分配指针，[bottom,top)是当前region已用（used）部分，[top, end)是尚未使用的可分配空间（unused）。 
(1): [bottom, prevTAMS): 这部分里的对象存活信息可以通过prevBitmap来得知 
(2): [prevTAMS, nextTAMS): 这部分里的对象在第n-1轮concurrent marking是隐式存活的 
(3): [nextTAMS, top): 这部分对象即此轮concurrent marking时所新建的，隐式存活

因此，最终标记阶段Jvm只需要对nextTAMS与top之间的对象进行继续标记即可。