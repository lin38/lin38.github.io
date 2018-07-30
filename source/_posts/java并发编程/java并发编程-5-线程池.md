---
title: java并发编程-5-线程池
date: 2018-07-30 11:00:40
categories: java并发编程
tags: 
  - 线程池
toc: true
list_number: false
---

# 1、**Executor框架**

​        为了更好的控制多线程，JDK提供了一套线程框架Executor，帮助开发人员有效地进行线程控制。它们都在java.util.concurrent包中，是JDK并发包的核心。其中有一个比较重要的类：Executors，它扮演着线程工厂的角色，我们通过Executors可以创建特定功能的线程池。

​        Executors创建线程池的方法：

1. newFixedThreadPool()方法：该方法返回一个固定数量的线程池。该方法的线程数始终不变，当有一个任务提交时，若线程池中有线程正在空闲，则立即执行，若没有，则会被暂缓在一个任务队列中等待有空闲的线程去执行。
2. newSingleThreadPool()方法：创建一个线程的线程池，若空闲则执行，若没有则暂缓在任务队列中。
3. newCachedThreadPool()方法：返回一个可根据实际情况调整线程个数的线程池，不限制最大线程数量，若有空闲的线程则执行任务，若无任务则不创建线程。并且每一个空闲线程会在60秒后自动回收。
4. newScheduledThread()方法： 创建一个定长线程池，支持定时及周期性任务执行。

<!--more-->



# 2、**自定义线程池**

```java
    public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler) {...}
```



# 3、**自定义线程池使用详细**

​        这个构造方法对于队列是什么类型的比较关键（BlockingQueue）：

1. 在使用有界队列时，若有新的任务需要执行，如果线程池实际线程数小于corePoolSize，则优先创建线程，若大于corePoolSize，则会将任务加入队列，若队列已满，则在总线程数不大于maximumPoolSize的前提下，创建新的线程，若线程数大于maximumPoolSize，则执行拒绝策略，或其他自定义方式。
2. 无界的任务队列时，LinkedBlockingQueue。与有界队列相比，除非系统资源耗尽，否则无界的任务队列不存在任务入队失败的情况。当有新任务到来，系统的线程数小于corePoolSize时，则新建线程执行任务。当达到corePoolSize后，就不会继续增加。若后续仍有新的任务加入，而没有空闲的线程资源，则任务直接进入队列等待。若任务创建和处理的速度差异很大，无界队列会保持快速增长，直到耗尽系统内存。

​        JDK默认实现的拒绝策略（RejectedExecutionHandler）： 

1. AbortPolicy：直接抛出异常，组织系统正常工作。
2. CallerRunsPolicy：只要线程池未关闭，直接在调用者线程中运行当前被丢弃的任务；若线程池已关闭，则直接抛弃任务。使用该策略时线程池饱和后将由调用线程池的主线程自己来执行任务，因此在执行任务的这段时间里主线程无法再提交新任务，从而使线程池中工作线程有时间将正在处理的任务处理完成。
3. DiscardOldestPolicy：丢弃最老的一个请求，尝试再次提交当前任务。
4. DiscardPolicy：丢弃无法处理的任务，不给予任何处理。

