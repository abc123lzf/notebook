---
title: Java并发编程 闭锁：FutureTask、CountDownLatch、Semaphore常见使用方法
date: 2018-05-14 14:00:00
tags: CSDN迁移
---
 版权声明：尊重原创，转载请注明出处 [ https://blog.csdn.net/abc123lzf/article/details/80307976]( https://blog.csdn.net/abc123lzf/article/details/80307976)   
  ### 一、什么是闭锁

 闭锁是一种同步工具，可以延迟线程的进度直到终止状态。可以把它理解为一扇门，当闭锁到达结束状态之前，这扇门一直是关闭的，没有任何线程可以通过。当闭锁到达结束状态时，这扇门会打开并允许所有线程通过，并且闭锁打开后不可再改变状态。   
 闭锁可以确保某些任务直到其他任务完成后才继续往下执行。

 这篇博客只讲述这些闭锁常见用法，如果想深入了解其中原理的话请参考其它博客。

 
### 二、FutureTask

 FutureTask相信大家也不陌生了，它是一种闭锁。FutureTask表示的任务是通过一个类实现Callable接口实现的，相当于一种有结果的Runnable，并且可以抛出异常。FutureTask可以处于下面三种状态：等待运行、正在运行、运行完成。执行完成状态表示该任务所有可能结束的方式：包括正常结束、抛出异常、任务中途取消。FutureTask一般与多线程管理工具Executor配合使用。   
 FutureTask实例的get行为取决于任务的状态。如果任务已完成，则返回结果；如果该任务正在执行，则get方法会一直阻塞直到完成、抛出异常。   
 FutureTask继承关系：   
 ![这里写图片描述](http://incdn1.b0.upaiyun.com/2017/06/728ad60436305482476012b9ac99c699.png)   
 图片转自[http://www.importnew.com/25286.html](http://www.importnew.com/25286.html) 如果想深入了解FutureTask请点击这篇博客，这里只简要讲述用法   
 FutureTask的构造方法:

 
     方法名                                            | 解释                                                   
     ---------------------------------------------- | ----------------------------------------------------- 
     public FutureTask(Callable&lt;V&gt; callable)  | 将Callable任务封装进FutureTask                             
     public FutureTask(Runnable runnable, V result) | 将Runnable任务封装进FutureTask，如果执行成功则会返回传入的result，否则返回null

 
### 三、CountDownLatch

 CountDownLatch是一种比较灵活的闭锁工具，它可以使多个线程等待一组事件发生后才继续执行。该闭锁状态由一个计数器控制，创建该闭锁实例时必须初始化为一个正数，一般用来表示需要等待的事件数量，可以通过调用countDown方法来递减计数器，await方法等待计数器直到为0时才继续执行，否则该线程会一直阻塞。   
 **CountDownLatch构造函数：**

 
```
public CountDownLatch(int count)
```
 count就是该闭锁需要等待的线程数量，这个值一旦设置不能更改   
 **CountDownLatch方法：**

 
     方法名                                        | 解释                                               
     ------------------------------------------ | ------------------------------------------------- 
     void countDown()                           | 计数器减1                                            
     long getCount()                            | 返回当前计数值                                          
     void await()                               | 当前线程阻塞，直到该计数器为0后才能继续执行                           
     boolean await(long timeout, TimeUnit unit) | 与上述await()类似，只不过可以设置等待时间，如果在规定时间内计数器的值依然大于0，则继续执行

 **下面是CountDownLatch和FutureTask使用示例**：

 
```
public class TestTask{
    private final ExecutorService exec; //线程池
    private final BlockingQueue<Future<Integer>> queue = new LinkedBlockingQueue<>();
    private final CountDownLatch startLock = new CountDownLatch(1); //启动门，当所有线程就绪时调用countDown
    private final CountDownLatch endLock; //结束门
    private final int nThread; //线程数量

    private volatile int count = 0;

    public TestTask(int nThread){
        this.nThread = nThread; //初始化线程数量
        exec = Executors.newFixedThreadPool(nThread); //创建线程池，线程池共有nThread个线程
        endLock = new CountDownLatch(nThread);  //设置结束门计数器，当一个线程结束时调用countDown
    }

    private class Task implements Callable<Integer>{
        @Override
        public Integer call() throws Exception {
            startLock.await(); //线程启动后调用await，当前线程阻塞，只有启动门计数器为0时当前线程才会往下执行
            try{
                for(int i = 0; i < 100; i++)
                    addCount();
                return count;
            }finally{
                endLock.countDown(); //线程执行完毕，结束门计数器减1
            }
        }
    }

    private synchronized void addCount() { count++; }

    public void init() throws InterruptedException, ExecutionException{
        for(int i = 0; i < nThread; i++){
            Future<Integer> future = exec.submit(new Task()); //将任务提交到线程池
            queue.add(future); //将Future实例添加至队列
        }
        startLock.countDown(); //所有任务添加完毕，启动门计数器减1，这时计数器为0，所有添加的任务开始执行
        endLock.await(); //主线程阻塞，直到所有线程执行完成
        for(Future<Integer> future : queue)
            System.out.println(future.get()); //获取Callable返回值
        System.out.println(count); //这里应当输出800
        exec.shutdown(); //关闭线程池
    }

    public static void main(String[] args) throws InterruptedException, ExecutionException{
        TestTask task = new TestTask(8);
        task.init();
    }
}
```
 
### 四、Semaphore

 Semaphore又称信号量，用来控制同时访问访问某个特定资源的操作数量。   
 Semaphore管理着一组许可(permit)，许可的数量可通过构造函数来指定。在执行某个操作时必须先获得许可，并在使用完毕后释放(release)许可。如果没有许可，那么尝试获取许可的线程会进入阻塞状态直到有许可可以使用。

 可以想象成一个房间，房间里只能容纳3个人(每个人相当于一个线程)。现在外面的人想要进来(尝试获取许可)，如果房间没有满，则可以直接进入(获取许可成功，线程继续执行)，如果房间已经满了，则必须在外面等待直到有人出来(线程阻塞，当有其它获取许可的线程执行完毕释放许可后，当前线程才可继续执行)。

 **Semaphore构造方法如下：**

 
     方法名                                         | 解释                                     
     ------------------------------------------- | --------------------------------------- 
     public Semaphore(int permits)               | permits为许可数量(相当于房间中能容纳的人)              
     public Semaphore(int permits, boolean fair) | 与上述方法类似，但这里多了一个参数：是否公平，即等待时间最长的线程优先获得许可

 **获取许可和释放许可的方法：**

 
     方法名                                                                                      | 解释                                          
     ---------------------------------------------------------------------------------------- | -------------------------------------------- 
     void acquire() throws InterruptedException                                               | 获取单个许可                                      
     void acquire(int permits) throws InterruptedException                                    | 获取permits个许可                                
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
   
  