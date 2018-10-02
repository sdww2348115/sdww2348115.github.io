---
layout: post
title:  "Scheduler"
date:   2018-10-02 13:19:00 +0800
categories: quartz
permalink: /quartz/schduler
---

## Scheduler

Scheduler 可用于注册JobDetail与Trigger。一旦被注册，当JobDetail的Trigger达到条件触发时，Scheduler将负责执行Job

Scheduler实例由SchedulerFactory所生成，当Scheduler被SchedulerFactory所生产出来时，就已经被创建并初始化完成了。当一个Scheduler被创建后，它将处于就绪状态；只有当执行了start()方法后才能执行Job的调用。

Job由用户程序所创建，必须实现`org.quartz.Jon`接口。JobDetail将作为Job的独立实例被创建出来。JobDetail实例可通过scheduleJob(JobDetail, Trigger)或addJob(JobDetail, boolean)方法注册至Scheduler中

Trigger用于基于一定规则触发独立的Job实例。SimpleTrigger主要用于只触发一次的任务或者间隔固定时长所触发的任务；CronTrigger通常用于执行基于天、星期、月、年等间隔所执行的任务。

任何处于同一个Scheduler中的Job与Trigger都应使用name与group唯一标识。group功能对于创建逻辑分组或按照特征分组区分Job与Trigger非常有用。没有这方面需要的话可以使用接口中的常量DEFAULT_GROUP作为默认分组。

已经保存的Job可以通过triggerJob(String jobName, String jobGroup)手动触发执行。

用户程序同样可以使用listener接口注册自己感兴趣的事件。

* JobListener提供了Job执行事件的通知接口。
* TriggerListener提供了Trigger触发事件的通知接口。
* SchedulerListener提供了Scheduler的事件与错误通知接口。

Listeners可以通过ListenerManager接口进行关联和管理。