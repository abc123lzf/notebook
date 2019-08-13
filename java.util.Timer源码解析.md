---
title: java.util.Timer源码解析
date: 2018-09-05 17:15:34
tags: CSDN迁移
---
 版权声明：尊重原创，转载请注明出处 [ https://blog.csdn.net/abc123lzf/article/details/82391337]( https://blog.csdn.net/abc123lzf/article/details/82391337)   
  ### 一、引言

 java.util.Timer是JDK提供的定时任务执行器，可以往Timer中添加定时任务并按期执行。   
 使用Timer首先需要创建Timer的实例，创建实例后可以通过调用schedule方法来创建任务，Timer中的定时任务需要用一个对象TimeTask表示，用户需要重写TimeTask的run方法来定义自己的任务。另外，Timer是线程安全的类。

 Timer是一个有缺陷的定时任务执行器，在新代码中已经不再推荐使用它，它目前已经被ScheduledExecutorService取代。下面我们会通过源码来分析缺陷到底在哪。

 
--------
 
### 二、源码分析

 首先来看Timer类的类定义和实例变量：

 
```
//定时任务队列
private final TaskQueue queue = new TaskQueue();
//执行定时任务的线程
private final TimerThread thread = new TimerThread(queue);
//当废弃这个Timer对象时，在GC之前执行finalize方法释放资源
private final Object threadReaper = new Object() {
    protected void finalize() throws Throwable {
        synchronized(queue) {
            thread.newTasksMayBeScheduled = false;
            queue.notify();
        }
    }
};
```
 Timer类只有三个实例变量：分别是保存定时任务的队列，执行定时任务的线程和用于释放资源的Object对象。   
 TimerThread是Thread的子类。   
 Timer类提供了4种public构造方法：

 
```
public Timer() {
    this("Timer-" + serialNumber());
}

public Timer(boolean isDaemon) {
    this("Timer-" + serialNumber(), isDaemon);
}

public Timer(String name) {
    thread.setName(name);
    thread.start();
}

public Timer(String name, boolean isDaemon) {
     thread.setName(name);
     thread.setDaemon(isDaemon);
     thread.start();
}
```
 在构造Timer类时，可以指定Timer的名称，也可以将这个执行定时任务的线程设置为后台线程（守护线程），如果不指定，则默认为非后台线程。   
 如果不指定名称，那么会将Timer的名称设为一个数字，这个数字根据当前线程上下文类加载器初始化了多少个Timer而定：

 
```
private final static AtomicInteger nextSerialNumber = new AtomicInteger(0);
private static int serialNumber() {
    return nextSerialNumber.getAndIncrement();
}
```
 构造完成后，定时任务执行线程也会随之启动。

 java.util.Timer类提供了多种方法提交定时任务：

 
```
//在delay毫秒后执行任务
public void schedule(TimerTask task, long delay);
//在指定的time(Date对象封装了一个时间戳)执行任务
public void schedule(TimerTask task, Date time);
//在delay毫秒后执行任务，并且根据实际开始执行的时间每隔period毫秒执行一次任务
public void schedule(TimerTask task, long delay, long period);
//在firstTime定义的时间后每隔period毫秒执行一次任务
public void schedule(TimerTask task, Date firstTime, long period);
//在delay毫秒后执行任务，每隔period毫秒后都会执行一次任务
public void scheduleAtFixedRate(TimerTask task, long delay, long period);
//在指定时间firstTime执行任务，并且从任务开始的时间每隔period执行一次任务
public void scheduleAtFixedRate(TimerTask task, Date firstTime, long period);
```
 TimeTask是一个抽象类，在定义任务时需要创建一个TimeTask的子类并重写run方法，run方法即任务逻辑。   
 schedule(TimeTask, long, long)和scheduleAtFixedRate(TimeTask, long, long)相同点是都会在delay毫秒开始执行任务，不同点是下一次执行任务的时间计算方式：   
 1、schedule方法在计算下次执行任务的时间时，会以任务线程实际执行开始的时间点为基准往后加上period毫秒。   
 2、scheduleAtFixedRate方法在计算下次执行任务的时间时，会严格按照delay+n*period时间戳执行

 这些public方法无一例外都调用了Timer内部的私有方法sched，在了解sched方法前，我们有必要首先了解下TimeTask类和任务执行队列TaskQueue的源码：

 **TimerTask**

 
```
public abstract class TimerTask implements Runnable {
    static final int VIRGIN = 0; //初始化状态
    static final int SCHEDULED = 1; //已计划状态，被添加到Timer类时为此状态
    static final int EXECUTED = 2; //任务已经被执行(定期循环执行的任务不会处于此状态)
    static final int CANCELLED = 3; //任务被取消时的状态

    //私有锁
    final Object lock = new Object();
    //初始化时状态为VIRGIN
    int state = VIRGIN;
    //任务下次执行时间
    long nextExecutionTime;
    //任务循环周期，0表示不循环
    long period = 0;
    //任务执行代码
    public abstract void run();

    //取消任务，返回操作是否成功
    public boolean cancel() {
        synchronized(lock) {
            boolean result = (state == SCHEDULED);
            state = CANCELLED;
            return result;
        }
    }  
    //返回计划执行时间戳
    public long scheduledExecutionTime() {
        synchronized(lock) {
            return (period < 0 ? nextExecutionTime + period 
                    : nextExecutionTime - period);
        }
    }
}
```
 TimerTask保存了一个任务的状态、开始执行的时间、任务循环周期。   
 其子类可以将任务执行代码重写到run方法中。

 **TaskQueue**   
 TaskQueue用来保存TimeTask任务对象。

 
```
class TaskQueue {
    //默认构造一个长度为128的任务队列
    private TimerTask[] queue = new TimerTask[128];
    //任务数量
    private int size = 0;

    int size() {
        return size;
    }

    //添加TimerTask任务对象
    void add(TimerTask task) {
        //如果任务数量超过数组queue的长度则将queue扩大到原来的2倍
        if (size + 1 == queue.length)
            queue = Arrays.copyOf(queue, 2*queue.length);
        //添加到尾部
        queue[++size] = task;
        //修复优先队列
        fixUp(size);
    }

    //将下次执行时间的时间戳最小的TimerTask置于queue[1]（优先队列算法）
    private void fixUp(int k) {
        while (k > 1) {
            int j = k >> 1;
            if (queue[j].nextExecutionTime <= queue[k].nextExecutionTime)
                break;
            //交换queue[j]和queue[k]
            TimerTask tmp = queue[j];  queue[j] = queue[k]; queue[k] = tmp;
            k = j;
        }
    }
    //返回下次执行时间的时间戳最小的TimerTask
    TimerTask getMin() {
        return queue[1];
    }

    //根据下标i获得任意一个TimeTask
    TimerTask get(int i) {
        return queue[i];
    }

    //移除时间戳最小的TimerTask
    void removeMin() {
        queue[1] = queue[size];
        queue[size--] = null;
        //修复队列
        fixDown(1);
    }

    //移除时间戳最小的TimerTask后修复优先队列
    private void fixDown(int k) {
        int j;
        while ((j = k << 1) <= size && j > 0) {
            if (j < size &&
                queue[j].nextExecutionTime > queue[j+1].nextExecutionTime)
                j++;
            if (queue[k].nextExecutionTime <= queue[j].nextExecutionTime)
                break;
            TimerTask tmp = queue[j];  queue[j] = queue[k]; queue[k] = tmp;
            k = j;
        }
    }

    //快速移除TimerTask，这个操作不会修复队列
    void quickRemove(int i) {
        assert i <= size;
        queue[i] = queue[size];
        queue[size--] = null;
    }

    //将时间戳最小的TimerTask的时间戳设为newTime，由TimerThread负责调用
    void rescheduleMin(long newTime) {
        queue[1].nextExecutionTime = newTime;
        //修复队列
        fixDown(1);
    }

    //返回优先队列是否为空
    boolean isEmpty() {
        return size == 0;
    }

    //清空所有的TimerTask
    void clear() {
        for (int i = 1; i <= size; i++)
            queue[i] = null;
        size = 0;
    }
}
```
 TaskQueue采用了优先队列，来保证queue[1]总是为时间戳最小的TimerTask。

 介绍完TimerTask和TaskQueue的源码后，我们再回过头来分析sched方法：

 
```
//time表示任务执行开始时间，period大于0时按照scheduleAtFixedRate定义的规则执行，
//等于0表示不循环执行任务，小于0则按schedule方法定义的规则执行
private void sched(TimerTask task, long time, long period) {
    if (time < 0)
        throw new IllegalArgumentException("Illegal execution time.");
    if (Math.abs(period) > (Long.MAX_VALUE >> 1))
        period >>= 1;
    //锁住任务队列，保证线程安全
    synchronized(queue) {
        //如果任务执行线程已被取消那么抛出异常
        if (!thread.newTasksMayBeScheduled)
            throw new IllegalStateException("Timer already cancelled.");
        synchronized(task.lock) {
            //如果TimerTask已经被添加到任务队列或
            if (task.state != TimerTask.VIRGIN)
                throw new IllegalStateException("Task already scheduled or cancelled");
            //设置任务的执行时间
            task.nextExecutionTime = time;
            //设置任务的循环执行时间间隔，为0则不循环
            task.period = period;
            //将状态设为已计划
            task.state = TimerTask.SCHEDULED;
        }
        //添加到任务队列
        queue.add(task);
        //如果这个任务是队列中时间戳最小的任务，那么激活任务线程
        if (queue.getMin() == task)
            queue.notify();
    }
}
```
 Timer类的私有方法sched可以将任务添加到Timer所持有的任务队列中。   
 TimerTask中的run方法由TimerThead负责执行，每个Timer都仅有一个TimerThread线程负责调用run方法。

 
```
class TimerThread extends Thread {
    //表示Timer是否正在运行，如果调用了Timer的cancel方法则会置为false
    boolean newTasksMayBeScheduled = true;
    //Timer持有的任务队列
    private TaskQueue queue;

    //由Timer类负责构造并传入任务队列
    TimerThread(TaskQueue queue) {
        this.queue = queue;
    }

    //线程执行代码
    public void run() {
        try {
            mainLoop();
        } finally {
            //释放任务队列中的任务对象
            synchronized(queue) {
                newTasksMayBeScheduled = false;
                queue.clear(); 
            }
        }
    }

    private void mainLoop() {
        while (true) {
            try {
                TimerTask task;
                boolean taskFired;
                synchronized(queue) {
                    //如果队列是空的并且Timer没有终止，就一直等待直到队列添加任务后唤醒它
                    while (queue.isEmpty() && newTasksMayBeScheduled)
                        queue.wait();
                    //如果队列依然是空的，说明Timer被终止了，然后结束该线程
                    if (queue.isEmpty())
                        break;
                    long currentTime, executionTime;
                    //获取最近需要执行的任务
                    task = queue.getMin();
                    synchronized(task.lock) {
                        //如果该任务被取消，则从任务队列中移除该任务重新开始本次循环
                        if (task.state == TimerTask.CANCELLED) {
                            queue.removeMin();
                            continue; 
                        }
                        //获取当前时间和任务的预定执行时间
                        currentTime = System.currentTimeMillis();
                        //获取任务的预计执行时间
                        executionTime = task.nextExecutionTime;
                        //如果当前时间戳大于任务预计执行时间，则将taskFired设为true，代表执行该任务
                        if (taskFired = (executionTime<=currentTime)) {
                            //如果该任务无需循环执行
                            if (task.period == 0) {
                                //从队列中移除这个任务
                                queue.removeMin();
                                //任务状态更改为已执行
                                task.state = TimerTask.EXECUTED;
                            } else {
                                //下次执行的时间根据添加任务时调用的是
                                //schedule还是scheduleAtFixedRate而定
                                queue.rescheduleMin(
                                  task.period<0 ? currentTime - task.period
                                                : executionTime + task.period);
                            }
                        }
                    }
                    //如果没有，则一直等待到预计任务执行时间等于系统时间
                    if (!taskFired)
                        queue.wait(executionTime - currentTime);
                }
                //执行任务
                if (taskFired)
                    task.run();
            } catch(InterruptedException e) {
                //仅捕获该异常，该线程继续执行
            }
        }
    }
}
```
 TimerThread有两个问题：   
 1、如果任务有一个未经处理的异常，那么就会导致TimeThread线程的终止，继而导致Timer所属的任务队列中的任务无法继续执行。   
 2、不论任务队列中有多少任务，这些任务都只能由一个线程串行执行，在单个任务执行时间长或任务数量过多的情况下，就很有可能无法保证任务能够按期执行。

 这些其实就是Timer类的两大缺陷之处。

 
```
/* <p>Java 5.0 introduced the {@code java.util.concurrent} package and
 * one of the concurrency utilities therein is the {@link
 * java.util.concurrent.ScheduledThreadPoolExecutor
 * ScheduledThreadPoolExecutor} which is a thread pool for repeatedly
 * executing tasks at a given rate or delay.  It is effectively a more
 * versatile replacement for the {@code Timer}/{@code TimerTask}
 * combination, as it allows multiple service threads, accepts various
 * time units, and doesn't require subclassing {@code TimerTask} (just
 * implement {@code Runnable}).  Configuring {@code
 * ScheduledThreadPoolExecutor} with one thread makes it equivalent to
 * {@code Timer}.*/
```
 Timer类的注释也说明了应当使用于JDK1.5引入ScheduledExecutorService。

   
  