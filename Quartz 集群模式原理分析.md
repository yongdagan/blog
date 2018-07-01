# Quartz 集群模式原理分析

### 概述
Quartz 是一个基于Java的作业调度系统，支持各种简单和复杂调度规则。Quartz的两个基本概念：
>- Job：调度任务，定义需要做什么。
>- Trigger：触发器，定义关联的Job在何时触发。可以是简单的时间区间重复调度，也可以是基于CronExpression定义。

Quartz 支持单机模式和集群模式。单机模式的调度保存在内存，集群模式将调度持久化到DB，因此多个相同的应用实例可以共同承担JOB的调度。Quartz的集群模式通过DB管理不同JOB的触发，不同实例通过争抢DB的数据库行锁来保证不会出现DB数据的覆盖，每次对DB的操作都需要获取到相应的锁。

### Quartz集群数据表
Quartz数据表是集群模式的基础，官方提供基于不同数据库的建表SQL（存放路径在官方JAR包docs/dbTables）。Quartz包含多个数据表，以cronExpression的调度为例，主要涉及到以下几个表：
>- quartz_locks：包含两个常量，STATE_ACCESS用于调度实例管理时获取数据行锁；TRIGGER_ACCESS用于获取trigger时请求的数据行锁。
>- quartz_job_details：保存调度系统包含的所有JOB信息
>- quartz_triggers：保存已配置的trigger信息
>- quartz_cron_triggers：保存已配置的cron_trigger信息，是上面的子集，但存储的信息不同
>- quartz_fired_triggers：包含将要触发的trigger信息，在调度的过程中，主要是改变这个表的值。

### Quartz调度基础概念
Quartz初始化SchedulerFactory（默认是StdSchedulerFactory）时会实例化调度资源QuartzSchedulerResources和核心调度类QuartzScheduler，其中QuartzScheduler是整个调度的框架。

涉及JOB调度执行的主要有两个部分：
>1. QuartzScheduler初始时会创建QuartzSchedulerThread作为调度的主线程，所有的JOB调度触发都由此线程负责；
>2. QuartzSchedulerResources关联的线程池（一般是）SimpleThreadPool,包含实际执行JOB的工作线程WorkerThread，线程数量可配置。

### Quartz集群调度流程
Quartz集群调度基于数据库行锁（select ... where ... for update），每次请求操作trigger前需要获取对应行锁，因此同一时间只有一个调度实例（机器）能操作trigger，避免数据的写覆盖。具体流程如下：
>1. 首先请求trigger_access数据库行锁，若行锁已被占用则阻塞直至获取到
>2. 获取状态为waiting且30秒内可触发的trigger，更新其状态为acquiring，并创建对应的fired_trigger
>3. 提交事务，释放trigger_access行锁
>4. 请求trigger_access行锁，并校验job和trigger是否还有效
>5. 更新fired_trigger状态为executing，以及修改trigger状态为waiting并刷新下次触发时间
>6. 释放trigger_access行锁
>7. 根据各个fired_trigger构造出job，并扔到线程池分别分配给各个workThread执行。
>8. 执行job
>9. 获取trigger_access，删除对应的fired_trigger后，释放锁。

![Quartz调度流程][1]

### 总结
使用quartz集群模式可实现分布式情况下的任务调度，其基于数据库悲观锁的实现可有效避免由于不同实例的竞争而导致的写覆盖。但每次的调度操作都涉及悲观锁阻塞，因此并发度并不高。

  [1]: http://static.zybuluo.com/yongdagan/5sq9br4gi17z6c0b439hjl1b/Quartz%E8%B0%83%E5%BA%A6%E6%B5%81%E7%A8%8B%20%282%29.png
