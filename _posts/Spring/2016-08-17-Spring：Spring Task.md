---
categories:
  - Technology
tags:
  - Spring
---

# 一 前置技能
首先看一下java.util.concurrent包中几个类的关系：
<img alt="ScheduledThreadPoolExecutor" src="http://o9l56z0kf.bkt.clouddn.com/image/blog/spring-task-scheduler.png" width=400/>

## 1.1 线程池ThreadPoolExecutor
该类是我们经常接触到的，他的完整的方法签名如下：

```java
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory,
                          RejectedExecutionHandler handler);
```
理解这几个参数的含义就理解了该类的工作流程，关于此类的介绍资料相当多，这里摘录主要流程如下：
1. 如果运行的线程少于 corePoolSize，则 Executor 始终首选添加新的线程，而不放入workQueue
2. 如果运行的线程等于或多于 corePoolSize，则 Executor 始终首选将请求加入队列，而不添加新的线程
3. 如果无法将请求加入队列，则创建新的线程，除非创建此线程超出maximumPoolSize，在这种情况下，任务将被拒绝

## 1.2 定时机制ScheduledExecutorService
ThreadPoolExecutor主要提供了线程池机制，但是我们实际中经常有需要定时执行任务的场景，为了提供定时或者周期执行任务的能力，java.util.concurrent包中增加了ScheduledExecutorService接口，该接口包括下面几个方法：

```java
public interface ScheduledExecutorService extends ExecutorService {

    public ScheduledFuture<?> schedule(Runnable command,long delay, TimeUnit unit);

    public <V> ScheduledFuture<V> schedule(Callable<V> callable,long delay, TimeUnit unit);

    public ScheduledFuture<?> scheduleAtFixedRate(Runnable command,long initialDelay,long period,TimeUnit unit);

    public ScheduledFuture<?> scheduleWithFixedDelay(Runnable command,long initialDelay,long delay,TimeUnit unit);
}
```
该接口的一个主要实现类是java.util.concurrent.ScheduledThreadPoolExecutor,关于ScheduledThreadPoolExecutor的实现原理需要大篇幅进行介绍，我理解的也不够深入，这里只简述下我的理解：
>所有预定执行的任务根据执行的时间顺序放入一个DelayedWorkQueue中，该DelayedWorkQueue会按照执行时间的优先级排序。线程池取队列中的首个任务，也就是最早要被执行的任务，如果还没到执行时间的话，取Queue元素会被阻塞，直至可以被执行或者有更提前的任务加入。对于周期性任务除了将最近一次要执行的加入队列里面还会将下一次执行任务加入队列里面，从而能够保证周期性进行。

## 1.3 勿忘前辈Timer和TimerTask
* Timer是基于绝对时间的延时执行或周期执行，当系统时间改变，则任务的执行会受到的影响。而ScheduleThreadPoolExecutore中，任务时基于相对时间进行周期或延时操作
* Timer也可以提交多个TimeTask任务，但只有一个线程来执行所有的TimeTask，这样并发性受到影响。而ScheduleThreadPoolExecutore可以设定池中线程的数量
* Timer不会捕获TimerTask的异常，只是简单地停止，这样势必会影响其他TimeTask的执行。而ScheduleThreadPoolExecutore中，如果一个线程因某些原因停止，线程池可以自动创建新的线程来维护池中线程的数量。


上面简单介绍了Java世界中最常用的线程池ThreadPoolExecutor和定时执行机制ScheduledThreadPoolExecutor这两个java类,Spring中Task的实现主要就是基于这两点，ThreadPoolExecutor用来实现@Async注解的异步任务，ScheduledThreadPoolExecutor用来实现@Scheduled注解的定时任务，当然也有其他类型的ThreadPool和SScheduledExecutorService实现。

有了这些前置知识，下面来进一步看下Spring Task。

# 二 中期加点

## 2.1 TaskExecutor(本质就是java.util.concurrent.Executor)

几种实现：
* SimpleAsyncTaskExecutor
>1. 不适用线程池，每个任务都单独启一个线程来异步执行
>2. 可以设置线程并发的数目，默认情况下线程数目可以无限多
```java
protected void doExecute(Runnable task) {
  Thread thread = (this.threadFactory != null ? this.threadFactory.newThread(task) : createThread(task));
  thread.start();
}
```
* SyncTaskExecutor
>1. 非异步方式执行
```java
@Override
public void execute(Runnable task) {
  Assert.notNull(task, "Runnable must not be null");
  task.run();
}
```
* ThreadPoolTaskExecutor:本质就是对java.util.concurrent.ThreadPoolExecutor的封装
* ConcurrentTaskExecutor
* WorkManagerTaskExecutor

## 2.2 TaskScheduler

首先看下TaskScheduler接口包含的函数列表，它和ScheduledExecutorService接口很类似，因此可以猜到TaskScheduler的主要实现就是依赖于ScheduledExecutorService。
```java
public interface TaskScheduler {

    ScheduledFuture schedule(Runnable task, Trigger trigger);

    ScheduledFuture schedule(Runnable task, Date startTime);

    ScheduledFuture scheduleAtFixedRate(Runnable task, Date startTime, long period);

    ScheduledFuture scheduleAtFixedRate(Runnable task, long period);

    ScheduledFuture scheduleWithFixedDelay(Runnable task, Date startTime, long delay);

    ScheduledFuture scheduleWithFixedDelay(Runnable task, long delay);

}
```
与我们使用最相关的是含有Trigger参数的这个接口，类似于各种Context，Trigger也有一个TriggerContext,记录的主要是任务上次执行的情况，从而方便要根据上次定时任务执行情况决定下次定时任务执行时间的应用场景。
```java
public interface TriggerContext {

    Date lastScheduledExecutionTime();

    Date lastActualExecutionTime();

    Date lastCompletionTime();
}
```
Trigger的主要实现是CronTrigger,使用示例如：
```java
scheduler.schedule(task, new CronTrigger("0 15 9-17 * * MON-FRI"));
```

__ThreadPoolTaskScheduler__

TaskScheduler有多个实现，其中ThreadPoolTaskScheduler是我们最常用的，配置文件中“task:scheduler”这个就代表的创建ThreadPoolTaskScheduler类型的bean。ThreadPoolTaskScheduler内部使用的java.util.concurrent.ScheduledExecutorService(实例为java.util.concurrent.ScheduledThreadPoolExecutor)来分配执行各种任务。ThreadPoolTaskScheduler同时实现了TaskExecutor,因此也可用于AsyncTask执行。

- 思考：如果既有@Scheduled定时任务,也有AsyncTask,是否有必要同时配置task:executor和task:scheduler?

## 2.3 整体机制

下面是常用的xml配置文件的形式：
```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:task="http://www.springframework.org/schema/task"
    xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/task http://www.springframework.org/schema/task/spring-task-3.0.xsd">
    <task:executor id="wishNumExecutor" pool-size="40" />
    <task:scheduler id="scheduler" pool-size="10" />
    <task:annotation-driven scheduler="scheduler" />
</beans>
```

- 思考：xsd文件的作用？task:annotation-driven的解析过程？
> 在META-INF中包含spring.handles配置文件，指出使用什么Handler处理解析xml中bean的定义

- @Async的解析工作流程
 * AsyncAnnotationBeanPostProcessor是一个BeanPostProcessor，解析出所有@Async注解的方法
 * 使用AOP的方式处理@Async注解的方法
- @Scheduled的解析工作流程
 * ScheduledAnnotationBeanPostProcessor是一个BeanPostProcessor，解析出所有@Scheduled注解的方法，加入到ScheduledTaskRegistrar中的List里面
 * ScheduledTaskRegistrar中会使用 __TaskScheduler__ 来执行List里面的所有任务

- 思考：如何列出系统中所有@Scheduled(cron=...) 这种注解的方法，并按照发生时间顺序排序？

# 三 后期总结

* SpringTask的实现主要依赖于java.util.concurrent包中线程池的实现和定时任务的实现
* SpringTask提供的@Async注解主要用于执行异步的任务，执行异步任务的线程池默认的是java.util.concurrent.ThreadPoolExecutor，@Async的实现方式类似于AOP
* SpringTask提供的@Scheduled注解主要用于定时触发的任务，包括延迟触发，周期触发等任务，定时功能的实现依赖于java.util.concurrent.ScheduledExecutorService，使用的线程池也是java.util.concurrent.ThreadPoolExecutor，与大多ThreadPoolExecutor的主要区别是使用的 __阻塞__ 的，有 __优先级__ 的DelayedWorkQueue，来完成定时任务的实现


# 四 参考资料
- [ThreadPoolExecutor](http://dongxuan.iteye.com/blog/901689)
- [ScheduledThreadPoolExecutor的实现原理](http://freish.iteye.com/blog/1766960)
- [ScheduledThreadPoolExecutor的实现原理](http://ju.outofmemory.cn/entry/25473)
- [Task Execution and Scheduling](http://docs.spring.io/spring/docs/current/spring-framework-reference/html/scheduling.html)
