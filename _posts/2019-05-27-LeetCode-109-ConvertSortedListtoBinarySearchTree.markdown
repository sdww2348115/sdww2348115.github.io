---
layout: post
title:  "ARTS-6th"
date:   2019-05-06 23:06:00 +0800
categories: ARTS
permalink: /arts/6
description: 左耳听风ARTS第6周
---

## Algorithm

[Word-Subsets](../../leetcode/916)

## Review

本周阅读：[The Garbage Collection Handbook]

章节：Incremental update solutions + Snapshot-at-the-beginning solutions

本文主要描述JVM中的CMS垃圾收集算法中Incremental Update与G1中的SATB

虽然CMS与G1都有并发标记阶段，但它们的处理方式却各不相同：

 * SATB的处理方式为：借助write barrier当一个白对象与持有其引用的对象之间关系断裂时，将该白对象标记为灰色。待下一阶段最终标记时，再以这些灰色对象为根进行一次扫描，即可保证所有存活的对象都不会被漏扫。

 * CMS的Incremental update处理方式则为：当一个黑对象的字段与一个白对象建立引用关系时，将该白对象置灰。由于存在着将白色对象放置于mutator的行为，仅扫描这些新增灰对象仍然有可能导致对象漏扫。
 
因此，CMS的Incremental update在执行完成并发标记过程后，在final mark/remark 阶段将再次对整个堆空间进行重新扫描。

## Tip

使用G1垃圾回收器时，需要特别注意其Region数量与大小。G1的Region应比应用中的大多数对象都大，避免对象直接被分配至Huge Region中，导致对象得不到释放。


## Share

[分代式GC算法与CardTable](../../jvm/gc/generationGc)