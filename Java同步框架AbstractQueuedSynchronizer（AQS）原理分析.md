---
title: Java同步框架AbstractQueuedSynchronizer（AQS）原理分析
date: 2018-09-16 22:41:58
tags: CSDN迁移
---
 版权声明：尊重原创，转载请注明出处 [ https://blog.csdn.net/abc123lzf/article/details/82532036]( https://blog.csdn.net/abc123lzf/article/details/82532036)   
  **本文参照的是JDK1.8版本的AbstractQueuedSynchronizer源码**

 
### []()一、引言

 AbstractQueuedSynchronizer（以下简称AQS）是Java并发工具包中用于构建锁或其它同步工具的基础框架，其ReentrantLock和Semaphore都是基于这个框架实现的。它的内部维护了一个FIFO队列，在有很多线程竞争资源时，线程对象会被加入到此队列等待。AQS的父类是AbstractOwnableSynchronizer，这个父类比较简单，仅有一个Thread类型的成员变量和它的get、set方法（protected权限），代表一个竞争到资源的线程（一般用于独占资源模式而不是共享资源模式）。

 AQS也可以实现类似于Object的wait和notify来等待、激活线程：AQS有一个内部类ConditionObject，实现了Condition接口，可以调用它的await、signal来实现wait和notify的功能。

 Java并发包有很多工具类采用了这个框架：ReentrantLock、Semaphore、CountDownLatch等，所以了解它的实现原理是十分必要的。

 AQS的代码还是相对来说比较复杂的，建议读者首先大概先过一遍源代码，在学习了解它的原理一定要静下心来慢慢看。

 
--------
 
### []()二、使用方法

 AQS的API：

 
     方法签名                                                                 | 作用                                     
     -------------------------------------------------------------------- | --------------------------------------- 
     void acquire(int)                                                    | 获取独占资源                                 
     void acquireInterruptibly(int) throws InterruptedException           | 获取独占资源，但是如果检测到当前线程被中断则会抛出异常            
     boolean tryAcquireNanos(int, long) throws InterruptedException       | 获取独占资源，可以指定最多阻塞的时间(单位：纳秒)              
     boolean release(int)                                                 | 释放独占资源                                 
     void acquireShared(int)                                              | 获取共享资源                                 
     void acquireSharedInterruptibly(int)throws InterruptedException      | 获取共享资源,但是如果检测到当前线程被中断则会抛出异常            
     boolean tryAcquireSharedNanos(int, long) throws InterruptedException | 获取共享资源，可以指定最多的阻塞时间(单位：纳秒)              
     boolean releaseShared(int)                                           | 释放共享资源                                 
     boolean hasQueuedThreads()                                           | 等待队列中是否有等待的线程                          
     boolean hasContended()                                               | 是否有其它线程竞争资源时失败而被加入到等待队列                
     Thread getFirstQueuedThread()                                        | 获取等待队列头部的线程对象                          
     boolean isQueued(Thread)                                             | 判断指定线程是否在等待队列中                         
     boolean hasQueuedPredecessors()                                      | 判断当前线程是不是在队列首部                         
     int getQueueLength()                                                 | 获取等待队列的长度                              
     Collection&lt;Thread&gt; getQueuedThreads()                          | 将等待队列中所有的线程对象包装为一个集合返回                 
     Collection&lt;Thread&gt; getExclusiveQueuedThreads()                 | 将等待队列中所有尝试获取独占资源的线程对象包装为集合返回           
     Collection&lt;Thread&gt; getSharedQueuedThreads()                    | 将等待队列中所有尝试获取共享资源的线程对象包装为集合返回           
     boolean owns(ConditionObject)                                        | 判断ConditionObject对象是不是属于当前AQS同步器       
     boolean hasWaiters(ConditionObject)                                  | 判断这个ConditionObject对象有没有调用await后正在等待的线程
     int getWaitQueueLength(ConditionObject)                              | 获取这个ConditionObject对象包含的等待线程数量         

AQS作为一个同步框架，提供了5种非final的protected方法用于同步器实现自己的功能：

 
```
//尝试获取独占式资源
protected boolean tryAcquire(int arg);
//尝试释放独占式资源
protected boolean tryRelease(int arg);
//尝试获得共享式资源
protected int tryAcquireShared(int arg);
//尝试释放共享式资源
protected int tryReleaseShared(int arg);
//返回当前线程是否获得了该资源
protected boolean isHeldExclusively();

```
 int参数一般用于调整state变量，其具体意义由同步器的实现类自己定义。

 这些方法由AQS的public方法负责调用（比如tryAcquire方法就会被acquire方法调用），并且在AQS的默认实现都是抛出UnsupportedOperationException异常，需要由子类根据自己的实际业务需求自定义获取资源的策略。比如ReentrantLock内部类Sync就重写了tryAcquire、tryRelease和isHeldExclusively方法，Semaphore作为共享资源同步器，其内部类Sync自然就重写了tryAcquireShared、tryReleaseShared方法。

 
--------
 
### []()三、AQS原理分析

 ![1](https://img-blog.csdn.net/20180909162111655?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FiYzEyM2x6Zg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)  
 AQS内部采用了一个非阻塞式队列（由CAS算法保证线程安全性），一般拥有一个头结点和多个包含等待获取资源的线程对象结点，这些结点有序地排列着，等待着资源的释放然后获取并占有它。另外，ConditionObject也维护了自己的队列。  
 在下文中，我们称AQS维护的等待队列为AQS队列，ConditionObject维护的队列称为Condition队列。

 AQS使用了一个int型的变量state代表资源的状态。比如在ReentrantLock中，state为0时代表没有线程持有锁，state为1时代表有线程持有了锁。对于尝试获得锁的线程而言，会通过CAS算法将其state由0设为1，代表取得了该锁。  
 在AQS中，和很多JDK的并发工具相似，CAS算法由Hotspot虚拟机实现（通过sun.misc.Unsafe类）。

 
##### []()1、Node结点

 AQS通过一个内部类Node来代表一个队列的元素。

 
```
static final class Node {
	//共享资源Node模式
	static final Node SHARED = new Node();
	//独占资源Node模式
	static final Node EXCLUSIVE = null;
    
    static final int CANCELLED =  1;
    static final int SIGNAL    = -1;
    static final int CONDITION = -2;
    static final int PROPAGATE = -3;
    
	//Node状态，对应上面几种，默认为0
    volatile int waitStatus;
	//前驱结点
    volatile Node prev;
	//后驱结点
    volatile Node next;
	//等待的线程
    volatile Thread thread;
    
	//在AQS队列中为等待节点的模式，为SHARED或EXCLUSIVE
	//在Condition队列中为下一个结点
    Node nextWaiter;
}

```
 Node实例保存了维持队列的前驱引用和后驱引用，并保存了等待线程的对象和资源模式。

 
     Node状态    | 解释                                                                           
     --------- | ----------------------------------------------------------------------------- 
     CANCELLED | 当这个Node对应的线程等待超时或被中断，没有获取到资源，处于这个状态的Node会被AQS清除                              
     默认状态      | 当这个Node初始化，或者处于正常等待状态                                                        
     SIGNAL    | 当这个Node对应的线程已经被激活获取到资源时，它会被置于头结点，如果它获取到的资源被释放，那么由它负责通知后继结点获取资源。(只有头结点会处于这个状态)
     CONDITION | 该Node在Condition队列上等待                                                         
     PROPAGATE | 只有头结点才会处于这个状态，表示下一次的共享状态会被无条件的传播下去（用于共享资源的同步器）                               

先记住上述Node状态所对应的情况，Node默认状态为0。

 
##### []()2、acquire方法

 我们从acquire方法开始分析AQS的原理：

 
```
public final void acquire(int arg) {
	if (!tryAcquire(arg) &&
		acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}

```
 acquire方法表示一个线程尝试获取线程独占式资源。

 方法执行步骤如下：  
 1、首先调用tryAcquire尝试获取资源（由子类自己实现获取资源的策略），如果获取成功，该方法结束。  
 2、如果没有获取成功，则首先调用addWaiter方法将当前线程添加到AQS队列中，然后调用acquireQueued方法等待获取资源。如果acquireQueued方法返回false，则该线程是正常获取到资源的，如果为true，则该线程被打断(interrupt)，则调用selfInterrupt中断当前线程。

 用流程图表示如下：  
 ![这里写图片描述](https://img-blog.csdn.net/20180909220336415?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FiYzEyM2x6Zg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

 addWaiter方法，mode表示资源模式（只有两种：Node.EXCLUSIVE表示独占资源，Node.SHARED表示共享资源）：

 
```
private Node addWaiter(Node mode) {
	//将当前线程包装为一个Node结点
	Node node = new Node(Thread.currentThread(), mode);
	//将尾结点引用赋给pred
    Node pred = tail;
    //如果尾结点不为null
    if (pred != null) {
	    //将新Node的前驱指针指向尾结点
        node.prev = pred;
        //利用CAS算法安全地将这个新Node添加到队列尾部
        if (compareAndSetTail(pred, node)) {
	        //将旧尾结点的后驱指针指向新Node
            pred.next = node;
            return node;
        }
    }
	//如果尾结点为null或尝试通过CAS将新Node设为尾结点没有成功，
	//则采用enq方法自旋地将新Node添加到队列尾部
    enq(node);
    return node;
}

private Node enq(final Node node) {
	//反复循环直至添加成功为止
	for (;;) {
		//将尾结点赋值给t
        Node t = tail;
        //如果队列为空
        if (t == null) {
	        //通过CAS将一个空Node作为头结点
            if (compareAndSetHead(new Node()))
	            //将空Node也作为尾结点
	            tail = head;
        } else {
	        //将新Node的前驱指针指向尾结点
            node.prev = t;
            //通过CAS将新Node作为尾结点，若失败则重新开始循环
            if (compareAndSetTail(t, node)) {
	            //尾结点的后驱指针指向新Node
                t.next = node;
                //返回旧的尾结点
                return t;
            }
        }
    }
}

```
 addWaiter总是能够保证以线程安全且非阻塞的方式将结点添加到队列尾部。  
 若AQS队列为空，则添加后的队列为：  
 ![这里写图片描述](https://img-blog.csdn.net/20180916144245334?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FiYzEyM2x6Zg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)  
 若AQS队列不为空，则一般情况下为：  
 ![这里写图片描述](https://img-blog.csdn.net/20180916144803165?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FiYzEyM2x6Zg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)  
 添加到队列以后，再通过acquireQueued方法尝试获取资源：

 
```
final boolean acquireQueued(final Node node, int arg) {
	//是否成功获取到资源
	boolean failed = true;
    try {
	    //该node对应的线程是否被打断
        boolean interrupted = false;
        for (;;) {
	        //获得新Node前驱Node
            final Node p = node.predecessor();
            //如果排在前面的是头结点，那么这个Node是除去头结点外的第一个结点
            //这时，这个Node就有资格尝试获取资源了(尝试调用tryAcquire)
            if (p == head && tryAcquire(arg)) {
	            //将此node设为头结点（会将前驱后驱指针设置为null）
                setHead(node);
                //删除这个结点，代表出队
                p.next = null; 
                //成功拿到资源
                failed = false;
                //返回该线程是否打断
                return interrupted;
            }
            //首先调用shouldParkAfterFailedAcquire判断本次获取资源失败后是否应当阻塞当前线程，
            //如果返回true，则使当前线程进入等待状态直到该方法返回
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
	            //如果等待过程中该线程被中断，则标记为true
                interrupted = true;
        }
    } finally {
	    //如果线程等待超时或被中断则将这个Node设置为CANCELLED状态
        if (failed)
            cancelAcquire(node);
    }
}

private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
	//获取前面的Node的等待状态
	int ws = pred.waitStatus;
	//如果前面的结点获取到资源时可以通知本结点，那么返回true，执行parkAndCheckInterrupt
    if (ws == Node.SIGNAL)
        return true;
    //如果前面的Node为CANCELLED状态
    if (ws > 0) {
	    //一直往队列前面找，直到找到正常状态的Node，处于CANCELLED状态的Node在循环结束后会失去引用
        do {
	        //往队列前面找
	        pred = pred.prev;
	        //将node的前驱指针指向pred（意味着忽略掉CANCELLED状态的Node，这种状态的Node失去了引用会被GC）
            node.prev = pred;
        } while (pred.waitStatus > 0);
        //将正常状态的Node的后驱指针指向node
        pred.next = node;
    } else {
	    //通过CAS将状态设为SIGNAL
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}

private final boolean parkAndCheckInterrupt() {
	//调用park使线程进入等待状态，这里会被阻塞直到等到前面的Node通知或被interrupt
	LockSupport.park(this);
	//返回线程有没有被中断
	return Thread.interrupted();
}

```
 Node.SIGNAL状态表示了这个结点对应的资源释放资源后会主动通知队列后面的线程。  
 shouldParkAfterFailedAcquire从方法名可以看出，该方法的作用是当获取资源失败时是否应该使该线程等待，该方法同时也会清除掉这个Node前面处于CANCELLED状态的Node。  
 parkAndCheckInterrupt方法会使当前线程进入阻塞等待状态，脱离阻塞状态后，会判断当前线程有没有被interrupt。如果调用的是acquireInterruptibly方法，那么如果这时Thread.interrupted()返回true，那么会抛出InterruptedException

 acquireQueued方法的执行流程如下：  
 1、首先判断该Node是否在队列最前端（不包括头结点），如果不是跳到步骤2，如果是则调用tryAcquire尝试获取资源，成功获取到资源后将此Node出队，方法结束。如果没有成功获取到资源，跳转到步骤2  
 2、调用shouldParkAfterFailedAcquire方法，如果该Node前面的Node处于SIGNAL状态则返回true进入步骤3。否则如果前面的Node处于CANCELLED状态，则循环删除CANCELLED状态的结点，返回false，跳转到步骤1。如果前面的Node状态正常，则通过CAS将前面的Node设为SIGNAL状态，跳转到步骤1  
 3、阻塞该线程直到资源释放后得到通知或线程被中断，跳转到步骤1。  
 用流程图表示acquireQueued状态：  
 ![这里写图片描述](https://img-blog.csdn.net/20180909223246139?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FiYzEyM2x6Zg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

 在acquireQueued方法结束前，若线程仍然没有成功地获取到资源，那么会调用cancelAcquire方法进行收尾工作：

 
```
private void cancelAcquire(Node node) {
	if (node == null)
        return;
	//清除Node的thread变量
    node.thread = null;
	//获取前驱结点
    Node pred = node.prev;
    //清除前面所有处于CANCELLED状态的结点
    while (pred.waitStatus > 0)
        node.prev = pred = pred.prev;
    Node predNext = pred.next;
    //将当前结点设为CANCELLED状态
    node.waitStatus = Node.CANCELLED;
    //如果当前结点本身处于队列尾部，那么直接清除掉自身(通过CAS重新设置尾结点的方法)
    if (node == tail && compareAndSetTail(node, pred)) {
        compareAndSetNext(pred, predNext, null);
    } else {
        int ws;
        if (pred != head &&
            ((ws = pred.waitStatus) == Node.SIGNAL ||
             (ws <= 0 && compareAndSetWaitStatus(pred, ws, Node.SIGNAL))) &&
            pred.thread != null) {
            //删除当前结点
            Node next = node.next;
            if (next != null && next.waitStatus <= 0)
                compareAndSetNext(pred, predNext, next);
        } else {
	        //唤醒node前面的线程
            unparkSuccessor(node);
        }
        node.next = node;
    }
}

```
 
##### []()3、release方法

 release方法用于释放独占资源

 
```
public final boolean release(int arg) {
	//调用tryRelase释放资源，如果没有成功方法结束并返回false
	if (tryRelease(arg)) {
		//获取队列头节点
        Node h = head;
        //如果头节点不为null且不为默认0
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}

private void unparkSuccessor(Node node) {
	//获取头结点状态
    int ws = node.waitStatus;
    //若头结点状态不为CANCELLED或0则通过CAS将其状态修改为0
    if (ws < 0)
        compareAndSetWaitStatus(node, ws, 0);
    //将node后面的结点赋给s
    Node s = node.next;
    //如果后面的Node状态为CANCELLED
	if (s == null || s.waitStatus > 0) {
		//删除这个结点
	    s = null;
	    //从队列尾部搜索，删除CANCELLED状态的结点
        for (Node t = tail; t != null && t != node; t = t.prev)
		    if (t.waitStatus <= 0)
		        s = t;
    }
    //激活正在等待资源的线程
    if (s != null)
        LockSupport.unpark(s.thread);
}

```
 release操作相对于acquire方法还是简单很多，执行步骤如下：  
 1、首先尝试调用tryRelease方法尝试释放资源，如果释放失败方法直接返回。如果释放成功，执行步骤2  
 2、获取头结点，如果头结点为null或头结点状态为0，则说明没有线程在队列中等待，方法返回true，资源释放成功。否则，执行步骤3  
 3、获取头结点的状态，如果状态正常（表明有线程正在等待）则将这个头结点状态设为0，如果头结点中后面的结点有处于CANCELLED状态的，则从队列后面依次移除这些节点直到遇到正常状态的结点。然后，激活那个队列最前端在等待的线程获取资源。

 
##### []()4、acquireShared

 线程调用acquireShared代表需要获取共享资源。

 
```
public final void acquireShared(int arg) {
	if (tryAcquireShared(arg) < 0)
	    doAcquireShared(arg);
}

```
 如果tryAcquireShared方法返回值小于0则获取资源失败，调用doAcquireShared将线程加入队列。

 
```
private void doAcquireShared(int arg) {
	//同样和acquire方法类似，将线程包装为Node加入到队列，只不过模式为SHARED
	final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
	    //该线程是否被打断
        boolean interrupted = false;
        //通过循环尝试获取资源直到成功获取或被打断
        for (;;) {
	        //获取前驱结点
            final Node p = node.predecessor();
            //如果前驱结点是头结点
            if (p == head) {
	            //尝试获取资源
                int r = tryAcquireShared(arg);
                //r大于等于0代表获取资源成功
                if (r >= 0) {
	                //将当前结点设为头结点并唤醒后面的线程
                    setHeadAndPropagate(node, r);
                    p.next = null;
                    //如果当前线程被中断，则调用当前线程的interrupt方法
                    if (interrupted)
                        selfInterrupt();
                    failed = false;
                    return;
                }
            }
            //和acquireQueued类似
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}

//该方法将头结点设为node，如果还有剩余资源可用再唤醒后面的线程
private void setHeadAndPropagate(Node node, int propagate) {
	//获取头结点
	Node h = head; 
	//将node设为新的头结点
    setHead(node);
    //如果还有剩余的资源
    if (propagate > 0 || h == null || h.waitStatus < 0 ||
	        (h = head) == null || h.waitStatus < 0) {
	    //获取node的后驱结点
        Node s = node.next;
        //如果后驱结点为空或者后驱结点是共享模式的
        if (s == null || s.isShared())
	        doReleaseShared();
    }
}

private void doReleaseShared() {
	for (;;) {
        Node h = head;
        //如果头结点不为空并且队列不为空
        if (h != null && h != tail) {
	        //获取头结点状态
            int ws = h.waitStatus;
            //如果处于SIGNAL状态
            if (ws == Node.SIGNAL) {
	            //通过CAS将SIGNAL状态修改为0，如果修改失败重新开始循环
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                    continue;
                //唤醒后面的节点
                unparkSuccessor(h);
            //如果状态为0那么通过CAS将0状态修改为PROPAGATE状态，修改失败重新开始循环
            } else if (ws == 0 && !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                continue;
        }
        //如果头结点没有被其它线程更改，方法结束
        if (h == head)
            break;
    }
}

```
 acquireShared方法逻辑上大体和acquire方法差不多，在获取资源之后，acquireShared会尝试激活后面的线程获取资源（因为资源是共享的）

 
##### []()5、releaseShared方法

 releaseShared方法用于释放共享资源

 
```
public final boolean releaseShared(int arg) {
	if (tryReleaseShared(arg)) {
	    doReleaseShared();
        return true;
    }
    return false;
}

```
 和release方法类似，该方法首先会调用tryReleaseShared方法尝试释放资源，如果释放失败方法结束，直接返回false。如果成功释放了资源则调用AQS内部方法doReleaseShared方法。

 
```
private void doReleaseShared() {
	for (;;) {
		//获取头结点
        Node h = head;
        //如果队列中存在等待的线程
        if (h != null && h != tail) {
	        //获取头结点状态
            int ws = h.waitStatus;
            //如果状态为SIGNAL
            if (ws == Node.SIGNAL) {
	            //通过CAS将状态由SIGNAL设为0，如果失败则重新开始循环
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                    continue;
                //成功后唤醒等待的线程
                unparkSuccessor(h);
            }
            //如果为默认状态那么尝试通过CAS将状态由0设为PROPAGATE，设置失败则重新开始循环
            else if (ws == 0 && !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                continue;
        }
        //如果头结点没有被其它线程更改则跳出循环
	    if (h == head)
            break;
	}
}

```
 
--------
 
### []()四、AQS中的ConditionObject

 ConditionObject是AQS的一个public权限的非静态内部类，可以用来实现类似于Object的wait和notify功能。  
 要想使用ConditionObject，AQS的实现类必须是独占式资源模式，需要重写tryAcquire、tryRelease、isHeldExclusively方法。ConditionObject的一个典型应用就是ReentrantLock。

 ConditionObject的API：

 
     方法签名                                                      | 作用                            
     --------------------------------------------------------- | ------------------------------ 
     void signal()                                             | 激活一个等待的线程(一般为Condition队列头部的结点)
     void signalAll()                                          | 激活所有等待的线程                     
     void awaitUninterruptibly()                               | 当前线程等待，忽略线程的interrupt         
     void await() throws InterruptedException                  | 当前线程等待，和Object的wait类似         
     long awaitNanos(long) throws InterruptedException         | 当前线程等待，最多等待指定的纳秒数             
     boolean awaitUntil(Date) throws InterruptedException      | 当前线程等待，最多等待到指定的Date           
     boolean await(long, TimeUnit) throws InterruptedException | 当前线程等待，最多等待指定的时间              

  
 await方法在AQS中，当前线程必须要获取资源后才能调用，否则会抛出IllegalMonitorStateException异常。 下面是ConditionObject的成员变量及其类定义：

 
```
public class ConditionObject implements Condition, java.io.Serializable {
	private static final int REINTERRUPT =  1;
	//表示wait方法调用后阻塞，被线程的interrupt打断
    private static final int THROW_IE    = -1;
	//等待队列头结点
	private transient Node firstWaiter;
	//等待队列尾结点
	private transient Node lastWaiter;
}

```
 每个ConditionObject实例维护了一个线程等待队列，调用await类似方法且尚未被唤醒的线程都保存在这个队列中

 
##### []()1、await方法解析

 
```
public final void await() throws InterruptedException {
	if (Thread.interrupted())
		throw new InterruptedException();
	//添加到等待队列
	Node node = addConditionWaiter();
	//尝试从AQS中释放所有的资源(即变量state)，返回释放的资源数
    int savedState = fullyRelease(node);
    int interruptMode = 0;
    
	//判断是否在AQS队列中，如果在AQS队列中则跳出循环
    while (!isOnSyncQueue(node)) {
	    //阻塞当前线程，等待被激活
		LockSupport.park(this);
		//检查是否在阻塞过程中被中断
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
			break;
    }
    //调用acquireQueued尝试获取资源（重新获得锁）
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    //如果Condition队列中依然由线程等待，那么检查是否是CANCELLED状态并清除这些废弃的结点
    if (node.nextWaiter != null)
        unlinkCancelledWaiters();
    //如果线程被中断过，那么抛出InterruptedException异常
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
}

```
 await状态分为4步骤：  
 1、调用addConditionWaiter加入Condition队列，该方法首先会清除掉Condition队列中CANCELLED状态的结点，然后构造一个新的结点，包含有当前线程的对象，接着添加到Condition队列的尾端。

 
```
private Node addConditionWaiter() {
	Node t = lastWaiter;
	//如果Condition队列不为空并且队列尾部结点状态不为CONDITION
    if (t != null && t.waitStatus != Node.CONDITION) {
	    //删除Condition队列中所有CANCELLED状态的结点
	    unlinkCancelledWaiters();
        t = lastWaiter;
    }
    //构造新的结点，状态为CONDITION
    Node node = new Node(Thread.currentThread(), Node.CONDITION);
    //如果队列为空那么将这个新结点作为Condition队列头结点，否则加入队列尾部
    if (t == null)
	    firstWaiter = node;
    else
	    //Condition队列是通过Node的nextWaiter而不是next来维持队列的
        t.nextWaiter = node;
    
    lastWaiter = node;
    return node;
}

```
 2、调用fullyRelease释放AQS所有的资源（可以理解为释放锁）。  
 如果释放资源失败（意味着已经有其它线程持有该资源（可以理解为没有加锁就调用wait方法）），那么会抛出IllegalMonitorStateException异常。抛出异常后，ConditionObject会将结点的状态设置为CANCELLED状态。

 
```
final int fullyRelease(Node node) {
	boolean failed = true;
    try {
	    //获取AQS中state变量的值
        int savedState = getState();
        //调用release释放所有资源
        if (release(savedState)) {
            failed = false;
            return savedState;
        //释放资源失败则抛出IllegalMonitorStateException异常
        } else {
            throw new IllegalMonitorStateException();
        }
    } finally {
	    //资源释放失败则将这个Node标记为CANCELLED状态等待被删除
        if (failed)
            node.waitStatus = Node.CANCELLED;
    }
}

```
 3、进入自旋操作：通过isOnSyncQueue方法判断当前Node是否在AQS队列中。

 
```
final boolean isOnSyncQueue(Node node) {
	//如果结点状态为CONDITION或者结点前驱指针为null，那么肯定不在AQS队列中
	if (node.waitStatus == Node.CONDITION || node.prev == null)
        return false;
    //如果该节点后驱结点不为null，那么肯定在AQS队列中(因为Condition)
    if (node.next != null)
        return true;
    //遍历AQS队列判断
    return findNodeFromTail(node);
}

```
 若在AQS队列中，跳出循环，当前步骤结束。若不在AQS队列中，则调用LockSupport类的park方法阻塞当前线程进行等待。等待结束后，调用checkInterruptWhileWaiting检查是否是因为被interrupt导致的线程脱离阻塞状态，若是，当前步骤结束。

 只有当调用了signal或者signalAll方法之后，这个保存有等待线程的Node才会被加入到AQS队列中。

 
```
private int checkInterruptWhileWaiting(Node node) {
	//若当前线程没有中断则直接返回0，否则调用transferAfterCancelledWait方法
	return Thread.interrupted() ? (transferAfterCancelledWait(node) ? THROW_IE : REINTERRUPT) : 0;
}

final boolean transferAfterCancelledWait(Node node) {
	//如果成功地将结点状态由CONDITION设为0
	if (compareAndSetWaitStatus(node, Node.CONDITION, 0)) {
		//把结点加入到AQS队列中
	    enq(node);
        return true;
    }
    //如果设置失败，则代表有线程已经将node由CONDITION设为0，那么一直等待
    //直到被该Node加入到AQS队列
    while (!isOnSyncQueue(node))
        Thread.yield();
    return false;
}

```
 4、线程被唤醒后，首先调用acquireQueued不停地尝试获取资源（可以理解为重新获得锁）。在获取到资源之后，才会执行await方法后面的代码。获取到资源后，await方法还会尝试清除掉Condition队列中CANCELLED状态的结点。并且如果当前等待的线程是通过interrupt脱离阻塞状态的，则抛出InterruptedException异常。

 
##### []()2、signal方法

 signal方法作用是唤醒ConditionObject中等待队列头部的线程，类似于Object中的notify方法

 
```
public final void signal() {
	//如果当前线程没有获得到资源，则抛出IllegalMonitorStateException异常
	if (!isHeldExclusively())
	    throw new IllegalMonitorStateException();
	//获取等待队列的第一个结点
    Node first = firstWaiter;
    //如果队列包含等待的线程(即在await方法阻塞的线程)
    if (first != null)
	    //激活这个线程
	    doSignal(first);
}

```
 要想通过signal唤醒线程，首先需要当前线程获取到独占资源，否则会抛出IllegalMonitorStateException异常。这点其实和Object的notify是一样的，要想唤醒其它线程首先必须要持有这个锁。然后，doSignal会尝试找出最前端的非CANCELLED状态的结点，并将这个结点加入到AQS队列中等待被激活。

 
```
private void doSignal(Node first) {
	//从队列头部开始，忽略掉CANCELLED状态的结点，直到找到CONDITION状态的结点为止
	do {
	    if ((firstWaiter = first.nextWaiter) == null)
	        lastWaiter = null;
        first.nextWaiter = null; 
    //调用transferForSignal将这个需要唤醒的结点加入到AQS队列
    } while (!transferForSignal(first) && (first = firstWaiter) != null);
}

```
 
```
final boolean transferForSignal(Node node) {
	//通过CAS将CONDITION设为0，如果设置没有成功，意味着Node处于CANCELLED状态
	if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
        return false;
	//将等待线程的结点加入到AQS队列，返回node的前驱结点
    Node p = enq(node);
    int ws = p.waitStatus;
    //如果前驱结点等待状态为CANCELLED或者没有成功设为SIGNAL状态，则激活这个前驱结点包含的线程
    if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
        LockSupport.unpark(node.thread);
    return true;
}

```
 这里有一点需要注意：signal方法唤醒等待线程不是通过LockSupport.unpark(node.thread)这段代码唤醒的，而是通过最上面compareAndSetWaitStatus(node, Node.CONDITION, 0)这段代码之后，保存等待线程的结点状态被设置为0，然后将其**加入AQS队列**中等待，当调用signal线程释放了资源后（相当于调用ReentrantLock的unlock方法后），由AQS的release方法的unparkSuccessor去唤醒这个等待的线程。

 为了更好地理解await和signal的等待和唤醒的过程，可以参考下面的流程图：  
 ![这里写图片描述](https://img-blog.csdn.net/20180916223510391?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FiYzEyM2x6Zg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

   
  