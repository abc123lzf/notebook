---
title: java.util.concurrent.FutureTask源码解析
date: 2018-09-15 22:18:46
tags: CSDN迁移
---
 版权声明：尊重原创，转载请注明出处 [ https://blog.csdn.net/abc123lzf/article/details/82715737]( https://blog.csdn.net/abc123lzf/article/details/82715737)   
  **本文参照的是JDK1.8版本的FutureTask源码**

 
### 一、引言

 FutureTask可以用来封装一个Runnable或者Callable任务，并异步执行，当用户想要返回的结果时，只需要调用get方法获取。

 FutureTask继承关系图：   
 ![这里写图片描述](https://img-blog.csdn.net/20180915170214591?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FiYzEyM2x6Zg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)   
 FutureTask继承了RunnableFuture接口，它可以直接作为一个Runnable或Callable提交到线程池执行。

 
--------
 
### 二、源码分析

 FutureTask的实例变量：

 
```
//任务的状态
private volatile int state;
//任务对象
private Callable<V> callable;
//保存任务返回值(或任务执行期间未捕获的异常)
private Object outcome;
//正在运行任务的线程
private volatile Thread runner;
//等待获取任务结果的线程队列(头结点)
private volatile WaitNode waiters;
```
 state变量表示当前任务的状态，任务状态仅有一下几种：

 
```
//任务尚未开始或处于执行期间
private static final int NEW          = 0;
//任务即将执行完成
private static final int COMPLETING   = 1;
//任务执行完毕
private static final int NORMAL       = 2;
//任务执行期间出现未捕获异常
private static final int EXCEPTIONAL  = 3;
//任务被取消
private static final int CANCELLED    = 4;
//任务正在被中断
private static final int INTERRUPTING = 5;
//任务已被中断
private static final int INTERRUPTED  = 6;
```
 状态仅有一下几种变化情况：   
 1、NEW -> COMPLETING -> NORMAL（正常执行完成）   
 2、NEW -> COMPLETING -> EXCEPTIONAL（执行期间有未捕获异常）   
 3、NEW -> CANCELLED（任务尚未执行，被取消）   
 4、NEW -> INTERRUPTING -> INTERRUPTED（任务被中断，在执行完成之前）

 对于任务对象，即使在构造阶段传入的是Runnable，也会被内部包装成Callable。   
 每个FutureTask都持有一个等待队列，当任务结束前，如果有线程尝试调用get方法获取其返回值，那么这个线程会被加入到这个队列等待任务完成后由FutureTask激活。   
 runner变量保存了**正在**执行任务的线程，如果runner不为null代表有线程正在执行任务

 等待队列在FutureTask中是一个单向链表，链表的结点为内部类WaitNode：

 
```
static final class WaitNode {
    volatile Thread thread;
    volatile WaitNode next;
    WaitNode() { thread = Thread.currentThread(); }
}
```
 WaitNode保存了等待线程对象和维护链表的后驱指针。

 FutureTask所有方法都是线程安全的，它没有通过加锁来实现线程安全，而是通过sun.misc.Unsafe类中的CAS方法来实现：

 
```
private static final sun.misc.Unsafe UNSAFE;
private static final long stateOffset; //state成员变量的内存偏移量
private static final long runnerOffset; //runner成员变量的内存偏移量
private static final long waitersOffset; //waiters成员变量的内存偏移量
static {
    try {
        UNSAFE = sun.misc.Unsafe.getUnsafe();
        Class<?> k = FutureTask.class;
        stateOffset = UNSAFE.objectFieldOffset(k.getDeclaredField("state"));
        runnerOffset = UNSAFE.objectFieldOffset(k.getDeclaredField("runner"));
        waitersOffset = UNSAFE.objectFieldOffset(k.getDeclaredField("waiters"));
    } catch (Exception e) {
        throw new Error(e);
    }
}
```
 
##### 1、构造方法

 FutureTask提供了两个public构造方法，分别用来传入Callable和Runnable任务

 
     构造方法                                 | 解释                                 
     ------------------------------------ | ----------------------------------- 
     public FutureTask(Callable&lt;V&gt;) | 传入一个Callable任务                     
     public FutureTask(Runnable, V)       | 传入一个Runnable和返回结果(任务完成后调用get方法的返回值)

 源码实现：

 
```
public FutureTask(Callable<V> callable) {
    if (callable == null)
        throw new NullPointerException();
    this.callable = callable;
    this.state = NEW;
}

public FutureTask(Runnable runnable, V result) {
    //调用Executors的callable方法将其包装为一个Callable类
    this.callable = Executors.callable(runnable, result);
    this.state = NEW;
}
```
 构造方法执行过程比较简单，主要任务是将任务对象的引用赋给callable成员变量，并将state状态设置为NEW。   
 下面我将Runnable、Callable统称为任务（任务对象）

 
##### 2、执行任务：run方法

 这个run方法是Runnable接口要求实现的，由线程对象负责调用此方法来异步执行

 
```
public void run() {
    //如果状态不属于NEW，那么通过CAS将runner变量由null设为当前线程，
    //如果设置失败方法返回（保证只有1个线程执行run方法）
    if (state != NEW ||
            !UNSAFE.compareAndSwapObject(this, runnerOffset,
                                         null, Thread.currentThread()))
            return;
    try {
        //获取构造时传入的Callable任务对象
        Callable<V> c = callable;
        //如果状态为NEW
        if (c != null && state == NEW) {
            V result;
            //任务是否正常完成
            boolean ran;
            try {
                //调用Callable的call方法执行任务
                result = c.call();
                //如果没有未捕获的异常
                ran = true;
            //如果任务执行期间有未捕获异常导致任务中断
            } catch (Throwable ex) {
                //没有返回值
                result = null;
                ran = false;
                //设置异常对象，由调用get方法的线程处理这个异常
                setException(ex);
            }
            //如果任务正常结束则设置返回值和state变量
            if (ran)
                set(result);
        }
    } finally {
        runner = null;
        //获取state状态
        int s = state;
        //如果处于任务正在中断状态，则等待直到任务处于已中断状态位置
        if (s >= INTERRUPTING)
            handlePossibleCancellationInterrupt(s);
    }
}
```
 FutureTask的state状态都是由sun.misc.Unsafe类中的CAS相关方法进行设置的，目的是在不加锁的情况下防止有多个线程尝试进行修改这个state变量，只允许在多线程竞争情况下有1个线程能够修改成功。这些方法都是本地方法，由C++代码实现。

 如果任务正常结束，会调用set方法：

 
```
protected void set(V v) {
    //通过CAS将NEW设为COMPLETING(即将完成)状态
    if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
        //将任务的返回值设置为
        outcome = v;
        UNSAFE.putOrderedInt(this, stateOffset, NORMAL); // final state
        finishCompletion();
    }
}
```
 如果任务执行期间有未捕获的异常，那么会调用以下方法设置异常：

 
```
protected void setException(Throwable t) {
    //通过CAS将state由NEW设为COMPLETING
    if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
        //将异常对象赋给outcome实例变量
        outcome = t;
        //将state设为EXCEPTIONAL（有异常抛出状态）
        UNSAFE.putOrderedInt(this, stateOffset, EXCEPTIONAL);
        //激活所有等待队列中的线程
        finishCompletion();
    }
}
```
 在run方法最后，会检测state是否为INTERRUPTING状态（任务中断状态），如果处于该状态则会调用handlePossibleCancellationInterrupt方法，该方法会等待state为INTERRUPTED为止

 
```
private void handlePossibleCancellationInterrupt(int s) {
    if (s == INTERRUPTING)
        //等待到INTERRUPTED时跳出循环
        while (state == INTERRUPTING)
            Thread.yield();
}
```
 当调用cancel(true)方法时，状态才有可能被置于INTERRUPTING。

 finishCompletion方法：

 
```
private void finishCompletion() {
    //不断获取队首
    for (WaitNode q; (q = waiters) != null;) {
        //通过CAS删除队列头部
        if (UNSAFE.compareAndSwapObject(this, waitersOffset, q, null)) {
            //如果删除成功，那么开始遍历这个队列
            for (;;) {
                //获取队列结点上的等待线程
                Thread t = q.thread;
                if (t != null) {
                    q.thread = null;
                    //激活该线程
                    LockSupport.unpark(t);
                }
                //如果已经达到队列尾部跳出循环
                WaitNode next = q.next;
                if (next == null)
                    break;
                q.next = null;
                q = next;
            }
            //跳出循环
            break;
        }
    }
    done();
    //删除任务对象引用
    callable = null;
}
```
 done方法非final的protected权限方法，默认实现为空，供子类重写，用来实现激活所有等待线程之后的后续操作。

 run方法执行流程图：   
 ![这里写图片描述](https://img-blog.csdn.net/20180915214957914?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FiYzEyM2x6Zg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

 
##### 3、中断（取消）任务：cancel方法

 cancel方法可以传入一个boolean参数，表示即使任务在运行是否应当中断(interrupt)它，该方法返回任务是否成功被中断

 
```
public boolean cancel(boolean mayInterruptIfRunning) {
    //如果任务状态为NEW并且成功通过CAS将state状态由NEW改为INTERRUPTING或CANCELLED（视参数而定）
    //那么方法继续执行，否则返回false
    if (!(state == NEW && UNSAFE.compareAndSwapInt(this, stateOffset, NEW,
                  mayInterruptIfRunning ? INTERRUPTING : CANCELLED)))
        return false;
    try {
        if (mayInterruptIfRunning) {
            try {
                //获取执行run方法的线程(执行任务的线程)
                Thread t = runner;
                //调用interrupt中断
                if (t != null)
                    t.interrupt();
            } finally {
                //将state状态设为INTERRUPTED(已中断)
                UNSAFE.putOrderedInt(this, stateOffset, INTERRUPTED);
            }
        }
    } finally {
        //激活所有在等待队列中的线程
        finishCompletion();
    }
    return true;
}
```
 对于mayInterruptIfRunning为true的情况（即使线程正在运行也尝试进行中断）：   
 1、如果任务状态处于NEW（任务尚未执行完成），那么尝试通过CAS将state由NEW设为INTERRUPTING（正在中断状态），如果设置失败，那么方法返回false，代表中断任务失败。   
 2、获取执行任务的线程对象，调用其interrupt方法，最后将state设为INTERRUPTED   
 3、激活所有等待队列中的线程（尝试通过get方法获取返回结果但是被阻塞的线程）

 对于mayInterruptIfRunning为false的情况：   
 1、如果任务状态处于NEW（任务尚未执行完成），那么尝试通过CAS将state由NEW设为CANCELLED（任务取消状态），如果设置失败，那么方法返回false，代表中断任务失败。   
 2、激活所有等待队列中的线程（尝试通过get方法获取返回结果但是被阻塞的线程）

 
##### 4、获取执行结果：get方法

 get方法有两个重载的方法：public V get()和public V get(long, TimeUnit)，前者线程会一直阻塞直到任务执行完毕或者被Interrupt，后者可以指定一个超时时间，当超出时间时任务依然没有执行完成会抛出TimeoutException

 
```
public V get() throws InterruptedException, ExecutionException {
    int s = state;
    //如果任务尚未执行完成，调用awaitDone使当前线程等待
    if (s <= COMPLETING)
        s = awaitDone(false, 0L);
    //任务执行完成后，调用report返回执行结果或抛出异常
    return report(s);
}

public V get(long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException {
    if (unit == null)
        throw new NullPointerException();
    int s = state;
    //如果任务没有完成那么调用awaitDone使当前线程等待，如果超时任务依然没有完成抛出TimeoutException
    if (s <= COMPLETING && (s = awaitDone(true, unit.toNanos(timeout))) <= COMPLETING)
        throw new TimeoutException();
    //同上
    return report(s);
}
```
 两个方法的实现其实差不多，都是调用了awaitDone方法将线程加入等待队列，然后调用report方法获取返回值或者未捕获的异常：

 
```
private int awaitDone(boolean timed, long nanos) throws InterruptedException {
    //获取超时的时间戳
    final long deadline = timed ? System.nanoTime() + nanos : 0L;
    WaitNode q = null;
    //是否成功入队
    boolean queued = false;
    //自旋操作
    for (;;) {
        //如果线程中断
        if (Thread.interrupted()) {
            //移除这个结点并抛出异常
            removeWaiter(q);
            throw new InterruptedException();
        }

        int s = state;
        //如果任务执行结束或被取消(中断)，方法结束
        if (s > COMPLETING) {
            if (q != null)
                q.thread = null;
            return s;
        //如果任务即将完成，让当前线程让步
        } else if (s == COMPLETING)
            Thread.yield();
        else if (q == null)
            q = new WaitNode();
        //如果没有入队，将这个WaitNode加入到FutureTask的等待队列尾部
        else if (!queued)
            queued = UNSAFE.compareAndSwapObject(this, waitersOffset, q.next = waiters, q);
        //如果设置了超时时间
        else if (timed) {
            nanos = deadline - System.nanoTime();
            //如果超时，那么从等待队列中移除这个结点，方法结束
            if (nanos <= 0L) {
                removeWaiter(q);
                return state;
            }
            //没有超时则阻塞当前线程nanos纳秒
            LockSupport.parkNanos(this, nanos);
        }
        //阻塞当前线程
        else
            LockSupport.park(this);
    }
}
```
 report方法负责获取任务执行的结果：

 
```
@SuppressWarnings("unchecked")
private V report(int s) throws ExecutionException {
    Object x = outcome;
    //如果任务正常执行完成直接返回结果
    if (s == NORMAL)
        return (V)x;
    //如果任务被取消或被中断抛出CancellationException（运行时异常的子类）
    if (s >= CANCELLED)
        throw new CancellationException();
    //如果任务有未捕获的异常则将异常包装到ExecutionException并抛出
    throw new ExecutionException((Throwable)x);
}
```
 
##### 5、任务的多次执行：runAndReset

 如果当前任务需要执行多次，那么需要调用runAndReset方法，该方法在执行任务完成后不会设置outcome变量，也不会激活等待的线程。

 
```
protected boolean runAndReset() {
    //和run方法类似，通过CAS操作将成员变量runner设置为当前线程
    if (state != NEW ||
            !UNSAFE.compareAndSwapObject(this, runnerOffset,
                                         null, Thread.currentThread()))
            return false;
    boolean ran = false;
    int s = state;
    try {
        Callable<V> c = callable;
        if (c != null && s == NEW) {
            try {
                //不会调用set方法设置返回值(outcome成员变量)
                c.call();
                ran = true;
            } catch (Throwable ex) {
                setException(ex);
            }
        }
    } finally {
        runner = null;
        s = state;
        if (s >= INTERRUPTING)
            handlePossibleCancellationInterrupt(s);
    }
    //这个方法不会修改state变量的值
    return ran && s == NEW;
}
```
 该方法默认是protected权限，如果想要通过外部调用该方法，需要子类重写这个方法并调用它。

   
  