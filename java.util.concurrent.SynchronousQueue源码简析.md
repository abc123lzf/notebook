---
title: java.util.concurrent.SynchronousQueue源码简析
date: 2018-09-23 16:09:20
tags: CSDN迁移
---
 版权声明：尊重原创，转载请注明出处 [ https://blog.csdn.net/abc123lzf/article/details/82792766]( https://blog.csdn.net/abc123lzf/article/details/82792766)   
  ### []()一、引言

 SynchronousQueue是一个比较特殊的阻塞队列实现类，它实际上并不会给元素维护一系列存储空间，它维护了一组线程，当尝试调用添加元素的方法时，必须要有另外一个线程尝试取出这些元素，否则就无法将元素"添加"进去。在生产-消费者模型中，这样做的好处是可以降低从生产者到消费者中间的延迟，而在LinkedBlockingQueue或者ArrayBlockingQueue中，生产者必须将元素插入到其中存储元素的结构中然后才能转交给消费者。这个类在Java并发工具包中用作CacheThreadPool的任务队列：

 
```
public static ExecutorService newCachedThreadPool() {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE, 60L, TimeUnit.SECONDS,
                      new SynchronousQueue<Runnable>());
}

```
 简单地来说，SynchronousQueue有以下几大特性：  
 1、SynchronousQueue和传统的集合类不同，它的内部没有恒定的存储元素的空间。每一个添加元素的操作都必须要有另外一个线程进行取出操作，否则添加元素操作不会成功，反之亦然。  
 2、因为内部没有恒定的存储元素的空间，所以SynchronousQueue不支持调用isEmpty、size、remainingCapacity、clear、contains、remove、toArray等方法，也不支持元素的迭代操作。  
 3、和ReentrantLock类似，它支持公平和非公平模式。

 **SynchronousQueue的自旋操作比较复杂，其实我个人也没看的太懂，也只明白个大致的思想，所以这篇博客仅供参考**

 
### []()二、源码分析

 SynchronousQueue提供了两个public构造方法，其方法签名和作用如下：

 
     方法签名                             | 解释                                        
     -------------------------------- | ------------------------------------------ 
     public SynchronousQueue()        | 创建一个非公平模式的阻塞队列                            
     public SynchronousQueue(boolean) | 当参数为true时，会创建一个公平模式的阻塞队列，为false时效果和无参构造器相同


```
public SynchronousQueue() {
	this(false);
}
public SynchronousQueue(boolean fair) {
	transferer = fair ? new TransferQueue<E>() : new TransferStack<E>();
}

```
 SynchronousQueue通过两个内部类TransferQueue和TransferStack来实现消费者的公平性，TransferQueue底层为队列实现，用来实现公平策略，而TransferStack底层为栈实现，用来实现非公平策略，它们都继承了内部抽象类Transferer。  
 在公平策略下，能够保证等待最久的消费者线程能够优先拿到元素或者等待最久的生产者线程能够优先转交元素。

 
```
abstract static class Transferer<E> {
	abstract E transfer(E e, boolean timed, long nanos);
}

```
 Transferer类泛型参数E为队列元素类型，它只提供了一个抽象方法：transfer方法，在SynchronousQueue类中，所有的入队出队操作都是基于这个方法实现的，其参数e为"插入"的元素引用，即向消费者转交元素（当e为null时，表示"取出"元素，即尝试从生产者取出元素），timed用于指示该操作是否有最大阻塞的时间，nanos表示最大阻塞时间。

 下面是SynchronousQueue的主要成员变量和静态常量

 
```
public class SynchronousQueue<E> extends AbstractQueue<E>
    	implements BlockingQueue<E>, java.io.Serializable {
    //获取CPU核心数
    static final int NCPUS = Runtime.getRuntime().availableProcessors();
    //在有阻塞时间的线程等待最大自旋次数，单核CPU为0，多核为32
    static final int maxTimedSpins = (NCPUS < 2) ? 0 : 32;
    //在无阻塞时间的线程等待最大自旋次数
    static final int maxUntimedSpins = maxTimedSpins * 16;
    static final long spinForTimeoutThreshold = 1000L;
    //队列功能实现类
    private transient volatile Transferer<E> transferer;
}

```
 作为生产者，如果希望向消费者线程转交元素，可以调用put、offer方法。

 
```
public void put(E e) throws InterruptedException {
	if (e == null) throw new NullPointerException();
	//如果没有成功地添加元素，则将当前线程中断并抛出中断异常
    if (transferer.transfer(e, false, 0) == null) {
        Thread.interrupted();
        throw new InterruptedException();
    }
}

```
 put方法会尝试添加元素，如果没有添加成功，则抛出InterruptedException  
 offer方法有两个重载的方法：boolean offer(E)和boolean offer(E, long, TimeUnit)  
 前者会以非阻塞的方式向消费者线程转交元素，如果添加失败，则直接返回false。后者则可以指定最长的阻塞时间。

 
```
public boolean offer(E e) {
    if (e == null) throw new NullPointerException();
    return transferer.transfer(e, true, 0) != null;
}

public boolean offer(E e, long timeout, TimeUnit unit) throws InterruptedException {
    if (e == null) throw new NullPointerException();
    //如果在指定时间内转交成功，返回true
    if (transferer.transfer(e, true, unit.toNanos(timeout)) != null)
        return true;
    //如果线程已被中断，返回false，否则抛出异常
    if (!Thread.interrupted())
        return false;
    throw new InterruptedException();
}

```
 作为消费者，如果希望从生产者线程获取元素，则可以调用poll、take方法  
 take方法以阻塞的方式让当前线程获取元素，直到从生产者线程拿到元素或者线程被中断。

 
```
public E take() throws InterruptedException {
    E e = transferer.transfer(null, false, 0);
   	//如果从生产者手中拿到了元素，则返回
    if (e != null)
        return e;
    Thread.interrupted();
    throw new InterruptedException();
}

```
 poll方法同样有两个重载的方法，其作用和上述offer类似，不再过多解释了。

 
```
public E poll() {
    return transferer.transfer(null, true, 0);
}

public E poll(long timeout, TimeUnit unit) throws InterruptedException {
    E e = transferer.transfer(null, true, unit.toNanos(timeout));
    if (e != null || !Thread.interrupted())
        return e;
    throw new InterruptedException();
}

```
 其实可以发现，这些入队、出队操作都是围绕这transfer方法进行的，其核心实现原理都是围绕Transferer类的，下面我们依次来分析内部类TransferQueue、TransferStack。

 
##### []()1、TransferQueue

 TransferQueue是SynchronousQueue的公平策略实现类，它在内部维护了一个线程队列。

 
```
static final class TransferQueue<E> extends Transferer<E> {
	//队列头结点
	transient volatile QNode head;
	//队列尾结点
	transient volatile QNode tail;
	//该引用指向尚未从队列中移除的被取消的结点
	transient volatile QNode cleanMe;
	
	TransferQueue() {
		//将一个空的结点作为头结点和尾结点
        QNode h = new QNode(null, false);
        head = h;
        tail = h;
    }
	//省略其它方法...
}

```
 对于TransferQueue，其结点都是的实现类TransferQueue的内部类QNode：

 
```
static final class QNode {
	//后驱指针
    volatile QNode next;
    //需要转交的对象引用，当item==this时，说明waiter已被interrupt
    volatile Object item;
    //调用transfer的线程对象
    volatile Thread waiter;
    //用来判断是数据结点还是请求结点
    final boolean isData;

	QNode(Object item, boolean isData) {
        this.item = item;
        this.isData = isData;
    }
}

```
 QNode分为数据结点和请求结点两大类型：数据结点由生产者线程构造，请求结点由消费者线程构造。  
 初步了解这些类的作用和成员变量后，我们回过头来分析TransferQueue的transfer方法实现：

 
```
@SuppressWarnings("unchecked")
E transfer(E e, boolean timed, long nanos) {
	//临时变量，一般指向新构建的结点
	QNode s = null;
    boolean isData = (e != null);
	//线程进入自旋操作
    for (;;) {
    	//获取队列的头结点和尾结点
    	QNode t = tail;
        QNode h = head;
        //如果任意一个为null，则重新开始循环
        if (t == null || h == null)
        	continue;
		//如果队列为空或者模式相同（例如尾结点为数据结点，并且当前线程也是生产者线程）
        if (h == t || t.isData == isData) {
        	//访问尾结点的下一个结点
            QNode tn = t.next;
            //如果下一个结点不是尾结点则重新开始循环
            if (t != tail)
                continue;
            //如果下一个结点不为null，说明有新的入队元素
            if (tn != null) {
            	//通过CAS操作更新尾结点，重新开始循环
                advanceTail(t, tn);
                continue;
            }
            //如果设置了阻塞时间并且为0秒，则返回null
            if (timed && nanos <= 0)
                return null;
            //如果变量为s，则构造一个结点
            if (s == null)
                s = new QNode(e, isData);
            //调用CAS将t的后驱引用由null变为指向s，即将元素入队，如果失败则重新开始循环
            if (!t.casNext(null, s))
                continue;
			//成功入队后，更新队列的tail尾结点所指向的对象
            advanceTail(t, s);
            //等待元素被成功转交
            Object x = awaitFulfill(s, e, timed, nanos);
            //如果返回的就是s，那么转交失败，调用clean清除这个结点
            if (x == s) {
                clean(t, s);
                return null;
            }
			//如果s没有出队
            if (!s.isOffList()) {
            	//通过CAS将队列的头结点(同时也是尾结点)更新为新的结点
                advanceHead(t, s);
                if (x != null)
                    s.item = s;
                s.waiter = null;
            }
            return (x != null) ? (E)x : e;
        } else { 
        	//获取头结点的下一个结点
            QNode m = h.next;
            //如果此时有新的结点入队或者头结点被更新，那么重新开始循环
            if (t != tail || m == null || h != head)
                continue;
			//获取头结点的下一个结点的对象引用
            Object x = m.item;
            //如果该节点是数据结点
            if (isData == (x != null) || x == m || !m.casItem(x, e)) { 
            	//将头结点更新为头结点的下一个结点，重新开始循环
                advanceHead(h, m);
                continue;
            }
			//将头结点更新为头结点的下一个结点，并唤醒m中等待的线程
            advanceHead(h, m); 
            LockSupport.unpark(m.waiter);
            //返回结果
            return (x != null) ? (E)x : e;
        }
    }
 }

```
 transfer方法是一个无锁的线程安全方法，它通过线程的自旋操作来保证其线程安全性。该方法归纳起来主要任务为：  
 1、如果此时队列为空（仅存在一个头结点），或者队列元素是数据结点并且当前线程也是尝试添加数据结点（或者队列元素是请求结点并且当前线程也是尝试获取数据，这里称其为模式相同），那么将当前线程对象、数据（如果有的话）及其结点类型插入到队列尾端，并阻塞当前线程等待被唤醒或者被interrupt。该方法大量的if判断和CAS操作是为了保证无锁条件下的线程安全。  
 2、否则，则说明队列不为空并且模式不同。该方法会尝试唤醒队列头部等待的线程并将其出队。

 这些CAS操作和很多Java并发工具包中的类一样，都是通过调用sun.misc.Unsafe类的CAS操作实现的。

 下面我们来分析将线程阻塞的方法：awaitFulfill

 
```
Object awaitFulfill(QNode s, E e, boolean timed, long nanos) {
	//超时时间戳
	final long deadline = timed ? System.nanoTime() + nanos : 0L;
	//获取当前线程
	Thread w = Thread.currentThread();
	//获取下面循环操作的自旋次数
    int spins = ((head.next == s) ? (timed ? maxTimedSpins : maxUntimedSpins) : 0);
    for (;;) {
    	//如果当前线程被中断，那么尝试将s的数据引用由e指向自身
    	if (w.isInterrupted())
        	s.tryCancel(e);
        //获取s的数据结点
        Object x = s.item;
        //如果s的数据引用不是e，说明线程被中断，操作失败，直接返回x
        if (x != e)
            return x;
        //如果设定了阻塞时间
        if (timed) {
            nanos = deadline - System.nanoTime();
            //如果超过了阻塞时间，那么将结点s数据引用由e指向自身，重新开始循环
            if (nanos <= 0L) {
                s.tryCancel(e);
                continue;
            }
        }
        //如果自旋次数没有达到则重新开始循环
        if (spins > 0)
            --spins;
        //如果结点s的线程对象为null，那么将其设为当前线程
        else if (s.waiter == null)
            s.waiter = w;
       //阻塞当前线程，进行等待
        else if (!timed)
            LockSupport.park(this);
        else if (nanos > spinForTimeoutThreshold)
            LockSupport.parkNanos(this, nanos);
    }
}

```
 awaitFulfill方法需要传入4个参数：结点引用s，数据引用e（一般情况下s的数据结点就是e）、是否有阻塞超时时间及其阻塞时间。  
 关于自旋次数spins，其目的在于不断计算有没有超时或者线程有没有interrupt（单核CPU下，不会发生自旋操作。在多核情况下，如果指定了超时时间，那么自旋次数为32次，如果没有指定则为512次）。自旋操作完成后，该方法会调用LockSupport的park方法阻塞当前线程进行等待。

 如果该方法调用的线程被interrupt或者超时，那么transfer方法会调用clean方法清除这个无用的结点：

 
```
//pred为s的前面一个结点
void clean(QNode pred, QNode s) {
	//清除线程引用
	s.waiter = null;
    while (pred.next == s) {
    	QNode h = head;
        QNode hn = h.next;
        //如果头结点的下一个结点被取消
        if (hn != null && hn.isCancelled()) {
        	//将头结点更新为hn，重新开始循环
        	advanceHead(h, hn);
            continue;
        }
        QNode t = tail;
        //如果队列此时为空，退出方法
        if (t == h)
        	return;
       	//如果尾结点被其它线程更新，重新开始循环
        QNode tn = t.next;
        if (t != tail)
            continue;
        //如果有新入队的元素并且尾结点未被更新，那么更新尾结点，重新开始循环
        if (tn != null) {
            advanceTail(t, tn);
            continue;
        }
        //如果结点s不是尾结点
        if (s != t) {
        	//获取结点s的下一个结点
        	QNode sn = s.next;
        	//如果下一个结点
            if (sn == s || pred.casNext(s, sn))
            	return;
        }
        //获取最后一个被取消的结点
        QNode dp = cleanMe;
        //如果这个被取消的结点不为null
        if (dp != null) {
        	QNode d = dp.next;
            QNode dn;
            if (d == null || d == dp || !d.isCancelled() ||
            	(d != t &&  (dn = d.next) != null &&  dn != d && dp.casNext(d, dn)))
            	//清除这个结点
            	casCleanMe(dp, null);
          	//如果这个结点就是pred，
            if (dp == pred)
                return;
        //否则将cleanMe设为pred
        } else if (casCleanMe(null, pred))
            return;
    }
}

```
 
##### []()2、TransferStack

 TransferStack是SynchronousQueue的非公平策略实现类，它在内部维护了一个线程栈。

 
```
static final class TransferStack<E> extends Transferer<E> {
	//代表请求结点
	static final int REQUEST = 0;
	//代表数据结点
	static final int DATA = 1;
	//结点正在交接数据
	static final int FULFILLING = 2;
	//头结点
	volatile SNode head;
}

```
 TransferStack类仅有一个实例变量：head，类型为SNode，SNode是TransferStack的内部类，它的实例代表一个节点。

 
```
static final class SNode {
	//后驱结点引用
	volatile SNode next;
	//匹配结点引用
	volatile SNode match;
	//等待的线程对象
	volatile Thread waiter;
	//数据对象的引用(如果该结点为请求结点则为null)
	Object item;
	//结点的类型和状态
	int mode;
	
	SNode(Object item) {
    	this.item = item;
    }
}

```
 其中，mode的第1位代表该结点是数据结点还是请求结点（0为请求结点，1为数据结点），第2位代表结点的状态（1代表该结点正在交接数据）。  
 下面是TransferStack的transfer方法实现：

 
```
@SuppressWarnings("unchecked")
E transfer(E e, boolean timed, long nanos) {
	SNode s = null;
	//根据e是否为null判断属于消费者还是生产者
	int mode = (e == null) ? REQUEST : DATA;
	//线程进入自旋
	for (;;) {
        SNode h = head;
        //如果栈为空或者模式相同
        if (h == null || h.mode == mode) {
        	//如果设置了阻塞时间并且时间为0
            if (timed && nanos <= 0) {
            	//如果头结点不为null并且头结点已被取消，那么将头结点的下一个元素作为新的头结点（即使下一个结点为null）
                if (h != null && h.isCancelled())
                    casHead(h, h.next);
                //否则直接返回null
                else
                    return null;
            //构造一个新的结点，通过CAS将其设为头结点，如果设置成功则继续执行
            } else if (casHead(h, s = snode(s, e, h, mode))) {
            	//调用awaitFulfill方法阻塞当前线程
                SNode m = awaitFulfill(s, timed, nanos);
                //如果返回的就是s，那么操作失败，返回null
                if (m == s) {
                    clean(s);
                    return null;
                }
                //如果栈不为空，并且头结点的下一个结点为s，那么从栈中弹出头结点
                if ((h = head) != null && h.next == s)
                    casHead(h, s.next);
                //如果本次操作是请求元素，那么返回结点m上的数据，否则返回s上的数据
                return (E) ((mode == REQUEST) ? m.item : s.item);
            }
        //如果栈不为空并且模式不同
        } else if (!isFulfilling(h.mode)) {
        	//如果这个头结点已被取消，那么通过CAS重设头结点为头结点的下一个结点
            if (h.isCancelled())
                casHead(h, h.next);
            //构造一个新的结点，通过CAS将头结点设为这个结点（将其压入栈），如果成功继续执行
            else if (casHead(h, s=snode(s, e, h, FULFILLING|mode))) {
                for (;;) {
                	//获取这个新结点的下一个结点
                	SNode m = s.next;
                	//如果下一个结点为null，那么将头结点设为null，并清除这个新结点，跳出循环
                    if (m == null) {
                        casHead(s, null); 
                        s = null;
                        break;
                    }
                    SNode mn = m.next;
                    //尝试匹配
                    if (m.tryMatch(s)) {
                    	//从栈中弹出两个节点，即s和m结点
                        casHead(s, mn);
                        return (E) ((mode == REQUEST) ? m.item : s.item);
                    //若匹配失败，说明有别的线程已经匹配成功
                    } else
                        s.casNext(m, mn);
                }
            }
        //如果上述条件都不满足，那么这里一般为结点正在匹配阶段（即结点的mode变量的第2位为1）
        } else {
            SNode m = h.next;
            //如果头结点的下一个结点为null，那么弹出头结点
            if (m == null) 
                casHead(h, null); 
            else {
            	//继续获取下一个结点
                SNode mn = m.next;
                //将m和h匹配，匹配成功同样弹出两个节点，即s和m结点
                if (m.tryMatch(h))
                    casHead(h, mn);
                //否则弹出第二个节点
                else
                    h.casNext(m, mn);
            }
        }
    }
}

```
 总结一下，transfer方法在自旋操作中有3种情况：  
 1、如果栈为空或者模式相同（类似TransferQueue），那么尝试构造一个新的结点并压入栈，然后让当前线程等待元素被交接或者被interrupt。  
 2、如果栈不为空，并且模式不同，则将当前结点的mode变量加上正在匹配的标记，压入栈中，与原栈顶元素进行匹配，匹配成功后，从栈中弹出这两个结点。  
 3、如果有结点正在匹配，那么将这个结点进行匹配，匹配成功后弹出两个节点，继续开始下一次循环。

 tryMatch方法是SNode的非静态方法：

 
```
boolean tryMatch(SNode s) {
	//如果成员变量match为null，那么通过CAS将match更新为s，更新成功后继续执行
	if (match == null && UNSAFE.compareAndSwapObject(this, matchOffset, null, s)) {
		Thread w = waiter;
		//如果waiter变量不为null，那么唤醒这个线程并将waiter引用指向null
        if (w != null) {
        	waiter = null;
            LockSupport.unpark(w);
        }
        //匹配成功
        return true;
    }
    //否则判断match是不是为s
    return match == s;
}

```
 tryMatch方法主要任务是唤醒这个结点上等待的线程，通过CAS算法保证只有一个线程能将match引用设为s，来保证在无锁条件下的线程安全性。

   
  