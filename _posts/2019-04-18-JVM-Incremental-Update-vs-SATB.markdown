---
layout: post
title:  "Incremental update vs. SATB"
date:   2019-04-18 23:39:01 +0800
categories: jvm
permalink: /jvm/IncrementalUpdateVsSATB
---

## 引言
作为JVM常用的垃圾回收算法，G1与CMS有许多相似之处，最大的共同点便是它们的主要步骤都是：

1. 初始标记（inital marking）
2. 并发标记（concurrent marking）
3. 最终标记（final marking/remarking）
4. 清理（cleanup）

虽然两者标记过程类似，但实际对于并发标记过程中对象引用状态改变的处理却完全不同。G1采用SATB模型，而CMS采用的是incrimental update模型。

* G1算法采用SATB模型：SATB即snapshot-at-the-beginning，G1会在一次GC开始时对堆中所有对象做一个抽象的"snapshot"，snapshot中存活的对象在整个GC过程中都是存活的。
* CMS算法采用incremental update模型：将GC过程中引用状态被修改的对象都记录下来，等到GC结束时再对遗漏的对象进行补充标记。