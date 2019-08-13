---
title: Java并发框架：Executor简析
date: 2018-05-10 23:51:32
tags: CSDN迁移
---
 版权声明：尊重原创，转载请注明出处 [ https://blog.csdn.net/abc123lzf/article/details/80272956]( https://blog.csdn.net/abc123lzf/article/details/80272956)   
  ### 一、简介

 很多刚刚接触多线程的同学喜欢通过new Thread().start()来创建并启动一个线程，在复杂的应用中这是一个很不好的习惯，这样创建出来的线程往往缺乏有效的管理，容易造成各种各样的问题并难以解决。当你为此感到困惑的时候，你就应该好好学习并使用这个框架来构建多线程应用程序了。

 Executor框架是java.util.concurrent中的一个线程管理工具，它是一个灵活强大的异步任务执行框架。

 首先我们应该先了解几个概念： **（1）线程池**   
 从字面意义上来看，是指管理一组同构工作线程的资源池。简单来说，线程池的作用就是限制系统中执行线程的数量。   
 相比直接创建Thread对象并执行，线程池有很多优点：   
 1、可以根据系统具体的配置，自定义一个合适的线程数量，防止因为创建过多的线程对象耗费过多的系统内存或造成程序的崩溃。   
 2、减少了创建和销毁线程的次数，线程池中的线程可以反复利用，执行提交的任务。   
 3、该框架基于生产者-消费者模式，可以很好的将任务的提交和任务的实际执行解耦。   
 4、可以方便地对其中的线程进行各种管理。   
 **（2）工作队列**   
 工作队列由Executor管理，其中工作队列保存了所有等待执行的任务。它的任务非常简单：从工作队列中获取一个任务，执行任务，然后返回线程池等待下一个任务，它不会像线程那样去竞争CPU资源。工作队列排列方法有三种：有界队列、无界队列、同步移交。   
 可以看成这样：   
 ![这里写图片描述](https://img-blog.csdn.net/20160917104549831)   
 图片转自[https://blog.csdn.net/yanyan19880509/article/details/52562039](https://blog.csdn.net/yanyan19880509/article/details/52562039)

 下面是Executor框架常用的接口及其子类： ![这里写图片描述](https://images2015.cnblogs.com/blog/776259/201604/776259-20160426201537486-1323529733.png)   
 图片转自[https://www.cnblogs.com/MOBIN/p/5436482.html](https://www.cnblogs.com/MOBIN/p/5436482.html)

 
### 二、配置线程池

 
#### 1、创建线程池

 
##### （1）工厂方法构建线程池

 我们可以通过Executors中的静态方法来创建一个ThreadPoolExecutor：   
 **newFixedThreadPool(int nThread)**   
 创建一个固定长度为nThread的线程池，每当提交一个任务就创建一个新的线程，直到达到最大数量，这个时候线程池的规模将不再变化。如果线程抛出一个未预期的异常，那么该线程池也会重新创建一个新的线程   
 **newCacheThreadPool()**   
 创建一个可缓存的线程池，该线程池的规模不存在限制。如果该线程池的规模超过了处理需求时，那么会自动回收空闲的线程，处理需求增加时也会创建一个新的线程池。   
 **newSingleThreadExecutor()**   
 创建一个工作者线程来执行当前任务，如果这个线程异常结束那么会重新新建一个线程，这将会保证以串行方式执行任务（无需保证其执行任务的线程安全性）   
 **示例代码：**

 
```
ExecutorService exec = Executors.newFixedThreadPool(16);
```
 
##### （2）自定义线程池

 **ThreadPoolExecutor构造函数如下：**

 
```
public ThreadPoolExecutor(int corePoolSize,     //线程池基本大小
                          int maximumPoolSize,  //线程池最大大小
                          long keepAliveTime,   //存活时间
                          TimeUnit unit,        
                          BlockingQueue<Runnable> workQueue, //任务队列
                          ThreadFactory threadFactory, //定义线程工厂
                          RejectedExecutionHandler handler) //饱和策略
                          //后面两个参数为可选参数
```
 线程池基本大小、最大大小以及存活时间分别共同负责线程的创建和销毁。基本大小是没有任务执行时线程池的大小，只有在工作队列已满的状态下才会创建超过这个数量的线程。最大大小表示线程数量的上限。   
 ThreadPoolExecutor允许提供一个BlockingQueue来保存等待执行的任务。newFixedThreadPool(int nThread)和newSingleThreadExecutor()默认采用了无界队列。如果任务到达速度超过了线程处理它们的速度，那么队列的长度会无限增加，有可能会因此而导致系统的崩溃。   
 一种更稳妥的方式是采用ArrayBlockingQueue或有界的LinkedBlockingQueue、PriorityBlockingQueue。可以有效地防止资源耗尽的情况发生。那么新的问题来了：队列填满后新的任务怎么办？这里我们就要提到饱和策略了   
 **饱和策略**   
 ThreadPoolExecutor的饱和策略可以通过调用setRejectedExecutionHandle来修改。Java提供了四种饱和策略：AbortPolicy、CallerRunsPolicy、DiscardPolicy、DiscardOldestPolicy。   
 AbortPolicy（中止策略）是默认的饱和策略，该策略将抛出未检查的RejectedExecutionException异常。调用者可捕获该异常并根据自己的需求编写处理的代码。   
 CallerRunsPolicy（调用者运行策略）：该策略既不会抛弃任务，也不会抛出异常，而是将任务回退给调用者（比如回退给main线程，由main线程自行处理）。   
 DiscardPolicy（抛弃策略）：当工作队列已满并无法添加，抛弃策略会悄悄地扔掉该任务。   
 DiscardOldestPolicy（抛弃最旧策略）：该策略会抛弃下一个执行的任务并尝试重新提交新的任务，如果工作者队列是优先队列，那么将抛弃优先级最高的队列（所以不要和优先队列同时使用）。   
 **线程工厂**   
 每当线程池创建一个新的线程时，都是通过线程工厂方法来完成的。通过制定一个自定义的ThreadFactory，可以定制线程池的配置信息，例如给线程指定一个UncaughtExceptionHandler，或是取个名字，或是实例化一个定制的Thread用来执行调试信息。   
 ThreadFactory接口只定义了一个newThread方法，每当线程池创建线程都会调用这个方法。   
 具体方法可以参照这篇博客：[https://blog.csdn.net/wxwzy738/article/details/8520630](https://blog.csdn.net/wxwzy738/article/details/8520630)

 
#### 2、构造完成后修改线程池配置

 线程构造完成后仍然可以通过setter函数来修改大多数传递给它的构造函数参数，如线程池基本大小、最大大小、存活时间、线程工厂及RejectedExecutionHandle（拒绝执行处理器）   
 如果不希望构造完成后被修改，可以通过unconfigurableExecutorService进行包装。

 
#### 3、扩展线程池

 ThreadPoolExecution提供了几个protected方法：beforeExecute、afterExecute、terminated，可以在子类对这些方法进行重写以扩展   
 **afterExecute**   
 无论任务是正常返回还是抛出异常而返回，afterExecute都会调用   
 **beforeExecute**   
 与afterExecute相反，该方法在人物执行前执行，如果抛出RuntimeException则不会执行任务   
 **terminated**   
 关闭线程池后自动调用该方法   
 具体可参照这篇博客：[https://blog.csdn.net/zhongxiangbo/article/details/70882309](https://blog.csdn.net/zhongxiangbo/article/details/70882309)

 
### 三、提交任务

 Executor接口提供了Runnable任务的方法，Executor接口扩展了ExecutorService，提供更广泛的线程管理方法：包含关闭其中管理的线程，提交Runnable、Callable任务、设定任务时限、判断该线程池是否关闭等方法。

 
```
public interface ExecutorService extends Executor 
{
    void shutdown();
    List<Runnable> shutdownNow();
    boolean isShutdown();
    boolean isTerminated();
    boolean awaitTermination(long timeout, TimeUnit unit) throws InterruptedException;
    <T> Future<T> submit(Callable<T> task);
    <T> Future<T> submit(Runnable task, T result);
    Future<?> submit(Runnable task);
    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks) throws InterruptedException;
    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks,
                                  long timeout, TimeUnit unit) throws InterruptedException;
    <T> T invokeAny(Collection<? extends Callable<T>> tasks)throws InterruptedException, ExecutionException;
    <T> T invokeAny(Collection<? extends Callable<T>> tasks, long timeout, TimeUnit unit)
                                  throws InterruptedException, ExecutionException, TimeoutException;
}
```
 
##### 提交Runnable任务方法

 创建一个任务类并继承Runnable接口,然后通过调用ExecutorService接口的execute(Runnable command)提交   
 例如:

 
```
private ExecutorService exec = Executors.newFixedThreadPool(16);
private class TestTask implements Runnable{
    public void run(){}
}
private void submitTask(){
    exec.execute(new TestTask());
}

```
 
##### 提交Callable任务和接收返回值

 提交方法有两种   
 （1）创建一个任务类并继承Callable接口,然后通过调用ExecutorService接口的submit(Callable<T> task)提交。   
 （2）采用invokeAll方法：将所有任务添加到一个List集合中，然后调用invokeAll方法提交，该方法会返回一个封装有Future的List集合。   
 由于Callable任务具有返回值，我们可以通过BlockingQueue来保存Future<T>对象，并在提交完成后调用get方法来获取返回值。   
 这里只给出第一种提交方法：

 
```
private ExecutorService service = Executors.newFixedThreadPool(16);
private BlockingQueue<Future<Integer>> queue = new ArrayBlockingQueue<>(100);
private class CalcTask implements Callable<Integer>{
        public Integer call() {
            int sum = calc(); //calc()为某个计算方法
            return sum;
        }
}
private void submitTask(){
    Future<Integer> future = service.submit(new CalcTask());
    Integer result = future.get();
    System.out.println(result);
}

```
 
### 四、关闭线程池

 Executor是以异步方式来执行任务的，因此在任何时刻，之前提交的任务不是立即可见的。有些任务已经完成，有些任务可能正在运行，也有些任务可能在工作队列中等待执行。既然Executor是为应用程序服务的，那么它们应该是可关闭的   
 为了解决生命周期问题，我们可以使用ExecutorService接口中定义的几个管理方法。

 Executor的生命周期有三种状态：运行、关闭、终止。

 
##### 下面简单介绍几个关闭Executor的方法：

 **shutdown()**   
 shutdown方法会采用一种平缓地关闭Executor的方法：不再接收新的任务，同时等待已经提交的任务执行完成。   
 **shutdownNow()**   
 采用一种粗暴的关闭方式：尝试取消正在执行的任务，包括已提交还未开始执行的任务。执行该方法会返回所有尚未启动的任务清单。如果希望获取正在运行但被强制关闭的任务，可以在任务run()方法中使用一个finally，当检测到isTerminated为真时将该任务添加至一个指定的集合。   
 **awaitTermination(long timeOut, TimeUnit unit)**   
 该方法与shutdown方法类似，但是awaitTermination会使调用该方法线程的阻塞直到线程池终止或者时间超限。该方法会返回boolean，代表是否停止。

 记住，为了保证良好的封装性，不要直接通过Thread来终止线程，应使用线程池来终止线程。

 如果关闭ExecutorService后继续提交任务，那么将由RejectedExecutionHandle（拒绝执行处理器）来处理。它会抛弃任务并抛出一个未检查的RejectedExecutionException异常。所有任务完成后，ExecutorService转入终止状态。可以调用isTerminated()方法来检查是否已经终止。

   
  