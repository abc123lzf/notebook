---
title: java.util.concurrent.Semaphore（信号量）原理解析
date: 2018-09-11 21:38:18
tags: CSDN迁移
---
 版权声明：尊重原创，转载请注明出处 [ https://blog.csdn.net/abc123lzf/article/details/82597077]( https://blog.csdn.net/abc123lzf/article/details/82597077)   
  ### 一、引言

 Semaphore又称信号量，用来控制同时访问访问某个特定资源的操作数量，可以理解为是用于控制线程访问数量的工具类。   
 Semaphore管理着一组许可(permit)，许可的数量可通过构造函数来指定。在执行某个操作时必须先获得许可，并在使用完毕后释放(release)许可。如果没有许可，那么尝试获取许可的线程会进入阻塞状态直到有许可可以使用。

 可以想象成一个房间，房间里只能容纳3个人(每个人相当于一个线程)。现在外面的人想要进来(尝试获取许可)，如果房间没有满，则可以直接进入(获取许可成功，线程继续执行)，如果房间已经满了，则必须在外面等待直到有人出来(线程阻塞，当有其它获取许可的线程执行完毕释放许可后，当前线程才可继续执行)。

 **Semaphore构造方法如下：**

 
     方法名                                         | 解释                                     
     ------------------------------------------- | --------------------------------------- 
     public Semaphore(int permits)               | permits为许可数量(相当于房间中能容纳的人)              
     public Semaphore(int permits, boolean fair) | 与上述方法类似，但这里多了一个参数：是否公平，即等待时间最长的线程优先获得许可

 **获取许可和释放许可的方法：**

 
     方法名                                                                                      | 作用                                          
     ---------------------------------------------------------------------------------------- | -------------------------------------------- 
     void acquire() throws InterruptedException                                               | 获取单个许可，若没有获取到则当前线程阻塞                        
     void acquire(int permits) throws InterruptedException                                    | 获取permits个许可，若没有获取到则当前线程阻塞                  
     void release()                                                                           | 释放一个许可                                      
     void release(int permits)                                                                | 释放permits个许可                                
     boolean tryAcquire()                                                                     | 尝试获取许可，若成功则立即返回true，否则返回false               
     boolean tryAcquire(long timeout, TimeUnit unit) throws InterruptedException              | 尝试获取许可，若在指定时间内成功获得，则返回true，否则返回false        
     boolean tryAcquire(int permits)                                                          | 尝试获取permits个许可，若成功则返回true，否则返回false         
     boolean tryAcquire(int permits, long timeout, TimeUnit unit) throws InterruptedException | 尝试获取permits个许可，若在指定时间内成功获得，则返回true，否则返回false
     int availablePermits()                                                                   | 获取当前许可数目                                    

 **下面为Semaphore简单使用示范**

 
```
public class TestTask{
    private final ExecutorService exec;
    private final Semaphore semaphore;

    public TestTask(int nThread){
        exec = Executors.newFixedThreadPool(nThread);
        semaphore = new Semaphore(nThread);
    }

    private final class Man implements Runnable{
        private int id;
        public Man(int id) {this.id = id;}

        @Override
        public void run() {
            try{
                semaphore.acquire(); //获取许可
                System.out.println("ID：" + id + "在此房间,剩余许可：" + semaphore.availablePermits());
            }
            catch (InterruptedException e){
            }finally{
                semaphore.release(); //释放许可
                System.out.println("ID：" + id + "走出房间,剩余许可：" + semaphore.availablePermits());
            }
        }
    }

    public void init() throws InterruptedException, ExecutionException{
        for(int i = 1; i <= 8; i++) //一共8个人想进入房间
            exec.submit(new Man(i), true);
        exec.shutdown();
    }

    public static void main(String[] args) throws InterruptedException, ExecutionException{
        TestTask task = new TestTask(4); //设置线程数，相当于房间最大能容纳4个人
        task.init();
    }
}
```
 输出：

 
```
ID：3在此房间,剩余许可：1
ID：2在此房间,剩余许可：1
ID：1在此房间,剩余许可：1
ID：4在此房间,剩余许可：0
ID：1走出房间,剩余许可：3
ID：2走出房间,剩余许可：2
ID：3走出房间,剩余许可：1
ID：5在此房间,剩余许可：3
ID：7在此房间,剩余许可：1
ID：4走出房间,剩余许可：4
ID：7走出房间,剩余许可：3
ID：5走出房间,剩余许可：2
ID：6在此房间,剩余许可：2
ID：8在此房间,剩余许可：2
ID：6走出房间,剩余许可：3
ID：8走出房间,剩余许可：4
```
 了解完基本使用方法后，如果你不了解Java并发框架AbstractQueuedSynchronizer（AQS）的基本原理，可以参考我的博客来熟悉AQS：   
 [https://blog.csdn.net/abc123lzf/article/details/82532036](https://blog.csdn.net/abc123lzf/article/details/82532036)

 如果你彻底理解了AQS，那么Semaphore对于你来说肯定是很简单的。

 
### 二、原理分析

 Semaphore基本结构和ReentrantLock非常类似，都有一个内部类Sync继承了AQS，其子类NonfairSync和FairSync分别实现非公平竞争模式和公平竞争模式。

 
```
public class Semaphore implements java.io.Serializable {
    private final Sync sync;

    abstract static class Sync extends AbstractQueuedSynchronizer {
        //...
    }

    static final class NonfairSync extends Sync {
        //...
    }

    static final class FairSync extends Sync {
        //...
    }
}
```
 Semaphore类只有一个实例变量sync，这个类的所有方法都是基于这个Sync的同步器实现的。

 对于Semaphore的内部类Sync来说，**AQS的state变量代表剩余的许可数量**。

 
##### 1、acquire方法

 acquire方法有两个重载的方法：acquire()和acquire(int)，前者会尝试获取1个许可(资源)，后者会尝试获取指定的许可(资源)数量（但不可小于0）   
 我们只分析acquire()：

 
```
public void acquire() throws InterruptedException {
    sync.acquireSharedInterruptibly(1);
}
```
 acquire方法它实际上调用了Sync类的acquireSharedInterruptibly方法，acquireSharedInterruptibly方法的实现在AQS中。

 
```
public final void acquireSharedInterruptibly(int arg) throws InterruptedException {
    //如果当前线程被打断则抛出异常
    if (Thread.interrupted())
        throw new InterruptedException();
    if (tryAcquireShared(arg) < 0)
        doAcquireSharedInterruptibly(arg);
}
```
 该方法会尝试调用tryAcquireShared方法获取资源，若没有获取到（返回值小于0），则加入到AQS的线程等待队列中等待。   
 tryAcquireShared方法并没有在Sync类中重写，而是实现在子类NonfairSync和FairSync中。

 NonfairSync的实现：

 
```
//acquire参数为申请的许可数量
protected int tryAcquireShared(int acquires) {
    return nonfairTryAcquireShared(acquires);
}

//该方法实现在Sync类中
final int nonfairTryAcquireShared(int acquires) {
    //通过循环反复尝试(自旋)
    for (;;) {
        //获取AQS的state变量
        int available = getState();
        int remaining = available - acquires;
        //如果remaining小于0（获取资源失败）
        //或成功通过CAS将其由旧的state改为remaining（获取资源成功，返回值大于0）
        if (remaining < 0 || compareAndSetState(available, remaining))
            return remaining;
    }
}
```
 Sync重写了AQS的tryReleaseShared方法，即释放资源的方法，获取资源的方法tryAcquireShared由子类实现。   
 nonfairTryAcquireShared非公平实现方式和ReentrantLock类似，通过CAS操作抢占式将available设为remaining，而不管AQS的等待队列中是否有其它线程正在等待资源释放。

 FairSync的实现：

 
```
protected int tryAcquireShared(int acquires) {
    for (;;) {
        //如果AQS的等待队列中有线程正在等待，则返回-1，资源(许可)获取失败
        if (hasQueuedPredecessors())
            return -1;
        //下面和NonfairSync的实现类似
        int available = getState();
        int remaining = available - acquires;
        if (remaining < 0 || compareAndSetState(available, remaining))
            return remaining;
    }
}
```
 FairSync之所以能实现公平，是因为该方法会首先判断AQS的等待队列中是否有线程正在等待，如果没有才会尝试去自己获得资源，若有线程等待，则会将当前线程入队等待。

 
##### 2、release方法

 release同样也有2个重载的方法：release()和release(int)，前者会尝试释放1个许可(资源)，后者会尝试释放指定的许可(资源)数量（但不可小于0）

 
```
public void release() {
    sync.releaseShared(1);
}
```
 releaseShared实现在AQS中：

 
```
public final boolean releaseShared(int arg) {
    //调用子类的tryReleaseShared方法尝试获取许可，大于0代表成功
    if (tryReleaseShared(arg)) {
        //通知等待队列的线程
        doReleaseShared();
        return true;
    }
    return false;
}
```
 tryReleaseShared方法实现在Sync类中，Nonfair和FairSync都是调用该方法来释放资源：

 
```
protected final boolean tryReleaseShared(int releases) {
    for (;;) {
        //获取AQS的state
        int current = getState();
        int next = current + releases;
        //如果数溢出则抛出错误
        if (next < current)
            throw new Error("Maximum permit count exceeded");
        //通过CAS更新AQS的state变量，更新失败则重新尝试
        if (compareAndSetState(current, next))
            return true;
    }
}
```
 tryReleaseShared方法的主要任务就是将AQS原先的state变量加上releases，表示许可(资源)的释放。

   
  