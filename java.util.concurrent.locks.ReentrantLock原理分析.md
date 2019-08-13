---
title: java.util.concurrent.locks.ReentrantLock原理分析
date: 2018-09-10 21:46:17
tags: CSDN迁移
---
 版权声明：尊重原创，转载请注明出处 [ https://blog.csdn.net/abc123lzf/article/details/82495354]( https://blog.csdn.net/abc123lzf/article/details/82495354)   
  ### []()一、引言

 ReentrantLock是JDK1.5引入的一种工具类：它可以实现可重入锁的功能。在分析它的原理之前，我们先来回顾下基础知识：

 **什么是可重入锁？**  
 可重入锁是一种可重复可递归调用的锁，在一个线程获取这个锁后，这个线程依然可以再次获得这个锁，而其它线程要想获得锁只能等到它全部释放之后。  
 比如：

 
```
public class Test {
	static final ReentrantLock lock = new ReentrantLock();
	static class Mission0 implements Runnable {
		@Override
		public void run() {
			lock.lock();
			lock.lock();
			System.out.println("Mission0");
			lock.unlock();
		}
	}	
	static class Mission1 implements Runnable {
		@Override
		public void run() {
			lock.lock();
			System.out.println("Mission1");
		}
	}	
	public static void main(String[] args) throws InterruptedException {
		new Thread(new Mission0()).start();
		Thread.sleep(500);
		new Thread(new Mission1()).start();
	}
}

```
 上述代码只会输出Mission0，不会输出Mission1，除非在Mission0类中再加一个lock.unlock()，Mission1才能获得到锁，并输出Mission1。

 在Java中，synchronized关键字也可用来实现可重入锁。在Java6以前，ReentrantLock的性能远好于synchronized关键字，在Java6以后JVM对synchronized关键字进行了大幅度的优化，现在两者性能几乎差不多。  
 另外，ReentrantLock大部分是纯Java代码实现的，而synchronized是由JVM内部实现的。

 **什么是公平锁？什么是非公平锁？**  
 对于公平锁，如果有多个线程等待一个锁的释放，那么这个锁释放的时候最早进行等待的线程优先获得该锁。对于非公平锁而言，如果同样有多个线程等待一个锁的释放，那么这个锁释放的时候会“随机”由一个线程获得该锁。显然，非公平锁的性能要好于公平锁。

 ReentrantLock可以实现公平锁也可实现非公平锁，而synchronized只可实现非公平锁。

 **什么是CAS算法？**  
 CAS，即CompareAndSwap（比较并交换）。CAS操作需要3个数：需要读写的内存地址V、进行比较的值A、打算写入的新值B。当且仅当V的值等于A时，CAS才会原子地用新值B来更新V的值，否则不会执行任何操作。大多数CPU指令集都实现了对CAS的底层支持。

 下面是ReentrantLock类的结构图：  
 ![这里写图片描述](https://img-blog.csdn.net/20180907192528897?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FiYzEyM2x6Zg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)  
 ReentrantLock继承了Lock接口，其主要功能由内部类NofairSync或FairSync实现，分别代表非公平锁和公平锁。

 **在了解ReentrantLock的实现机制前，你需要对AbstractQueuedSynchronizer类的实现原理有一个大致的理解，可以参考我的博客：**[点击进入](https://blog.csdn.net/abc123lzf/article/details/82532036)

 **ReentrantLock的API简介：**

 
```
//使当前线程获得锁，如果没有获得到则一直阻塞到获得锁为止
public void lock();
//使当前线程获得锁，如果没有获得到则一直阻塞，但是可以由别的线程调用该线程对象interrupt中断
public void lockInterruptibly() throws InterruptedException;
//尝试获得锁并返回是否成功，如果没有获得到锁当前线程不会阻塞
public boolean tryLock();
//尝试获得锁，如果成功获得立刻返回true，没有获得则最多等待timeout,直到获得到锁或超时
public boolean tryLock(long timeout, TimeUnit unit) throws InterruptedException;
//释放锁
public void unlock();
//创建一个Condition
public Condition newCondition();
//当前线程加锁的次数(调用lock次数减去unlock次数)
public int getHoldCount();
//返回这个ReentrantLock是否是当前线程构造的
public boolean isHeldByCurrentThread();
//返回当前线程是否持有锁
public boolean isLocked();
//是否是公平锁
public final boolean isFair();
//是否有线程等待获得锁
public final boolean hasQueuedThreads();
//这个线程是否在等待锁的队列中
public final boolean hasQueuedThread(Thread thread);
//获得等待获得锁的线程队列长度
public final int getQueueLength();
//判断Condition是否有等待线程
public boolean hasWaiters(Condition condition);
//根据Condition获取等待队列长度
public int getWaitQueueLength(Condition condition);

```
 
--------
 ### 二、原理分析 查看ReentrantLock的源码，ReentrantLock仅有一个Sync类型的实例变量，可以看出ReentrantLock中的方法都是基于内部类Sync实现的。 
```
public class ReentrantLock implements Lock, java.io.Serializable {
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
 Sync继承了同步框架AbstractQueuedSynchronizer（AQS），并重写了其中的isHeldExclusively、tryRelease方法。Sync是NofairSync和FairSync的基类，tryAcquire方法由子类NonfairSync和FairSync重写。

 ReentrantLock有两个构造器：public ReentrantLock() 和public ReentrantLock(boolean)，无参构造器构造时默认是非公平锁模式，对于后者可以传入一个boolean参数的构造器而言，当参数为true时，是公平锁模式，参数为false时，是非公平锁模式。

 在ReentrantLock中，AQS的state变量代表锁重入的次数，每调用一次unlock操作，state变量就减去1，直到state为0时，代表这个线程已经释放了这个锁。

 
```
@Override
protected final boolean tryRelease(int releases) {
	//获取AQS的state状态并减1(releases都为1)
	int c = getState() - releases;
	//从AQS获取持有锁的线程，如果不是当前线程则抛出异常(相当于尝试直接调用unlock方法而不是先调用lock)
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();
    //是否应该释放锁
    boolean free = false;
    //如果state已经为0，代表线程已经释放了锁
    if (c == 0) {
	    free = true;
		//将AQS保存持有锁的线程设为null
        setExclusiveOwnerThread(null);
    }
    //更新state值
    setState(c);
    //返回结果
    return free;
}
    
@Override
protected final boolean isHeldExclusively() {
	//如果AQS中持有锁的线程就是当前执行线程，返回true
	return getExclusiveOwnerThread() == Thread.currentThread();
}

```
 调用unlock方法时，tryRelease方法会被调用。

 现在我们来讨论公平锁的实现原理：  
 在ReentrantLock中，公平锁的功能由内部类FairSync实现

 
```
static final class FairSync extends Sync {
	private static final long serialVersionUID = -3000897897090466540L;
	
	//由ReentrantLock的lock方法调用
    final void lock() {
	    //公平锁体现在这里，acquire方法会主动将这个线程加入到
	    //AQS的等待队列中，保证能够按照线程等待顺序获得锁
        acquire(1);
    }
    
    @Override
    protected final boolean tryAcquire(int acquires) {
	    //获取当前线程
        final Thread current = Thread.currentThread();
        //获取AQS中的state变量
        int c = getState();
        //如果state为0，则没有线程持有锁，执行加锁流程
        if (c == 0) {
	        //如果没有其它线程正在等待锁的释放，那么通过CAS操作将state由0更新到1
	        //如果更新失败则获取锁失败(有其它线程抢先一步拿到了锁)，方法返回false
            if (!hasQueuedPredecessors() &&
                compareAndSetState(0, acquires)) {
                //设置持有锁的线程为当前线程
                setExclusiveOwnerThread(current);
                return true;
            }
        //如果有线程持有锁并且属于当前线程，那么进行锁的重入操作
        } else if (current == getExclusiveOwnerThread()) {
	        //将state的值加1
            int nextc = c + acquires;
            //如果state溢出则抛出错误(一般没人会重入这么多次)
            if (nextc < 0)
                throw new Error("Maximum lock count exceeded");
            //更新state变量的值
            setState(nextc);
            return true;
        }
        //获取锁失败返回
        return false;
    }
}

```
 FairSync和NonfairSync的区别体现在lock方法的实现上。FairSync的lock方法直接调用AQS的acquire方法尝试获得锁，如果没有成功获得到锁，则将当前线程加入到AQS的线程等待队列，按照先后顺序来获得锁。

 非公平锁的实现：

 
```
static final class NonfairSync extends Sync {
	private static final long serialVersionUID = 7316153563782823691L;

    final void lock() {
	    //通过CAS将state由0设为1
        if (compareAndSetState(0, 1))
	        //设置成功后将持有锁的线程设置为当前线程
            setExclusiveOwnerThread(Thread.currentThread());
        else
	        //将线程进入等待队列
            acquire(1);
    }

    protected final boolean tryAcquire(int acquires) {
        return nonfairTryAcquire(acquires);
    }
}

```
 ReentrantLock处于非公平锁模式时，调用lock方法后实际上会调用NonfairSync的lock方法。非公平锁lock方法首先会通过CAS将状态尝试从0置为1，如果成功则说明当前线程获得到了锁。否则，将这个线程加入AQS的线程等待队列等待锁的释放。  
 非公平锁之所以非公平在于lock方法首先尝试通过CAS尝试将state由0设为1，并不会尝试加入AQS的等待队列，如果此时AQS等待队列中有线程正好准备获得锁，那么有可能这个线程因为被另外一个线程抢占而不会获得到锁。

 对于tryAcquire方法，默认调用的是父类Sync的nonfairTryAcquire尝试获得锁。

 
```
final boolean nonfairTryAcquire(int acquires) {
	final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
	    //区别在这
	    if (compareAndSetState(0, acquires)) {
	        setExclusiveOwnerThread(current);
            return true;
        }
        
    } else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0)
	        throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    
    return false;
}

```
 咋一看和FairSync的tryAcquire方法很像，只有一个地方不同，就是FairSync类的tryAcquire会在尝试通过CAS将state由0设置为1时必须要判断AQS的等待队列中没有其它线程正在等待锁的释放。而NonfairSync因为是非公平锁，所以无需判断。

   
  