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

Job由用户程序所创建，必须实现`org.quartz.Job`接口。JobDetail将作为Job的独立实例被创建出来。JobDetail实例可通过scheduleJob(JobDetail, Trigger)或addJob(JobDetail, boolean)方法注册至Scheduler中

Trigger用于基于一定规则触发独立的Job实例。SimpleTrigger主要用于只触发一次的任务或者间隔固定时长所触发的任务；CronTrigger通常用于执行基于天、星期、月、年等间隔所执行的任务。

任何处于同一个Scheduler中的Job与Trigger都应使用name与group唯一标识。group功能对于创建逻辑分组或按照特征分组区分Job与Trigger非常有用。没有这方面需要的话可以使用接口中的常量DEFAULT_GROUP作为默认分组。

已经保存的Job可以通过triggerJob(String jobName, String jobGroup)手动触发执行。

用户程序同样可以使用listener接口注册自己感兴趣的事件。

* JobListener提供了Job执行事件的通知接口。
* TriggerListener提供了Trigger触发事件的通知接口。
* SchedulerListener提供了Scheduler的事件与错误通知接口。

Listeners可以通过ListenerManager接口进行关联和管理。

## Job&JobDetail

JobDetail是Quartz框架所执行的任务实例，其核心方法为execute(JobExecutionContext context);在使用该类时需要注意以下特点:

* Scheduler将会在Trigger触发时根据设置新建Job实例并执行其exectute方法，执行完成后该实例将会被丢弃，被JVM所GC掉。

这样的特征带来两个衍生的限制：

1. 使用默认的JobFactory实现时，Job类必须含有默认的构造函数。
2. 在同一个job的不同执行时不能通过内置参数的方式进行数据传递。

解决Job执行间数据传递问题必须使用JobDataMap。JobDataMap实现了Map接口，通常可直接当做Map来使用。Job与Trigger都含有自己的JobDataMap。在job之间执行参数传递时，不能使用jobContext.getMergedJobDataMap().put()方法与不能使用jobContext.getTrigger().getJobDataMap.put()方法，只能使用jobContext.getJobDetail.getJobDataMap().put()。

使用JobDataMap时需要注意，如果采用了持久化地JobStore，Map中的对象将被序列化，这样可能会带来版本等一系列问题。例如：版本升级后无法反序列化之前的对象；某些对象无法被序列化等。在这种情况下，我们一般只往JobDataMap中保存JDBC对象或String这种不会随着版本变化的可序列化对象。

如果Job对象中定义了某个域的set方法，比如setName().当该job实例化的时候，JobFactory将尝试从map中获取key为name的value，并赋值给job实例。

每个Trigger也含有自己的DataMap，Job运行execute方法时，所获取的JobDataMap是Trigger与Job的DataMap合并后所得到的。注：JobDataMap合并的优先级为： Trigger > Job

### Job JobDetail & JobInstance

* Job：实现了Quartz中的Job接口，且用于执行某项工作的类，例如统计数据类
* JobDetail：在Job的基础上，通过properties与jobDataMap赋予Job实际工作，相对于job的例子就是设置统计玩家在线人数的统计类
* JobInstance：当Trigger触发后，由JobFactory通过Job的newInstance()方法所创建的实例

### Annotations

* @DisallowConcurrentExecution 基于JobDetail，同一JobDetail的不同实例在同一时刻只能有一个被执行（由于JobInstance由JobFactory于Trigger触发时创建，所以通用的synchronized等并发工具不能直接使用）
* @PersistJobDataAfterExecution 该注解同样以JobDetail为调度对象，每当execute()方法成功执行之后（未抛出异常）将JobDataMap中的所有内容持久化，该任务下一次执行的时候将直接读取持久化的数据并执行execute()方法。注：该注解最好搭配@DisallowConcurrentExecution注解使用，避免不必要的并发性问题

### Others

* Durability：如果一个job被设置为非durable的话，一旦没有任何Trigger关联至Job，该Job将被从Scheduler中自动删除
* RequestsRecovery ：如果一个任务被设置为RequestsRecovery，在执行的过程中出现可恢复的情况下，程序将尝试恢复上一次执行的状态并继续执行任务。job抛出异常不算可恢复，可恢复包括集群模式下某job抛出异常或者jvm崩溃
* 另：Job执行过程中的所有异常都应该被catch住，唯一允许被抛出的异常为：JobExecutionException

## Trigger

Trigger中各属性的含义

* JobKey Trigger触发时所执行的job，该Job必须被注册至Scheduler中
* startTime 表明Trigger开始的时刻。注：如果Trigger为每5分钟执行一次，startTime并不满足条件，Trigger不会立即触发，而是在下一次满足条件时触发
* endTime 超过该时刻后，Trigger不会再被触发。
* Priority 如果同一时刻有多个job等待执行，该参数用于表明多个job抢占线程资源时的优先级。