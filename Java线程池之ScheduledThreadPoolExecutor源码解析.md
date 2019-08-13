---
title: Java线程池之ScheduledThreadPoolExecutor源码解析
date: 2018-09-25 13:03:34
tags: CSDN迁移
---
 版权声明：尊重原创，转载请注明出处 [ https://blog.csdn.net/abc123lzf/article/details/82823612]( https://blog.csdn.net/abc123lzf/article/details/82823612)   
  ### []()一、引言

 ScheduledThreadPoolExecutor是Java并发工具包中的线程池实现类，可以用来执行定时、周期任务。相对于java.util.Timer，ScheduledThreadPoolExecutor性能更好，也能够提供更好的稳定性。  
 Java线程池UML类图：  
 ![在这里插入图片描述](https://img-blog.csdn.net/20180923175623366?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FiYzEyM2x6Zg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)  
 可以看出，ScheduledThreadPoolExecutor继承了ThreadPoolExecutor，实现了ScheduledExecutorService接口。  
 所以，**在了解ScheduledThreadPoolExecutor前，需要首先理解ThreadPoolExecutor、FutureTask的原理**。

 在Executors类中，可以调用静态方法newScheduledThreadPool方法获取ScheduledThreadPoolExecutor实例：

 
```
public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {
	return new ScheduledThreadPoolExecutor(corePoolSize);
}

public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize, ThreadFactory threadFactory) {
    return new ScheduledThreadPoolExecutor(corePoolSize, threadFactory);
}

```
 其中，参数corePoolSize可以指定线程池核心线程数量，threadFactory可以用来指定一个线程工厂。

 对于接口java.util.concurrent.ScheduledExecutorService，其源码如下：

 
```
public interface ScheduledExecutorService extends ExecutorService {
	//创建一个Runnable定时任务，并在delay时间后执行
	public ScheduledFuture<?> schedule(Runnable command, long delay, TimeUnit unit);
	//创建一个Callable定时任务，并在delay时间后执行
	public <V> ScheduledFuture<V> schedule(Callable<V> callable, long delay, TimeUnit unit);
	//创建一个Runnable周期任务，在initialDelay时间开始执行，每隔period时间再执行一次
	public ScheduledFuture<?> scheduleAtFixedRate(Runnable command, long initialDela,y long period, TimeUnit unit);
	//创建一个Runnable周期任务，在initialDelay时间开始执行，以一次任务结束的时间为起点，每隔delay时间再执行一次
	public ScheduledFuture<?> scheduleWithFixedDelay(Runnable command, long initialDelay, long delay, TimeUnit unit);
}

```
 返回的ScheduledFuture可以用来查看任务状态、获取定时任务的执行结果、获取下次执行的时间。

 
```
public interface ScheduledFuture<V> extends Delayed, Future<V> {
}

```
 ScheduledFuture继承了Delayed、Future接口，Delayed接口用来获取下次任务的执行时间，Future用来获取任务状态。泛型参数V为任务执行结果的类型。  
 ![在这里插入图片描述](https://img-blog.csdn.net/2018092415385664?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FiYzEyM2x6Zg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

 了解完这些接口后，我们来分析ScheduledThreadPoolExecutor的源码实现。

 
--------
 
### []()二、源码分析

 ScheduledThreadPoolExecutor类定义及其成员变量：

 
```
public class ScheduledThreadPoolExecutor extends ThreadPoolExecutor implements ScheduledExecutorService {
	//是否在线程池处于正在关闭的状态后继续执行周期任务，默认为false
	private volatile boolean continueExistingPeriodicTasksAfterShutdown;
	//是否在线程池处于正在关闭的状态后继续执行非周期任务，默认为true
	private volatile boolean executeExistingDelayedTasksAfterShutdown = true;
	//当任务被取消后是否应当从任务队列中移除
	private volatile boolean removeOnCancel = false;
	//用来给任务加上序列号
	private static final AtomicLong sequencer = new AtomicLong();
}

```
 
##### []()1、构造方法

 ScheduledThreadPoolExecutor提供了4个public构造方法，方法签名及其作用如下：

 
     构造方法                                                                    | 解释                               
     ----------------------------------------------------------------------- | --------------------------------- 
     ScheduledThreadPoolExecutor(int)                                        | 创建一个指定核心线程数量的线程池                 
     ScheduledThreadPoolExecutor(int,ThreadFactory)                          | 创建一个指定核心线程数量的线程池，并可以指定一个线程工厂     
     ScheduledThreadPoolExecutor(int,RejectExecutionHandler)                 | 创建一个指定核心线程数量的线程池，并可以指定饱和(拒绝执行)处理器
     ScheduledThreadPoolExecutor(int,ThreadFactory,RejectedExecutionHandler) | 作用同上                             

这些构造方法都是基于父类ScheduledThreadPoolExecutor的构造器实现的。

 下面通过源码对这些构造方法进行解析：

 
```
public ScheduledThreadPoolExecutor(int corePoolSize) {
	super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS, new DelayedWorkQueue());
}

```
 该方法会构造一个核心线程数量为corePoolSize，最大线程数量为Integer.MAX_VALUE，而且规定了非核心线程在空闲时会立刻被回收，指定ScheduledThreadPoolExecutor的内部类DelayedWorkQueue为任务队列。  
 对于线程工厂，默认采用的是Executors类的DefaultThreadFactory。对于饱和处理器，默认是ThreadPoolExecutor.AbortPolicy类。  
 如果想要自定义线程工厂或者饱和处理器，可以调用以下构造方法：

 
```
public ScheduledThreadPoolExecutor(int corePoolSize, ThreadFactory threadFactory) {
	super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS, new DelayedWorkQueue(), threadFactory);
}

public ScheduledThreadPoolExecutor(int corePoolSize, RejectedExecutionHandler handler) {
    super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS, new DelayedWorkQueue(), handler);
}

public ScheduledThreadPoolExecutor(int corePoolSize, ThreadFactory threadFactory, RejectedExecutionHandler handler) {
	super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS, new DelayedWorkQueue(), threadFactory, handler);
}

```
 
##### []()2、ScheduledThreadPoolExecutor的任务队列：DelayedWorkQueue

 DelayedWorkQueue是ScheduledThreadPoolExecutor的静态内部类，它实现了BlockingQueue接口，负责存储线程池的任务。

 
```
static class DelayedWorkQueue extends AbstractQueue<Runnable> implements BlockingQueue<Runnable> {
	//初始化数组大小
	private static final int INITIAL_CAPACITY = 16;
	//存储任务的数组
	private RunnableScheduledFuture<?>[] queue = new RunnableScheduledFuture<?>[INITIAL_CAPACITY];
	//线程安全锁
	private final ReentrantLock lock = new ReentrantLock();
	//元素数量
	private int size = 0;
	//领导线程
	private Thread leader = null;
	//和lock对应的Condition用于线程的等待和唤醒
	private final Condition available = lock.newCondition();
}

```
 DelayedWorkQueue内部通过一个RunnableScheduledFuture数组保存Runnable任务，默认初始长度为16。RunnableScheduledFuture接口继承了RunnableFuture，ScheduledFuture接口（参考引言中的UML类图），RunnableScheduledFuture额外增加了一个方法定义：boolean isPeriodioc()，该方法返回这个Runnable任务是否是定时任务。

 **元素的添加操作：offer方法**

 
```
public boolean offer(Runnable x) {
	if (x == null)
    	throw new NullPointerException();
    //将插入的元素强制转换成RunnableScheduledFuture类型
    RunnableScheduledFuture<?> e = (RunnableScheduledFuture<?>)x;
    //获取锁并加锁，保证线程安全
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        int i = size;
        //如果数组的长度不够，那么调用grow方法增加长度
        if (i >= queue.length)
            grow();
        size = i + 1;
        //如果数组元素数量为0，那么放入数组首部
        if (i == 0) {
            queue[0] = e;
            setIndex(e, 0);
        //否则调用siftUp方法入队
        } else {
            siftUp(i, e);
        }
        //如果队列头部就是插入的元素（即执行时间最近的任务），那么唤醒一个在available上等待的线程
        if (queue[0] == e) {
            leader = null;
            available.signal();
        }
    } finally {
  		lock.unlock();
    }
    return true;
}

```
 grow方法的分配策略和ArrayList相同，都是将原数组的长度扩大到原来的1.5倍。  
 setIndex方法主要是调整ScheduledFutureTask的heapindex变量，关于ScheduledFutureTask我们稍作讨论。  
 其中，siftUp方法的实现如下：

 
```
private void siftUp(int k, RunnableScheduledFuture<?> key) {
	while (k > 0) {
		//相当于将k/2
    	int parent = (k - 1) >>> 1;
        RunnableScheduledFuture<?> e = queue[parent];
        //比较大小，直到key的执行时间晚于queue[parent]
        if (key.compareTo(e) >= 0)
            break;
        queue[k] = e;
        setIndex(e, k);
        k = parent;
    }
    //向数组插入元素
    queue[k] = key;
    setIndex(key, k);
}

```
 可以看出，DelayedWorkQueue实际上是一个基于数组实现的优先队列，因为Delayed接口继承了Comparable接口，它根据这个任务的下次执行时间来比较大小，这样能够保证queue[0]位置上的元素是最近需要执行的任务。

 此外，put、add方法实际上都是调用这个offer方法来完成元素的插入操作，不会抛出非运行时异常。

 **元素的取出操作：take方法**

 
```
public RunnableScheduledFuture<?> take() throws InterruptedException {
	final ReentrantLock lock = this.lock;
	//尝试获得锁，如果线程被Interrupt则方法抛出异常，随即结束
    lock.lockInterruptibly();
    try {
    	//自旋操作，直到获取到元素为止
        for (;;) {
            RunnableScheduledFuture<?> first = queue[0];
            //如果队列头部为空，那么在available上等待，同时释放锁
            if (first == null)
                available.await();
            else {
            	//获取这个任务开始执行的时间，单位为纳秒
                long delay = first.getDelay(NANOSECONDS);
                //如果小于等于0，说明已经达到执行时间，从队列中弹出这个元素并返回
                if (delay <= 0)
                    return finishPoll(first);
                first = null;
                //如果领导线程不为null，那么在available上进行等待
                if (leader != null)
                    available.await();
                else {
                	//否则将当前线程作为领导线程
                    Thread thisThread = Thread.currentThread();
                    leader = thisThread;
                    try {
                    	//当前线程等待直到delay纳秒
                        available.awaitNanos(delay);
                    } finally {
                    	//如果领导线程依然是当前线程，那么设置为null
                        if (leader == thisThread)
                            leader = null;
                    }
                }
            }
        }
    } finally {
    	//如果领导线程不为null并且队列头部不为空，那么唤醒一个在available上等待的线程
        if (leader == null && queue[0] != null)
            available.signal();
        lock.unlock();
    }
}

```
 take方法会不断地尝试获取队首的任务对象，如果此时任务执行时间尚未达到或者任务队列为空，那么take方法会让当前线程在Condition上等待，唤醒后会再次尝试进行上述操作。  
 找到目标任务后，会调用finishPoll方法执行后续操作：

 
```
private RunnableScheduledFuture<?> finishPoll(RunnableScheduledFuture<?> f) {
	int s = --size;
	//弹出元素
    RunnableScheduledFuture<?> x = queue[s];
    queue[s] = null;
    //如果此时队列不为空，那么调整优先队列
    if (s != 0)
        siftDown(0, x);
    setIndex(f, -1);
    return f;
 }

```
 
##### []()3、任务对象：ScheduledFutureTask

 在ScheduledThreadPoolExecutor中，每个任务都是以ScheduledFutureTask的形式存在的，它继承了FutureTask类，实现了RunnableScheduledFuture接口。在上面的DelayedWorkQueue中，其数组的元素就是ScheduledFutureTask。

 
```
private class ScheduledFutureTask<V> extends FutureTask<V> implements RunnableScheduledFuture<V> {
	//任务序列号
	private final long sequenceNumber;
	//开始执行的时间
	private long time;
	//执行周期，为0时表示该任务不是周期任务
	private final long period;
	//this的包装任务，用于周期任务的提交
	RunnableScheduledFuture<V> outerTask = this;
	//在任务队列中数组的位置，为-1时表示已经出队
	int heapIndex;
	//构造一个定时任务，在ns纳秒后执行，执行完成后结果为result
	ScheduledFutureTask(Runnable r, V result, long ns) {
        super(r, result);
        this.time = ns;
        this.period = 0;
        this.sequenceNumber = sequencer.getAndIncrement();
    }
	//构造一个周期任务，在ns纳秒后执行，每隔period纳秒执行一次
	ScheduledFutureTask(Runnable r, V result, long ns, long period) {
        super(r, result);
        this.time = ns;
        this.period = period;
        this.sequenceNumber = sequencer.getAndIncrement();
    }
	//构造一个Callable定时任务，在ns纳秒后执行
	ScheduledFutureTask(Callable<V> callable, long ns) {
        super(callable);
        this.time = ns;
        this.period = 0;
        this.sequenceNumber = sequencer.getAndIncrement();
    }
}

```
 对于成员变量period，当period大于0时，表示下次执行任务的时间为 time + n * period（n为正整数）。当period小于0时，会按照任务执行结束的时间往后-period纳秒为下次任务的执行时间。

 在DelayedWorkQueue中，ScheduledFutureTask的大小比较是通过compareTo方法实现的，因为Delayed接口继承了Comparable接口。

 
```
public int compareTo(Delayed other) {
	if (other == this)
        return 0;
    //如果比较的对象是ScheduledFutureTask
    if (other instanceof ScheduledFutureTask) {
        ScheduledFutureTask<?> x = (ScheduledFutureTask<?>)other;
        //比较执行时间
        long diff = time - x.time;
        //如果this先执行，返回-1，如果this后执行，返回1
        if (diff < 0)
			return -1;
        else if (diff > 0)
            return 1;
        //如果diff等于0，那么比较序列号
        else if (sequenceNumber < x.sequenceNumber)
            return -1;
        else
            return 1;
    }
    long diff = getDelay(NANOSECONDS) - other.getDelay(NANOSECONDS);
    return (diff < 0) ? -1 : (diff > 0) ? 1 : 0;
}

```
 ScheduledFutureTask重写了父类FutureTask的run、cancel方法。

 
```
public void run() {
	//判断是不是周期任务
	boolean periodic = isPeriodic();
	//在线程池处于正在关闭状态是能否继续执行，默认是false
    if (!canRunInCurrentRunState(periodic))
    	cancel(false);
   	//如果不是周期任务，那么调用父类的run方法执行任务
    else if (!periodic)
        ScheduledFutureTask.super.run();
    //如果是周期任务，那么调用父类的runAndReset方法执行任务，调用成功后，设置下次执行的时间，并加入到任务队列
    else if (ScheduledFutureTask.super.runAndReset()) {
        setNextRunTime();
        reExecutePeriodic(outerTask);
    }
}

```
 在调用run方法的时候，会检测线程池的状态，如果线程池处于正在关闭的状态（比如调用shutdown方法但尚未执行完成），会调用canRunInCurrentRunState方法，这个方法根据ScheduledThreadPoolExecutor的成员变量continueExistingPeriodicTasksAfterShutdown决定是否继续执行：

 
```
//参数periodic为是否是周期任务
boolean canRunInCurrentRunState(boolean periodic) {
	return isRunningOrShutdown(periodic ? continueExistingPeriodicTasksAfterShutdown :
                                   executeExistingDelayedTasksAfterShutdown);
}

```
 ScheduledThreadPoolExecutor提供了成员变量continueExistingPeriodicTasksAfterShutdown和executeExistingDelayedTasksAfterShutdown的public权限的getter/setter方法，程序可以根据自己的需求进行调整。

 经过上述判断后，任务的执行就正式开始了：对于一般的定时任务，默认通过调用父类FutureTask的run方法执行任务。对于周期任务，则是调用父类的runAndReset方法执行任务。对于周期任务，每执行一次，就会调用setNextRunTime方法来算出下次执行的时间：

 
```
private void setNextRunTime() {
	long p = period;
    if (p > 0)
        time += p;
    else
        time = triggerTime(-p);
}

```
 setNextRunTime方法会根据周期任务的执行策略来算出下次执行的时间，当period为正数时，说明用户调用的是scheduleAtFixedRate提交的周期任务，反之则是调用的scheduleWithFixedDelay提交的任务。前者会严格按照每隔时间period纳秒就执行一次，后者会根据任务实际的完成时间为起点往后推triggerTime纳秒作为下次的执行时间。  
 设定完时间后，就会调用reExecutePeriodic方法来使任务重新加入到任务队列：

 
```
void reExecutePeriodic(RunnableScheduledFuture<?> task) {
	//再次检测线程池状态，并进行上述判断
	if (canRunInCurrentRunState(true)) {
		//重新入队
    	super.getQueue().add(task);
        if (!canRunInCurrentRunState(true) && remove(task))
            task.cancel(false);
        else
        	//确保线程池有工作线程
            ensurePrestart();
    }
}

```
 如果想取消任务，可以调用它的cancel方法：  
 cancel方法：

 
```
public boolean cancel(boolean mayInterruptIfRunning) {
	//首先调用父类的cancel方法
	boolean cancelled = super.cancel(mayInterruptIfRunning);
	//如果任务成功被取消，那么根据removeOnCancel的策略决定是否从任务队列中移除这个任务
    if (cancelled && removeOnCancel && heapIndex >= 0)
        remove(this);
    return cancelled;
}

```
 cancel方法可以取消这个任务，它首先会调用父类FutureTask的cancel方法，然后根据ScheduledThreadPoolExecutor的成员变量removeOnCancel（默认为false）决定是否从任务队列中移除这个任务。

 
##### []()4、任务的提交

 ScheduledThreadPoolExecutor可以调用很多方法提交任务，它重写了父类的execute、submit方法，并实现了ScheduledExecutorService接口的schedule、scheduleAtFixedRate、scheduleWithFixedDelay方法，这些方法都可以用来提交任务。

 **定时任务的提交：schedule方法**

 
```
public ScheduledFuture<?> schedule(Runnable command, long delay, TimeUnit unit) {
	if (command == null || unit == null)
        throw new NullPointerException();
    //构造一个ScheduledFutureTask，并调用decorateTask方法
    RunnableScheduledFuture<?> t = decorateTask(command, 
    		new ScheduledFutureTask<Void>(command, null, triggerTime(delay, unit)));
    //提交任务
    delayedExecute(t);
    return t;
}

```
 decorateTask的默认实现为直接返回第二个参数：

 
```
protected <V> RunnableScheduledFuture<V> decorateTask(Runnable runnable, RunnableScheduledFuture<V> task) {
	return task;
}

```
 子类可以根据自己实际的业务需求来重写这个方法。  
 构造好RunnableScheduledFuture后，schedule方法会继续调用delayedExecutor方法向线程池提交任务

 
```
private void delayedExecute(RunnableScheduledFuture<?> task) {
	//如果线程池已被关闭，那么将任务转交给饱和处理器处理
    if (isShutdown())
        reject(task);
    else {
    	//向任务队列中添加任务
        super.getQueue().add(task);
        //如果被添加后，线程池已被关闭，并且根据策略不可继续执行，那么从任务队列中移除这个任务，成功后调用cancel方法取消任务
        if (isShutdown() && !canRunInCurrentRunState(task.isPeriodic()) && remove(task))
            task.cancel(false);
    	else
    		//确保线程池中有线程执行任务
            ensurePrestart();
    }
}

```
 scheduled还有一个重载的方法：

 
```
public <V> ScheduledFuture<V> schedule(Callable<V> callable, long delay, TimeUnit unit) {
    if (callable == null || unit == null)
        throw new NullPointerException();
    RunnableScheduledFuture<V> t = decorateTask(callable, new ScheduledFutureTask<V>(callable, triggerTime(delay, unit)));
    delayedExecute(t);
    return t;
}

```
 其实现大致相同，只不过是提交的Callable任务。

 **周期任务的提交：scheduleAtFixedRate、scheduleWithFixedDelay**

 
```
public ScheduledFuture<?> scheduleAtFixedRate(Runnable command, long initialDelay, long period, TimeUnit unit) {
    if (command == null || unit == null)
        throw new NullPointerException();
    if (period <= 0)
        throw new IllegalArgumentException();
    ScheduledFutureTask<Void> sft = new ScheduledFutureTask<Void>(command, null, 
   			 triggerTime(initialDelay, unit), unit.toNanos(period));
    RunnableScheduledFuture<Void> t = decorateTask(command, sft);
    //将outerTask变量设为t
    sft.outerTask = t;
    delayedExecute(t);
    return t;
}

public ScheduledFuture<?> scheduleWithFixedDelay(Runnable command, long initialDelay, long delay, TimeUnit unit) {
    if (command == null || unit == null)
        throw new NullPointerException();
    if (delay <= 0)
        throw new IllegalArgumentException();
    ScheduledFutureTask<Void> sft = new ScheduledFutureTask<Void>(command, null, 
    					triggerTime(initialDelay, unit), unit.toNanos(-delay));
    RunnableScheduledFuture<Void> t = decorateTask(command, sft);
    sft.outerTask = t;
    delayedExecute(t);
    return t;
}

```
 这两个方法的区别，在引言中也介绍了，这里就不再赘述了。

 ScheduledThreadPoolExecutor任务的执行策略和父类ThreadPoolExecutor相同，这里就不再赘述了。

 
##### []()5、线程池的关闭

 ScheduledThreadPoolExecutor的shutdown方法默认调用的是父类的shutdown方法，它和父类ThreadPoolExecutor的区别在于它重写了onShutdown方法，这个方法在父类中的实现为空。

 在shutdown方法中，onShutdown方法会在tryTerminate方法之前调用。

 
```
@Override 
void onShutdown() {
	BlockingQueue<Runnable> q = super.getQueue();
    boolean keepDelayed = getExecuteExistingDelayedTasksAfterShutdownPolicy();
    boolean keepPeriodic = getContinueExistingPeriodicTasksAfterShutdownPolicy();
    //如果两者都为false，那么遍历任务队列取消所有的任务并清空队列
    if (!keepDelayed && !keepPeriodic) {
        for (Object e : q.toArray())
            if (e instanceof RunnableScheduledFuture<?>)
                ((RunnableScheduledFuture<?>) e).cancel(false);
        q.clear();
    //否则，根据上述两个策略取消任务并从任务队列中移除
    } else {
        for (Object e : q.toArray()) {
            if (e instanceof RunnableScheduledFuture) {
                RunnableScheduledFuture<?> t = (RunnableScheduledFuture<?>)e;
                if ((t.isPeriodic() ? !keepPeriodic : !keepDelayed) || t.isCancelled()) {
                    if (q.remove(t))
                        t.cancel(false);
                }
            }
        }
    }
    tryTerminate();
}

```
 对于shutdownNow方法，其执行策略和ThreadPoolExecutor相同。

   
  