---
title: Java线程池之ThreadPoolExecutor源码分析
date: 2018-09-19 14:59:56
tags: CSDN迁移
---
 版权声明：尊重原创，转载请注明出处 [ https://blog.csdn.net/abc123lzf/article/details/82733490]( https://blog.csdn.net/abc123lzf/article/details/82733490)   
  ### []()一、引言

 Java并发工具包自带了很多常用的线程池，程序可以将定义的 `Runnable` 、 `Callable` 任务提交到线程池当中运行，由线程池负责异步执行其中的任务。

 Java线程池框架结构图：  
 ![在这里插入图片描述](https://img-blog.csdn.net/20180923175657642?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FiYzEyM2x6Zg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)  
 其中， `Executors` 是一个线程池静态工厂类，可以调用其中的静态方法获取一些常用的线程池实现类。 `Executors` 的内部类 `DelegatedExecutorService` 采用了装饰者设计模式，其内部持有一个 `ExecutorService` 的实现类。

 `ThreadPoolExecutor` 是Java并发工具包中非常常用的一个线程池，在我们通过调用 `Executors` 类的 `newCachedThreadPool` 、 `newSingleThreadExecutor` 、 `newFixedThreadPool` 返回的线程池其实都是 `ThreadPoolExecutor` 。

 `ThreadPoolExecutor` 的public方法及其作用：

 
     方法签名                                                       | 作用                                     
     ---------------------------------------------------------- | --------------------------------------- 
     void execute(Runnable)                                     | 提交一个Runnable任务到线程池任务队列，等待执行            
     void shutdown()                                            | 线程池不再接受新的任务，并将已有的任务执行完毕后关闭线程池          
     List&lt;Runnable&gt; shutdownNow()                         | 立刻关闭线程池，正在执行的任务会被interrupt，返回尚未被执行的任务集合
     boolean isShutdown()                                       | 返回线程池是否已经被关闭                           
     boolean isTerminating()                                    | 返回线程池是否正在被关闭                           
     boolean isTerminated()                                     | 返回线程池是否已经彻底被关闭                         
     boolean awaitTermination(long, TimeUnit)                   | 使当前线程阻塞到线程池完全被关闭或者超时                   
     void setThreadFactory(ThreadFactory)                       | 重新设置线程池的线程工厂                           
     ThreadFactory getThreadFactory()                           | 获取线程池的线程工厂                             
     void setRejectedExecutionHandler(RejectedExecutionHandler) | 设置饱和策略处理器，当任务队列满了后提交的任务由这个处理器处理        
     RejectedExecutionHandler getRejectedExecutionHandler()     | 获得该线程池的饱和策略处理器                         
     void setCorePoolSize(int)                                  | 重新设置线程池最低线程(核心线程)数量                    
     int getCorePoolSize()                                      | 获得该线程池的最低线程数量                          
     boolean prestartCoreThread()                               | 预先启动一个核心线程                             
     int prestartAllCoreThreads()                               | 预先启动所有核心线程，返回启动的线程数量                   
     boolean allowsCoreThreadTimeOut()                          | 返回核心线程是否会因为等待超时而被回收                    
     void allowCoreThreadTimeOut(boolean)                       | 指定核心线程会不会因为等待超时而被回收                    
     void setMaximumPoolSize(int)                               | 设置线程池最大的活跃线程数量                         
     int getMaximumPoolSize()                                   | 获得线程池允许的最大线程数量                         
     void setKeepAliveTime(long, TimeUnit)                      | 设置线程被回收前的最长空闲时间                        
     long getKeepAliveTime(TimeUnit)                            | 获得线程被回收前的最长空闲时间                        
     BlockingQueue&lt;Runnable&gt; getQueue()                   | 获得该线程池的任务队列引用                          
     boolean remove(Runnable)                                   | 从任务队列中移除指定的任务，返回是否移除成功                 
     void purge()                                               | 移除任务队列中所有的已经被取消掉的Future任务              
     int getPoolSize()                                          | 获取当前线程池的线程数量                           
     int getActiveCount()                                       | 获取当前线程池正在执行任务的线程数量                     
     int getLargestPoolSize()                                   | 获取线程池曾经达到的最大线程数量                       
     long getTaskCount()                                        | 返回该线程池执行过的任务数量加上任务队列中尚未执行的任务（近似值）      
     long getCompletedTaskCount()                               | 返回该线程池执行过的任务数量（近似值）                    


--------
 
### []()二、源码解析

 下面是ThreadPoolExecutor的类定义和实例变量：

 
```
public class ThreadPoolExecutor extends AbstractExecutorService {
	//线程池运行状态和线程数量
	private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
	//任务队列，存放被提交的任务
	private final BlockingQueue<Runnable> workQueue;
	//私有锁，保证线程池并发安全
	private final ReentrantLock mainLock = new ReentrantLock();
	//保存执行线程的Set集合
	private final HashSet<Worker> workers = new HashSet<Worker>();
	//私有锁对应的Condition,用来实现线程的等待和唤醒
	private final Condition termination = mainLock.newCondition();
	//线程池运行期间达到过的最大线程数量
	private int largestPoolSize;
	//已完成的任务总数
	private long completedTaskCount;
	//线程工厂，负责构造线程实例
	private volatile ThreadFactory threadFactory;
	//饱和策略处理器，当任务队列已满时，提交的任务会由这个处理器处理
	private volatile RejectedExecutionHandler handler;
	//非核心线程最大存活时间(单位:纳秒)，超出这个时间线程对象会被回收
	private volatile long keepAliveTime;
	//是否允许核心线程超时后被回收
	private volatile boolean allowCoreThreadTimeOut;
	//线程池最小线程数量(即核心线程数量)
	private volatile int corePoolSize;
	//线程池最大线程数量
	private volatile int maximumPoolSize;
	//安全管理器(隶属java.security包)
	private final AccessControlContext acc;
}

```
 每个 `ThreadPoolExecutor` 维护了一个原子变量 `ctl` ， `ctl` 中保存了线程池的运行状态和目前持有的线程数量：

 
```
private static final int COUNT_BITS = Integer.SIZE - 3;
private static final int CAPACITY   = (1 << COUNT_BITS) - 1;

private static final int RUNNING    = -1 << COUNT_BITS;
private static final int SHUTDOWN   =  0 << COUNT_BITS;
private static final int STOP       =  1 << COUNT_BITS;
private static final int TIDYING    =  2 << COUNT_BITS;
private static final int TERMINATED =  3 << COUNT_BITS;

private static int runStateOf(int c)     { return c & ~CAPACITY; }
private static int workerCountOf(int c)  { return c & CAPACITY; }
private static int ctlOf(int rs, int wc) { return rs | wc; }

```
 原子变量ctl高3位负责保存运行状态，低29位保存线程的数量，可通过位运算提取其中的数值。  
 从上面的静态常量可以看出，线程池一共有5个状态：  
  `RUNNING` ：当前线程池正在运行  
  `SHUTDOWN` ：当前线程池被挂起（调用 `shutdown` 方法）  
  `STOP` ：当前线程池被停止（调用 `shutdownNow` 方法）  
  `TIDYING` ：所有任务已经被终止，并且线程池拥有的线程数量为0  
  `TERMINATED` ：线程池彻底被终止（ `terminated` 方法已被调用）

 
##### []()1、构造方法

 线程池提供了4个public构造方法，这几个构造方法其实相当于1个：

 
```
public ThreadPoolExecutor(int corePoolSize, int maximumPoolSize, long keepAliveTime,
		TimeUnit unit, BlockingQueue<Runnable> workQueue, ThreadFactory threadFactory,
        RejectedExecutionHandler handler) {
    //检查参数是否合法
	if (corePoolSize < 0 || maximumPoolSize <= 0 || maximumPoolSize < corePoolSize || keepAliveTime < 0)
        throw new IllegalArgumentException();
    if (workQueue == null || threadFactory == null || handler == null)
        throw new NullPointerException();
    //检查有没有指定安全管理器
    this.acc = System.getSecurityManager() == null ? null : AccessController.getContext();
    this.corePoolSize = corePoolSize;
    this.maximumPoolSize = maximumPoolSize;
    this.workQueue = workQueue;
    this.keepAliveTime = unit.toNanos(keepAliveTime);
    this.threadFactory = threadFactory;
    this.handler = handler;
}

```
 这个构造方法一共有7个参数，其含义如下：

 
     参数名             | 参数类型                          | 含义                            
     --------------- | ----------------------------- | ------------------------------ 
     corePoolSize    | int                           | 该线程池的核心线程数量，即最小线程数量           
     maximumPoolSize | int                           | 该线程池所允许的最大线程数量                
     keepAliveTime   | long                          | 非核心线程的最大空闲生存时间，超出这个时间后会被回收    
     unit            | TimeUnit                      | 上述生存时间的时间单位                   
     workQueue       | BlockingQueue&lt;Runnable&gt; | 该线程池存放任务的任务队列                 
     threadFactory   | ThreadFactory                 | 线程工厂，线程池需要线程对象时由它负责创建         
     handler         | RejectedExecutionHandler      | 当任务队列已满无法接受新的任务时，由它负责对这个任务进行处理

构造方法的作用基本上就是给成员变量指定初始值，构造结束后，线程池中的线程数量为0，只有当提交了任务后，线程池才会真正运行起来。  
 **其它构造方法：**

 
```
public ThreadPoolExecutor(int corePoolSize, int maximumPoolSize, long keepAliveTime,
                              TimeUnit unit, BlockingQueue<Runnable> workQueue) {
    this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
             Executors.defaultThreadFactory(), defaultHandler);
}

```
 这个构造方法会指定默认的线程工厂和默认的饱和处理器。

 
##### []()2、线程工厂

 `ThreadPoolExecutor` 线程池通过线程工厂( `ThreadFactory` )来获取线程的实例，以达到异步执行任务的目的。  
 线程工厂需要实现ThreadFactory接口，该接口只定义了一个方法：

 
```
public Thread newThread(Runnable r);

```
 即通过任务对象r返回一个线程实例。

 通过调用 `Executors` 的静态方法 `defaultThreadFactory` ，可以获得一个静态工厂的实现类，这个实现类是 `Executors` 的静态内部类 `DefaultThreadFactory` ：

 
```
static class DefaultThreadFactory implements ThreadFactory {
	//线程池序列号，用于指定线程对象的名称
	private static final AtomicInteger poolNumber = new AtomicInteger(1);
	//生产出来的线程所属的线程组
    private final ThreadGroup group;
    //线程序列号，用于指定线程对象的名称
    private final AtomicInteger threadNumber = new AtomicInteger(1);
    //线程对象名称前缀
    private final String namePrefix;

    DefaultThreadFactory() {
	    //获得安全管理器
        SecurityManager s = System.getSecurityManager();
        //如果安全管理器不为null的话通过安全管理器指定线程组，否则设置为当前线程的线程组
        group = (s != null) ? s.getThreadGroup() : Thread.currentThread().getThreadGroup();
        //设置名称前缀，比如"pool-1-thread-"
        namePrefix = "pool-" + poolNumber.getAndIncrement() + "-thread-";
    }

    public Thread newThread(Runnable r) {
	    //构造一个新的线程对象，指定线程组、Runnable任务、线程名
	    Thread t = new Thread(group, r, namePrefix + threadNumber.getAndIncrement(), 0);
	    //将线程对象设为非后台(守护)线程
        if (t.isDaemon())
            t.setDaemon(false);
        //将线程的优先级设为一般优先级
        if (t.getPriority() != Thread.NORM_PRIORITY)
            t.setPriority(Thread.NORM_PRIORITY);
        return t;
    }
}

```
 默认的线程工厂只是给线程对象指定一个名称和 `Runnable` 任务，并将其设为非守护线程和一般优先级。  
 当我们调用 `Executors` 类的 `newCachedThreadPool` 、 `newFixedThreadPool` 、 `newSingleThreadExecutor` 时，其线程工厂都会默认设置为它。

 
##### []()3、饱和策略

 `ThreadPoolExecutor` 定义了4种 `RejectedExecutionHandler` 实现类，分别是 `AbortPolicy` 、 `CallerRunsPolicy` 、 `DiscardPolicy` 、 `DiscardOldestPolicy` ，这些都是 `ThreadPoolExecutor` 的 `public` 权限的非final静态内部类，代表着4种饱和策略。  
 下面通过代码来介绍这几种自带的饱和策略：

 
```
public static class AbortPolicy implements RejectedExecutionHandler {
	public AbortPolicy() { }
	public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
        throw new RejectedExecutionException("Task " + r.toString() + " rejected from " + e.toString());
    }
}

```
 `AbortPolicy` （中止策略）是 `ThreadPoolExecutor` 的默认饱和策略，该策略将抛出未检查的 `RejectedExecutionException` 异常。调用者可以选择捕获该异常并根据自己的需求编写处理的代码。

 
```
public static class CallerRunsPolicy implements RejectedExecutionHandler {
	public CallerRunsPolicy() { }
	public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
        if (!e.isShutdown()) {
	        //由当前线程执行任务
            r.run();
        }
    }
}

```
 `CallerRunsPolicy` （调用者运行策略）：该策略既不会抛弃任务，也不会抛出异常，而是将任务回退给调用者（比如回退给main线程，由main线程自行处理）。

 
```
public static class DiscardPolicy implements RejectedExecutionHandler {
	public DiscardPolicy() { }
	public void rejectedExecution(Runnable r, ThreadPoolExecutor e) { /*啥也不做*/ }
}

```
 `DiscardPolicy` （抛弃策略）：当工作队列已满并无法添加，抛弃策略会扔掉该任务，什么也不做。

 
```
public static class DiscardOldestPolicy implements RejectedExecutionHandler {
	public DiscardOldestPolicy() { }
	public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
        if (!e.isShutdown()) {
	        //从任务队列取出队首元素
            e.getQueue().poll();
            //重新将任务添加
            e.execute(r);
        }
    }
}

```
 `DiscardOldestPolicy` （抛弃最旧任务策略）：该策略会抛弃任务队列队首的任务并尝试重新提交新的任务，如果任务队列是优先队列，那么将抛弃优先级最高的任务（所以不要和优先队列同时使用）。

 除此之外，用户还可以自定义饱和处理器，只需要在构造阶段指定饱和处理器或者通过 `setRejectedExecutionHandle` 方法指定。

 
##### []()4、任务的提交

 用户可以通过 `executor` 方法向线程池提交一个 `Runnable` 任务

 
```
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();
    //获取线程池的状态和线程数量
    int c = ctl.get();
    //如果目前的线程数量小于核心线程数量
    if (workerCountOf(c) < corePoolSize) {
	    //生产一个线程，并执行任务
        if (addWorker(command, true))
            return;
        c = ctl.get();
    }
    //如果线程池正在运行并且成功将任务添加至队列
    if (isRunning(c) && workQueue.offer(command)) {
	    //再次获取ctl
        int recheck = ctl.get();
        //如果线程池此时停止，那么从任务队列移除这个任务，并转交给饱和处理器处理
        if (!isRunning(recheck) && remove(command))
            reject(command);
        //如果线程池没有停止，那么如果判断线程池线程数量为0的话，就向线程池添加一个线程对象
        else if (workerCountOf(recheck) == 0)
            addWorker(null, false);
    }
    //添加线程并执行任务，如果没有成功，那么转交给饱和处理器处理
    else if (!addWorker(command, false))
        reject(command);
}

```
 `execute` 方法执行步骤如下：  
 1、如果线程池目前拥有的线程数量小于核心线程数量，那么调用 `addWorker` 方法向线程池添加一个线程并执行这个任务，操作成功后 `execute` 方法结束。  
 2、否则，就将这个任务对象添加到任务队列，如果线程池停止运行或者任务已满，那么调用 `reject` 方法将这个任务对象转交给饱和处理器处理。另外如果线程池没有停止并且线程数量为0的话， `execute` 方法会向线程池添加一个线程。

 `addWorker` 方法的源码：

 
```
private boolean addWorker(Runnable firstTask, boolean core) {
	retry:
	//线程进入自旋
    for (;;) {
	    int c = ctl.get(); 
	    //获取运行状态
        int rs = runStateOf(c);
        //如果线程池停止，那么返回false
        if (rs >= SHUTDOWN && ! (rs == SHUTDOWN && firstTask == null && !workQueue.isEmpty()))
            return false;
        for (;;) {
	        //获取线程池的线程数量
            int wc = workerCountOf(c);
            //如果线程数量超出线程池允许的最大线程数量（或者当core为true时超出核心线程数量），则返回false
            if (wc >= CAPACITY || wc >= (core ? corePoolSize : maximumPoolSize))
                return false;
            //增加线程数量，如果增加成功，跳出retry块
            if (compareAndIncrementWorkerCount(c))
                break retry;
            c = ctl.get();
            //如果线程池状态发生改变，则从retry块开始
            if (runStateOf(c) != rs)
                continue retry;
        }
    }
	//线程是否成功启动
    boolean workerStarted = false;
    //是否成功添加到线程集合中
    boolean workerAdded = false;
    Worker w = null;
    try {
	    //创建一个Worker(相当于一个线程)
        w = new Worker(firstTask);
        final Thread t = w.thread;
        if (t != null) {
	        //获取私有锁并加锁，保证线程安全
            final ReentrantLock mainLock = this.mainLock;
            mainLock.lock();
            try {
	            //获取线程池状态
                int rs = runStateOf(ctl.get());
                //如果线程池正在运行
                if (rs < SHUTDOWN || (rs == SHUTDOWN && firstTask == null)) {
	                //如果线程已经在运行，抛出IllegalThreadStateException
                    if (t.isAlive())
                        throw new IllegalThreadStateException();
                    //向保存Worker的HashSet添加这个Worker
                    workers.add(w);
                    int s = workers.size();
                    //如果当前线程数量超出曾经达到的最大线程数量的话，更新这个值
                    if (s > largestPoolSize)
                        largestPoolSize = s;
                    workerAdded = true;
                }
            } finally {
                mainLock.unlock();
            }
            //如果成功地将Worker添加到集合中，那么启动这个线程
            if (workerAdded) {
                t.start();
                workerStarted = true;
            }
        }
    } finally {
        if (! workerStarted)
            addWorkerFailed(w);
    }
    return workerStarted;
}

```
 `reject` 方法比较简单，直接获取 `ThreadPoolExecutor` 指定的饱和处理器并执行

 
```
final void reject(Runnable command) {
	//调用饱和处理器的rejectExecution处理这个任务
	handler.rejectedExecution(command, this);
}

```
 execute方法的过程可以用流程图表示如下：  
 ![在这里插入图片描述](https://img-blog.csdn.net/20180918221013254?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FiYzEyM2x6Zg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

 
##### []()5、任务的执行

 每个提交的任务都由线程池中的线程负责执行。  
  `ThreadPoolExecutor` 定义了一个非静态内部类： `Worker` ，每个 `Worker` 实例表示一个线程  
 线程池通过将线程封装为一个 `Worker` 对象，保存在线程池的 `HashSet` 类型的成员变量 `workers` 中：

 
```
private final class Worker extends AbstractQueuedSynchronizer implements Runnable {
	//工作线程
	final Thread thread;
	//第一个任务对象
	Runnable firstTask;
	//这个线程已经完成的任务数量
	volatile long completedTasks;
	
	Worker(Runnable firstTask) {
		//将AQS的state状态设为-1，表示必须要首先调用release方法
        setState(-1);
        this.firstTask = firstTask;
        //通过线程池的线程工厂获取线程对象
        this.thread = getThreadFactory().newThread(this);
    }
    
    public void run() {
	    //调用外部类的runWorker方法执行任务
        runWorker(this);
    }
    //省略其它方法
}

```
 `Worker` 持有三个实例变量：执行任务的线程对象、第一个执行的任务对象（仅用于这个线程执行的第一个任务）和这个 `Worker` 已经执行完成的任务数量。  
  `Worker` 继承了 `AQS` 类，是为了能够判断 `Worker` 是否在执行任务，并能够使 `Worker` 在执行完任务后继续执行下一个任务。 `Worker` 重写了AQS的 `tryAcquire` 、 `tryRelease` 、 `isHeldExclusively` 方法，可以通过 `state` 变量的值来判定 `Worker` 的状态。

 对于AQS的实现类 `Worker` 来说，其state变量含义如下：  
  `state` 为-1时，该 `Worker` 处于初始化状态，即还没有开始执行第一个任务。  
  `state` 为0时，该 `Worker` 正在从任务队列获取任务，或者正准备执行第一个任务。  
  `state` 为1时，该 `Worker` 正在执行任务。

 
```
//如果AQS的state状态不为0，则持有锁
protected boolean isHeldExclusively() {
    return getState() != 0;
}
//尝试获取锁
protected boolean tryAcquire(int unused) {
	//通过CAS将state由0设为1，成功后表示成功获得锁
    if (compareAndSetState(0, 1)) {
	    //将锁的持有线程设置为当前线程
        setExclusiveOwnerThread(Thread.currentThread());
        return true;
    }
    return false;
}
//释放锁，这里只会返回true
protected boolean tryRelease(int unused) {
	//清除持有锁的线程并将state设为0
	setExclusiveOwnerThread(null);
    setState(0);
    return true;
}

//加锁
public void lock()        { acquire(1); }
public boolean tryLock()  { return tryAcquire(1); }
//解锁
public void unlock()      { release(1); }
public boolean isLocked() { return isHeldExclusively(); }

```
 每个线程都会执行外部类 `ThreadPoolExecutor` 的 `runWorker` 方法并将自身作为参数传递进去：

 
```
final void runWorker(Worker w) {
	//获得当前线程
	Thread wt = Thread.currentThread();
	//获取需要执行的任务对象
	Runnable task = w.firstTask;
    w.firstTask = null;
    //解锁，将state由-1设为0
    w.unlock();
    boolean completedAbruptly = true;
    try {
	    //如果任务为null，则获取新的任务继续执行，否则跳出循环，线程结束
        while (task != null || (task = getTask()) != null) {
	        //加锁
            w.lock();
            //线程池如果处于STOP(这里检查两次是为了防止shutdownNow方法调用interrupt中断)
            if ((runStateAtLeast(ctl.get(), STOP) || 
                    (Thread.interrupted() && runStateAtLeast(ctl.get(), STOP)))  && 
                     !wt.isInterrupted())
                //中断当前线程
                wt.interrupt();
            try {
	            //在执行任务前的操作(在ThreadPoolExecutor类实现为空，由子类实现)
                beforeExecute(wt, task);
                //任务执行期间未捕获的异常
                Throwable thrown = null;
                try {
                    task.run();
                //如果有异常则抛出
                } catch (RuntimeException x) {
                    thrown = x; throw x;
                } catch (Error x) {
                    thrown = x; throw x;
                } catch (Throwable x) {
                    thrown = x; throw new Error(x);
                } finally {
                    //执行任务完成之后的操作(在ThreadPoolExecutor类实现为空，由子类实现)
                    afterExecute(task, thrown);
                }
            } finally {
	            //任务执行完成，task设为null
                task = null;
                //Worker的任务计数器加1
                w.completedTasks++;
                //解锁
                w.unlock();
            }
        }
        completedAbruptly = false;
    } finally {
        processWorkerExit(w, completedAbruptly);
    }
}

```
 每个 `Worker` 中的线程都会通过一个 `while` 循环不断获取任务队列中的任务并执行。  
 在 `while` 循环中， `Worker` 通过调用 `getTask` 方法来获取新的任务：

 
```
private Runnable getTask() {
	boolean timedOut = false;
    for (;;) {
        int c = ctl.get();
        //获取线程池状态
        int rs = runStateOf(c);
		//如果线程池已被关闭并且任务队列为空
        if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
	        //更新线程池执行过的任务数量计数器并返回null
            decrementWorkerCount();
            return null;
        }
		//获取线程池的线程数量
        int wc = workerCountOf(c);
        //当前线程是否会因为超时而被回收
        boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;
		//如果线程数量超出最大限制或者等待时间超时并且等待队列为空或者线程数量大于1
        if ((wc > maximumPoolSize || (timed && timedOut)) && (wc > 1 || workQueue.isEmpty())) {
	        //将线程计数器的值减1，如果设置成功返回null
            if (compareAndDecrementWorkerCount(c))
                return null;
           //否则重新开始循环
            continue;
        }
		//如果当前线程会因为等待超时而被回收，那么调用poll方法获取任务并指定最长等待时间，否则通过调用take方法获取
        try {
            Runnable r = timed ? workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) : workQueue.take();
            //如果获取到了任务，返回任务对象
            if (r != null)
                return r;
            //将timeOut设为true，重新开始循环
            timedOut = true;
        //如果在等待获取元素时被interrupt
        } catch (InterruptedException retry) {
            timedOut = false;
        }
    } 
}

```
 当 `Worker` 需要停止运行时，可以调用 `Worker` 的 `interruptIfStarted` 方法将其停止：

 
```
void interruptIfStarted() {
	Thread t;
	//如果线程正在运行，那么中断这个线程
	if (getState() >= 0 && (t = thread) != null && !t.isInterrupted()) {
	    try {
		    t.interrupt();
	    } catch (SecurityException ignore) {
		    //如果没有中断这个线程的权限，那么忽略
	    }
    }
}

```
 这个方法会由 `shutdownNow` 方法调用。

 
##### []()6、线程池的关闭

 可以调用 `shutdown` 或者 `shutdownNow` 方法来关闭线程池，这两者之间的区别在于 `shutdown` 方法采用温和的策略关闭线程池：不再接受新的任务，对于任务队列中尚未完成的任务线程池会将它们执行完成之后再关闭。而 `shutdownNow` 则不一样， `shutdownNow` 将所有的工作线程全部 `interrupt` ，并返回任务队列中尚未执行的任务集合。

 我们先从 `shutdown` 方法开始分析：

 
```
public void shutdown() {
	final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
	    //检查调用者有没有关闭线程的权限
        checkShutdownAccess();
        //将线程池状态设置为SHUTDOWN
        advanceRunState(SHUTDOWN);
        //将所有工作线程interrupt
        interruptIdleWorkers();
        //实现为空，由子类ScheduledThreadPool实现
        onShutdown();
    } finally {
        mainLock.unlock();
    }
    //彻底关闭线程池
    tryTerminate();
}

```
 `shutdown` 方法会首先调用 `checkShutdownAccess` 检查调用者是否有关闭线程池的权限，否则抛出 `AccessControlException` ：

 
```
private void checkShutdownAccess() {
	SecurityManager security = System.getSecurityManager();
    if (security != null) {
	    //检查关闭权限
        security.checkPermission(shutdownPerm);
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
	        //检查每个工作线程的权限
            for (Worker w : workers)
                security.checkAccess(w.thread);
        } finally {
            mainLock.unlock();
        }
    }
}

```
 检查完权限后， `shutdown` 方法会通过 `CAS` 将线程池记录状态的变量设为 `SHUTDOWN` 状态，即关闭状态。然后调用 `interruptIdleWorkers` 方法，尝试将所有正在执行任务的线程 `interrupt` 。

 
```
private void interruptIdleWorkers() {
	interruptIdleWorkers(false);
}

private void interruptIdleWorkers(boolean onlyOne) {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
	    //遍历workers集合中所有的Worker对象
        for (Worker w : workers) {
            Thread t = w.thread;
            //尝试每个空闲的工作线程interrupt
            if (!t.isInterrupted() && w.tryLock()) {
                try {
                    t.interrupt(); //尝试中断线程
                } catch (SecurityException ignore) {
                //如果遇到访问权限不足则忽略
                } finally {
                    w.unlock();
                }
            }
            if (onlyOne)
                break;
        }
    } finally {
        mainLock.unlock();
    }
}

```
 `interruptIdleWorkers` 会尝试对线程池中每个空闲的线程，即没有在执行任务的线程进行中断操作。该方法会尝试对 `Worker` 进行 `tryLock` 操作，这个操作会尝试通过 `CAS` 将 `AQS` 的 `state` 由0设为1，如果state不为0那么会设置失败，如果设置成功就可以证明Worker并没有在执行任务。那么什么时候 `state` 为0呢，就是在 `Worker` 对象 `unlock` 之后， `lock` 之前。回顾下 `runWorker` 方法，在 `Worker` 尝试从任务队列获取任务时，就处于这个状态。

 `interruptIdleWorkers` 方法结束后，就会调用 `onShutdown` 方法，这个方法在 `ThreadPoolExecutor` 中实现为空（该方法为包访问权限），由同包下的子类 `ScheduledThreadPoolExecutor` 实现。

 最后会调用 `tryTerminate` 来彻底关闭线程池：

 
```
final void tryTerminate() {
    for (;;) {
        int c = ctl.get();
        //如果线程池正在运行或处于TIDYING状态，或者线程池的任务队列不为空，那么该方法直接返回
        if (isRunning(c) || runStateAtLeast(c, TIDYING) || (runStateOf(c) == SHUTDOWN && ! workQueue.isEmpty()))
            return;
        //如果线程池还持有活动的线程
        if (workerCountOf(c) != 0) {
	        //尝试中断一个活动的线程
            interruptIdleWorkers(ONLY_ONE);
            return;
        }
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
	        //将线程池状态设为TIDYING
            if (ctl.compareAndSet(c, ctlOf(TIDYING, 0))) {
                try {
	                //状态变为TIDYING的代码逻辑（默认实现为空，留给子类定义）
                    terminated();
                } finally {
	                //将状态设为TERMINATED，线程数量设为0
                    ctl.set(ctlOf(TERMINATED, 0));
                    //唤醒所有等待在termination上的线程（调用awaitTermination方法的线程）
                    termination.signalAll();
                }
                return;
            }
        } finally {
            mainLock.unlock();
        }
    }
}

```
 `tryTerminate` 方法会让线程池从状态 `SHUTDOWN` 到 `TIDYING` 再到 `TERMINATED` 状态。  
  `shutdownNow` 方法：

 
```
public List<Runnable> shutdownNow() {
	List<Runnable> tasks;
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
	    //和shutdown方法一样，首先检查调用者权限
        checkShutdownAccess();
        //将线程池状态更改为STOP
        advanceRunState(STOP);
        //中断所有的线程
        interruptWorkers();
        //返回任务队列中尚未执行完的任务
        tasks = drainQueue();
    } finally {
        mainLock.unlock();
    }
    tryTerminate();
    //返回尚未执行完的任务集合
    return tasks;
}

```
 和 `shutdown` 方法相似， `shutdownNow` 方法同样首先会检查调用者权限，通过权限认证后更改线程池状态（更改为 `STOP` ， `shutdown` 方法会更改为 `SHUTDOWN` ）。和 `shutdown` 方法不同的是， `shutdownNow` 方法接着会调用 `interruptWorkers` 中断所有正在运行的线程（ `AQS` 的 `state` 状态为0或1）：

 
```
private void interruptWorkers() {
	final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
	    //遍历workes集合，中断所有正在运行的线程
        for (Worker w : workers)
            w.interruptIfStarted();
    } finally {
        mainLock.unlock();
    }
}

```
 接着，调用 `drainQueue` 遍历任务队列，将其中的 `Runnable` 任务对象组装为一个集合返回：

 
```
private List<Runnable> drainQueue() {
    BlockingQueue<Runnable> q = workQueue;
    ArrayList<Runnable> taskList = new ArrayList<Runnable>();
    //将任务队列中的元素添加到taskList中
    q.drainTo(taskList);
    //清空任务队列
    if (!q.isEmpty()) {
        for (Runnable r : q.toArray(new Runnable[0])) {
            if (q.remove(r))
                taskList.add(r);
        }
    }
    return taskList;
}

```
 最后和 `shutdown` 方法类似，调用 `tryTerminate` 方法彻底关闭线程池。

 
--------
 
### []()三、Executors和ThreadPoolExecutor

 `Executors` 作为线程池的静态工厂提供了诸多创建线程池的方法，用户无需再直接 `new` 一个 `ThreadPoolExecutor` 。

 
##### []()1、创建一个可缓存的线程池

 可以调用 `Executors` 的 `newCacheThreadPool()` 方法创建一个可缓存的线程池，当线程池中的线程处于空闲状态时，线程池会自动回收这些空闲的线程。

 
```
public static ExecutorService newCachedThreadPool() {
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>());
}

```
 该方法指定 `ThreadPoolExecutor` 的核心线程数量为0，最大线程数量为 `Integer.MAX_VALUE` ，线程空闲回收时间设置为60秒，并以 `SynchronousQueue` 的实例为任务队列。

 
##### []()2、创建一个恒定线程数量的线程池

 该方法会创建一个定长线程池，指定最大并发数为 `nThreads` 。

 
```
public static ExecutorService newFixedThreadPool(int nThreads) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>());
    }

```
 其核心线程数量和最大线程数量都由该方法指定，并且线程被创建后不会被回收。

   
  