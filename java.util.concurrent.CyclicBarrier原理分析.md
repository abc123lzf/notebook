---
title: java.util.concurrent.CyclicBarrier原理分析
date: 2018-09-13 14:08:38
tags: CSDN迁移
---
 版权声明：尊重原创，转载请注明出处 [ https://blog.csdn.net/abc123lzf/article/details/82658201]( https://blog.csdn.net/abc123lzf/article/details/82658201)   
  ### 一、引言

 CyclicBarrier是Java并发工具包中的一个同步工具类，它允许一组线程互相等待，直到等待的线程达到指定数量后，所有等待的线程才会继续执行。我们可以称它为栅栏。

 CyclicBarrier的构造方法有：

 
     构造方法                                                      | 解释                                     
     --------------------------------------------------------- | --------------------------------------- 
     public CyclicBarrier(int parties)                         | parties为等待线程的数量，等待的线程达到这个数后才会继续执行      
     public CyclicBarrier(int parties, Runnable barrierAction) | 等待的线程突破屏障后，会让其中一个线程首先执行barrierAction的动作

 public 方法：

 
     方法                                            | 解释                                 
     --------------------------------------------- | ----------------------------------- 
     public int getParties()                       | 获取该CyclicBarrier的需要打破的等待线程数量(返回值恒定)
     public int await()                            | 使当前线程等待直到栅栏被打破                     
     public int await(long timeout, TimeUnit unit) | 使当前线程等待到栅栏被打破或者超时                  
     public boolean isBroken()                     | 判断栅栏是否被打破                          
     public void reset()                           | 重置栅栏                               
     public int getNumberWaiting()                 | 获取正在等待的线程数量                        

 **比如下面这个例子：**

 
```
public class Test {

    static final CyclicBarrier cb = new CyclicBarrier(3, ()->{
        System.out.println(Thread.currentThread().getName() + " before run");
    });

    static class Mission implements Runnable {
        @Override
        public void run() {
            try {
                Thread current = Thread.currentThread();
                System.out.println(current.getName() + " wait");
                cb.await();
                System.out.println(current.getName() + " run");
            } catch (InterruptedException | BrokenBarrierException e) {
            }
        }
    }

    public static void main(String[] args) {
        Thread t0 = new Thread(new Mission());
        Thread t1 = new Thread(new Mission());
        Thread t2 = new Thread(new Mission());
        t0.start();
        t1.start();
        t2.start();
    }
}
```
 上述CyclicBarrier初始化时指定的线程等待数为3，并传入了一个匿名Runnable实现类以实现释放线程前执行的任务，所以，当有3个线程调用了CyclicBarrier实例的await方法后，才会释放所有的线程。   
 输出：

 
```
Thread-1 wait
Thread-2 wait
Thread-0 wait
Thread-2 before run
Thread-2 run
Thread-1 run
Thread-0 run
```
 每次运行结果都不确定，但是可以肯定的是：执行顺序必然是wait -> before run -> run

 CyclicBarrier的内部原理相对来说比较简单。

 
--------
 
### 二、原理分析

 CyclicBarrier所有实例变量：

 
```
public class CyclicBarrier {
    //保存一个栅栏是否被打破的对象，打破后CyclicBarrier会重新new一个该对象给generation变量
    private static class Generation {
        //栅栏是否被打破(所有等待的线程被唤醒)
        boolean broken = false;
    }
    //私有锁，保证线程安全
    private final ReentrantLock lock = new ReentrantLock();
    //Condition对象，用于线程的等待和唤醒
    private final Condition trip = lock.newCondition();
    //线程等待数量
    private final int parties;
    //构造时传入的Runnable，当线程唤醒后让最后一个await的线程执行该方法
    private final Runnable barrierCommand;

    private Generation generation = new Generation();
}
```
 
##### 1、await方法

 await有两个重载的方法：int await()和int await(long timeout, TimeUnit unit)，前者线程会一直处于阻塞状态，直到栅栏被打破或线程被interrupt，后者可以指定一个等待时间，超时后线程会继续执行。

 
```
public int await() throws InterruptedException, BrokenBarrierException {
    try {
        return dowait(false, 0L);
    } catch (TimeoutException toe) {
        throw new Error(toe); //不会发生
    }
}

public int await(long timeout, TimeUnit unit) throws InterruptedException,
               BrokenBarrierException, TimeoutException {
        return dowait(true, unit.toNanos(timeout));
}
```
 两个方法实际上都调用了CyclicBarrier的dowait方法：

 
```
private int dowait(boolean timed, long nanos) throws InterruptedException
        , BrokenBarrierException, TimeoutException {
    final ReentrantLock lock = this.lock;
    //加锁,保证该方法线程安全
    lock.lock();
    try {
        final Generation g = generation;
        //如果栅栏被打破，抛出异常
        if (g.broken)
            throw new BrokenBarrierException();
        //如果当前线程被中断，则将栅栏设为打开状态并激活所有等待线程
        if (Thread.interrupted()) {
            breakBarrier();
            throw new InterruptedException();
        }
        //等待数减1
        int index = --count;
        //若count为0，则尝试激活所有线程
        if (index == 0) {
            boolean ranAction = false;
            try {
                final Runnable command = barrierCommand;
                //执行构造方法传入的Runnable，如果不为null的话
                if (command != null)
                    command.run();
                ranAction = true;
                //将CyclicBarrier设为初始状态，并重置Generation对象
                nextGeneration();
                return 0;
            } finally {
                //如果runAction为false（执行Runnable时有异常抛出），那么打破栅栏，激活所有等待的线程
                if (!ranAction)
                    breakBarrier();
            }
        }

        //自旋操作
        for (;;) {
            try {
                //如果没有设置等待超时时间，那么调用Condition的await等待
                if (!timed)
                    trip.await();
                //如果设置了，则调用Condition的awaitNanos方法
                else if (nanos > 0L)
                    nanos = trip.awaitNanos(nanos);
            //如果等待过程中线程被打断
            } catch (InterruptedException ie) {
                //如果Generation没有重置并且没有打破栅栏
                if (g == generation && !g.broken) {
                    //打破栅栏并抛出InterruptedException
                    breakBarrier();
                    throw ie;
                } else {
                    Thread.currentThread().interrupt();
                }
            }
            //如果栅栏被打破则抛出异常
            if (g.broken)
                throw new BrokenBarrierException();
            //如果栅栏已经重置则直接返回
            if (g != generation)
                return index;
            //如果nanos参数小于等于0，意味着等待超时，那么打破栅栏并抛出异常
            if (timed && nanos <= 0L) {
                breakBarrier();
                throw new TimeoutException();
            }
        }
    } finally {
        lock.unlock();
    }
}
```
 dowait方法的流程图（建议放大看）：   
 ![这里写图片描述](https://img-blog.csdn.net/20180913155053967?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FiYzEyM2x6Zg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

 
##### 2、reset方法

 reset方法可以用来重置CyclicBarrier，如果此时CyclicBarrier有等待的线程，那么调用该方法会激活所有线程继续执行，然后将CyclicBarrier重新设定为初始状态。

 
```
public void reset() {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        breakBarrier();
        nextGeneration(); 
    } finally {
        lock.unlock();
    }
}
```
 该方法会依次调用breakBarrier和nextGeneration方法。前者会激活所有等待的线程，后者会将Generation对象更新。   
 方法实现比较简单，不再赘述。

 
```
private void breakBarrier() {
    generation.broken = true;
    count = parties;
    trip.signalAll();
}

private void nextGeneration() {
    trip.signalAll();
    count = parties;
    generation = new Generation();
}
```
   
  