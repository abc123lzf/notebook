---
title: java.lang.ref.Reference和ReferenceQueue源码解析
date: 2018-09-04 17:57:02
tags: CSDN迁移
---
 版权声明：尊重原创，转载请注明出处 [ https://blog.csdn.net/abc123lzf/article/details/82381719]( https://blog.csdn.net/abc123lzf/article/details/82381719)   
  ### []()一、引言

 `java.lang.ref.Reference` 类表示引用类型，于JDK1.2被引入，其 `public` 权限子类有 `SoftReference` （软引用）， `WeakReference` （弱引用）， `PhantomReference` （虚引用，又称幽灵引用）。

 回顾一下Java引用方面的基础知识：  
 1、如果一个对象没有任何引用了，那么这个对象会在GC的时候被回收。在我们平时写程序时大部分都是使用强引用：

 
```
Object obj = new Object();

```
 其中 `obj` 就是一个强引用。

 2、只要强引用还存在，垃圾回收器永远不会回收这个对象。对于软引用，在系统将要面临内存溢出异常前，才会把这个引用的对象进行回收。对于弱引用，其关联的对象只能生存到下一次垃圾收集之前（除非有其它强引用或软引用引用它）。对于幽灵引用，无论如何都无法通过幽灵引用来取得这个对象，它唯一的作用就是能在这个对象被回收时在引用队列中得到。

 3、 `ReferenceQueue` ，即引用队列，当一个 `Reference` 引用的对象被垃圾回收后，这个 `Reference` 会被加入到这个队列（前提是在构造 `Reference` 时声明了这个队列）。

 
--------
 
### []()二、Reference源码分析

 `java.lang.ref.Reference` 是一个泛型抽象类，其泛型参数T代表引用的对象类型  
  `Reference` 对象提供了两个构造方法：

 
```
Reference(T referent) {
    this(referent, null);
}

Reference(T referent, ReferenceQueue<? super T> queue) {
	this.referent = referent;
    this.queue = (queue == null) ? ReferenceQueue.NULL : queue;
}

```
 `reference` 表示需要被引用的对象， `queue` 表示引用队列。  
 如果指定了引用队列，那么当这个引用的对象被回收后，这个 `Reference` 对象本身会被加入到指定的 `ReferenceQueue` 。  
 另外， `PhantomReference` 只提供了第二种参数类型构造方法。

 `public` 方法：

 
```
public T get(); //获取引用的对象
public void clear(); //清除这个引用的对象，并不会导致这个Reference加入到引用队列
public boolean isEnqueued(); //返回这个对象是否已经入队
public boolean enqueue(); //将这个Reference加入到引用队列，返回是否加入成功

```
 这些方法的实现比较简单，就不再分析了。

 **在Reference内部实现中，引用的对象有4种状态，分别是： `active` 、 `pending` 、 `enqueued` 、 `inactive` **  
 下面附上源码对这些状态的注释和我个人对这几个状态的理解：  
 **1、active状态：**  
 Subject to special treatment by the garbage collector. Some time after the collector detects that the reachability of the referent has changed to the appropriate state, it changes the instance’s state to either Pending or Inactive, depending upon whether or not the instance was registered with a queue when it was created. In the former case it also adds the instance to the pending-Reference list. Newly-created instances are Active.  
  `Reference` 刚开始被构造时处于这个状态。当对象的可达性发生改变（不再可达）的某个时间后，会被更改为 `pending` 状态（前提是构造 `Reference` 对象时传入了引用队列）。  
 **2、pending状态：**  
 An element of the pending-Reference list, waiting to be enqueued by the Reference-handler thread. Unregistered instances  
 are never in this state.  
 处于这个状态时，说明引用列表即将被 `ReferenceHandler` 线程加入到引用队列的对象（前提是构造 `Reference` 对象时传入了引用队列）。  
 **3、enqueued状态：**  
 An element of the queue with which the instance was registered when it was created. When an instance is removed from its ReferenceQueue, it is made Inactive. Unregistered instances are never in this state.  
 这个引用的对象即将被垃圾回收时处于这个状态，此时已经被 `JVM` 加入到了引用队列（如果构造时指定的话），当从引用队列中取出时，状态随之变为 `inactive` 状态。  
 **4、inactive状态：**  
 Nothing more to do. Once an instance becomes Inactive its state will never change again.  
 引用的对象已经被垃圾回收，一旦处于这个状态就无法改变了。

 这些状态都是和JVM密切相关的。

 下面来看 `Reference` 类的实例变量和静态变量：

 
```
public abstract class Reference<T> {
	//引用的对象，由垃圾回收器控制其引用
	private T referent;
	
	//所属的引用队列，如果指定了引用队列并且没有入队，那么queue指向的就是指定的引用队列，如果没有指定
	//引用队列，则默认为ReferenceQueue.NULL，如果入队，则queue改为Reference.QUEUE,所以这个变量
	//既可以保存指定的引用队列，也可以作为一个标记判断这个Reference有没有入队
	//设置为volatile是因为有ReferenceHandler线程负责入队操作，即会更改这个queue指向ENQUEUED
	volatile ReferenceQueue<? super T> queue;
	
	//引用队列中下一个Reference对象，ReferenceQueue通过这个来维持队列的顺序
	@SuppressWarnings("rawtypes")
    Reference next;
    //由JVM控制，表示下一个要被回收的对象
    transient private Reference<T> discovered;
    
	//私有锁
	static private class Lock { }
    private static Lock lock = new Lock();
    
	//引用列表中下一个要进入引用队列的对象(这个引用的对象已经被GC)，由ReferenceHandler线程负责加入队列
	//pending由JVM赋值
	private static Reference<Object> pending = null;
}

```
 `Reference next` ：引用队列中的下一个 `Reference` 对象，只有当入了队后才会赋值（如果构造没有声明引用队列则始终为null）  
 当对象状态为 `active` 时， `next` 为 `null` 。  
 当状态为 `pending` 时， `next == this` 。  
 当状态为 `enqueued` 时， `next` 为队列中的下一个 `Reference` 对象。  
 当状态为 `inactive` 后， `next==this` 。

 `Reference discovered` ：垃圾收集器管理的引用列表中下一个引用对象，由 `JVM` 管理和赋值。  
  `active` 状态：引用列表中的下一个对象，如果 `this` 是最后一个则 `discovered == this`   
  `pending` 状态：下一个引用对象，如果 `this` 是最后一个则为 `null`   
 其它状态： `null` 。

 `Reference` 类的静态初始化块：

 
```
 static {
	//获取当前线程的线程组
	ThreadGroup tg = Thread.currentThread().getThreadGroup();
	//找到最顶层的线程组并赋给变量tg
    for (ThreadGroup tgn = tg; tgn != null; tg = tgn, tgn = tg.getParent());
    //创建ReferenceHandler线程负责操作引用队列
    Thread handler = new ReferenceHandler(tg, "Reference Handler");
    //设置线程优先级为最高
    handler.setPriority(Thread.MAX_PRIORITY);
    //设置为守护线程(后台线程)
    handler.setDaemon(true);
    handler.start();
	//检查java.lang.ref包的访问权限？
    SharedSecrets.setJavaLangRefAccess(new JavaLangRefAccess() {
	    @Override
        public boolean tryHandlePendingReference() {
            return tryHandlePending(false);
        }
    });
}

```
 静态初始化块主要的工作就是启动 `ReferenceHandler` 线程。

 
```
private static class ReferenceHandler extends Thread {
	private static void ensureClassInitialized(Class<?> clazz) {
	    try {
            Class.forName(clazz.getName(), true, clazz.getClassLoader());
        } catch (ClassNotFoundException e) {
            throw (Error) new NoClassDefFoundError(e.getMessage()).initCause(e);
        }
    }
    static {
        ensureClassInitialized(InterruptedException.class);
        ensureClassInitialized(Cleaner.class);
    }

    ReferenceHandler(ThreadGroup g, String name) {
        super(g, name);
    }

    public void run() {
        while (true)
            tryHandlePending(true);
    }
}

```
 在加载类 `ReferenceHandler` 时，会首先尝试加载 `InterruptedException` 和 `Cleaner` 类，如果加载不成功则抛出 `NoClassDefFoundError` 错误（一般不会发生）。  
 可以看到，线程 `ReferenceHandler` 总是在不停的调用 `tryHandlePending` 方法

 
```
static boolean tryHandlePending(boolean waitForNotify) {
    Reference<Object> r;
    Cleaner c; //Claner隶属sun.misc包，是JDK内部提供的用来释放非堆内存资源的API
    try {
        synchronized (lock) {
	        //如果pending为null，则一直等待到pending赋值(由JVM负责notify或interrupt)
            if (pending != null) {
                r = pending;
                //这里可能会抛出内存溢出异常
                c = r instanceof Cleaner ? (Cleaner) r : null;
                pending = r.discovered;
                r.discovered = null;
            } else {
				//这里同样可能会发生内存溢出
                if (waitForNotify)
                    lock.wait();
                return waitForNotify;
            }
        }
    } catch (OutOfMemoryError x) {
	    //让步给其它线程，它们可能会清除一些对象
        Thread.yield();
        return true;
    } catch (InterruptedException x) {
        return true;
    }

    if (c != null) {
	    //调用Clean实例的clean方法清理堆外内存
        c.clean();
        return true;
    }
	//获取pending的引用队列，如果构造时指定了引用队列，并将q入队
    ReferenceQueue<? super Object> q = r.queue;
    if (q != ReferenceQueue.NULL) 
	    q.enqueue(r);
    return true;
}

```
 这个方法的任务就是将失去对象的Reference对象加入到所属的引用队列中。

 
--------
 
### []()三、ReferenceQueue源码解析

 `ReferenceQueue` 事实上是通过 `Reference` 对象本身来维持队列的。

 
```
public class ReferenceQueue<T> {
	private static class Null<S> extends ReferenceQueue<S> {
        boolean enqueue(Reference<? extends S> r) {
            return false;
        }
    }
    //当Reference入队前，其成员变量queue为NULL，入队后，其成员变量为ENQUEUED
    static ReferenceQueue<Object> NULL = new Null<>();
    static ReferenceQueue<Object> ENQUEUED = new Null<>();

    static private class Lock { };
    private Lock lock = new Lock();
    //队列的首部
    private volatile Reference<? extends T> head = null;
    //队列的长度
    private long queueLength = 0;
	
	public ReferenceQueue(){ }
	//入队，由Reference的ReferenceHandler线程调用
	boolean enqueue(Reference<? extends T> r) {
		synchronized (lock) {
			//获取r的queue属性，检查有没有指定引用队列或已经入队或出队
            ReferenceQueue<?> queue = r.queue;
            if ((queue == NULL) || (queue == ENQUEUED))
                return false;
            assert queue == this;
            //标记为已经入队
            r.queue = ENQUEUED;
            //如果head为null，则将这个Reference作为head
            r.next = (head == null) ? r : head;
            //将r作为head，实际上是一个栈?
            head = r;
            //队列长度加1
            queueLength++;
            if (r instanceof FinalReference)
                sun.misc.VM.addFinalRefCount(1);
            //唤醒调用remove的线程
            lock.notifyAll();
            return true;
        }
	}
	
	//出队操作，如果没有元素直接返回null
	public Reference<? extends T> poll() {
        if (head == null)
            return null;
        synchronized (lock) {
            return reallyPoll();
        }
    }
    
	private Reference<? extends T> reallyPoll() {
		Reference<? extends T> r = head;
        if (r != null) {
	        //将队列中下一个Reference赋给head
            head = (r.next == r) ? null : r.next;
            //标记为NULL表示已出队
            r.queue = NULL;
            //将next设为r(即自身)
            r.next = r;
            //队列长度减1
            queueLength--;
            if (r instanceof FinalReference)
                sun.misc.VM.addFinalRefCount(-1);
            return r;
        }
        return null;
	}
	//出队，如果没有元素该线程会阻塞到有元素返回
	public Reference<? extends T> remove() throws InterruptedException {
        return remove(0);
    }
	//出队，如果没有元素那么最多等待timeOut毫秒直到超时或者有元素入队
	public Reference<? extends T> remove(long timeout)
	        throws IllegalArgumentException, InterruptedException {
        if (timeout < 0)
            throw new IllegalArgumentException("Negative timeout value");
        synchronized (lock) {
            Reference<? extends T> r = reallyPoll();
            if (r != null) 
	            return r;
            long start = (timeout == 0) ? 0 : System.nanoTime();
            for (;;) {
	            //最多等待timeOut毫秒
                lock.wait(timeout);
                r = reallyPoll();
                if (r != null) 
	                return r;
                if (timeout != 0) {
                    long end = System.nanoTime();
                    timeout -= (end - start) / 1000_000;
                    if (timeout <= 0) 
	                    return null;
                    start = end;
                }
            }
        }
    }
}

```
 代码比较简单，就不详细解释了

   
  