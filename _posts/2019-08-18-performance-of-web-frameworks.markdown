---
layout: post
title:  "几种常见java web框架吞吐性能比较"
date:   2019-08-18 00:14:00 +0800
categories: java web spring webflux vertx play
permalink: /network/performanceOfFrameworkds
description: 几种常见java web框架吞吐性能比较
---

## 测试背景

### 物理机

* i7-4790k + HDD 7200转 + 16GB内存
* ubuntu 19.4 Desktop版本操作系统

### 虚拟机

* 2核CPU + 2GB内存 + 5GB 磁盘（qcow2格式，cluster_size为65536）
* centos7 -1810 minimal版本
* open-jdk-11.0.4(为了兼容play 2.7)
* 采用虚拟网桥与物理机相连

### web框架

* web框架收到HTTP请求后立即返回内存中的String信息，不涉及任何外部IO操作（包括磁盘与网络）
* web框架的response字节数经过微调保证完全一致
* web框架运行于kvm虚拟机中，更换web框架时会重启虚拟机

### 测试软件

* 使用wrk 4.1.0-4-g0896020作为测试软件
* 线程数为4，连接数为8，每次测试持续时间1min
* 所有请求都为GET请求，无Body内容

## 测试流程

* web 框架的fat jar运行于虚拟机中，wrk程序运行于物理机上
* 每种框架将进行15轮测试：response body字节数依次从128增加至最大1920，用于模拟不同数据量情况下的IO吞吐效率
* 每轮测试持续时间为1min，取其QPS作为最终结果
* 每次进行框架切换时将重启虚拟机

## 测试结果

![performance](../resources/img/web-framework-performace-test.png)

## 简单分析

* 使用reactor模型的框架均优于传统的BIO模型框架（spring-web）
* 综合看来，vertx框架在小数据量时比较优秀；而spring webflux框架则更适合单次数据量交互较大的情况
* play框架代码编写、调试、运行的难度最高，但是性能表现与其他reactor模型框架相比相对平庸，适合对HTTP进行深度编程

## 总结与展望

reactor模型的web框架性能相比BIO优势明显，目前reactor模型框架应用较少的原因主要有以下几个：

1. 没有可靠的数据库等异步通信框架，无法整合入现有的eventloop线程模型中
2. spring-web包含了许多开箱即用的工具，vertx与webflux对应的生态相差较远
3. 异步模型编码和理解难度相比同步较大，同时坑也较多，学习难度相对较大

从JDK引入ForkJoin框架、Reactor编码模型、使用双枢轴快排替代原有经典快排算法等能够看出，今后程序的趋势是向计算机硬件设计架构靠拢，把程序的局部性原理贯彻到底，彻底榨干每一分硬件性能。因此，有理由相信web通信的明天是属于这些rector框架的。
