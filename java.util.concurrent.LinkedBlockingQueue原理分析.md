---
title: java.util.concurrent.LinkedBlockingQueue原理分析
date: 2018-09-14 10:49:38
tags: CSDN迁移
---
 版权声明：尊重原创，转载请注明出处 [ https://blog.csdn.net/abc123lzf/article/details/82689900]( https://blog.csdn.net/abc123lzf/article/details/82689900)   
  ### 一、引言

 LinkedBlockingQueue是一个用于并发环境下的阻塞队列集合类，它可以用于生产-消费者模型。

 **什么是阻塞队列？**   
 阻塞队列和一般的普通队列（如LinkedList）不同，它可以实现线程的阻塞添加元素和取出元素   
 阻塞添加：当队列元素的数量达到上限时，队列会阻塞之后尝试添加元素的线程，直到有元素被移除后才会通知该线程。   
 阻塞删除：当队列元素为空时，队列会阻塞之后尝试取得元素的线程，直到有元素被添加时才会激活该线程。

 在Java并发工具包中，阻塞队列都实现了BlockingQueue接口，该接口继承了Queue接口：

 
```
//省略Queue接口的方法
public interface BlockingQueue<E> extends Queue<E> {
    //将元素e入队，如果队列满了调用该方法会阻塞
    void put(E e) throws InterruptedException;
    //将元素e入队，最多等待timeout时间，返回是否添加成功
    boolean offer(E e, long timeout, TimeUnit unit) throws InterruptedException;
    //取出元素并返回，队列为空会阻塞
    E take() throws InterruptedException;
    //取出元素并返回，最多等待timeout时间
    E poll(long timeout, TimeUnit unit) throws InterruptedException;
    //返回该队列还可以无阻塞地添加几个元素
    int remainingCapacity();
    //将队列中所有元素加入到集合c中
    int drainTo(Collection<? super E> c);
    //将队列中队首前maxElements个元素加入到集合c中
    int drainTo(Collection<? super E> c, int maxElements);
}
```
 在java.util.concurrent包中，实现BlockingQueue接口的，除了LinkedBlockingQueue，还有LinkedBlockingDeque、ArrayBlockingQueue、PriorityBlockingQueue

 LinkedBlockingQueue的构造方法：

 
     构造方法                                                        | 解释                                    
     ----------------------------------------------------------- | -------------------------------------- 
     public LinkedBlockingQueue()                                | 最大队列长度为Integer.MAX_VALUE              
     public LinkedBlockingQueue(int capacity)                    | 最大队列长度为capacity                       
     public LinkedBlockingQueue(Collection&lt;? extends E&gt; c) | 最大长度为Integer.MAX_VALUE，并将集合c所有元素添加到该队列

 
### 二、源码分析

 LinkedBlockingQueue的类定义和成员变量：

 
```
public class LinkedBlockingQueue<E> extends AbstractQueue<E> 
        implements BlockingQueue<E>, java.io.Serializable {
    //队列的最大长度
    private final int capacity;
    //队列包含的元素数量
    private final AtomicInteger count = new AtomicInteger();
    //队列的头结点
    transient Node<E> head;
    //队列的尾结点
    private transient Node<E> last;
    //出队操作锁
    private final ReentrantLock takeLock = new ReentrantLock();
    //用于实现线程等待获取元素
    private final Condition notEmpty = takeLock.newCondition();
    //入队操作锁
    private final ReentrantLock putLock = new ReentrantLock();
    //用于实现线程等待放入元素
    private final Condition notFull = putLock.newCondition();
}
```
 LinkedBlockingQueue采用两个锁两个Condition来实现队列的阻塞操作。

 和很多集合类一样，LinkedBlockingQueue采用了内部类Node来保存元素，它通过Node维护了一个单向链表来实现队列。

 
```
static class Node<E> {
    E item;
    Node<E> next;
    Node(E x) { item = x; }
}
```
 Node类很简单，保存了一个元素的引用和后驱指针

 
##### 1、加入元素（入队）

 对于LinkedBlockingQueue，加入元素的方法有：add、put、offer方法

 下面我们讲解put方法：   
 put方法用于将元素加入到队列尾部，若队列已满，则当前线程会一直阻塞直到被队列通知或Interrupt   
 LinkedBlockingQueue不能添加null元素，所以put方法尝试添加null元素会抛出NullPointerException

 
```
public void put(E e) throws InterruptedException {
    if (e == null) throw new NullPointerException();
    int c = -1;
    //构造一个新的Node
    Node<E> node = new Node<E>(e);
    //获得入队锁
    final ReentrantLock putLock = this.putLock;
    //获得元素数量计数器
    final AtomicInteger count = this.count;
    //加锁
    putLock.lockInterruptibly();
    try {
        //如果队列元素已满，则调用Condition的await等待
        while (count.get() == capacity)
            notFull.await();
        //将元素加入到队列
        enqueue(node);
        //将队列元素数量计数器加1并赋给c
        c = count.getAndIncrement();
        //如果队列元素没满，则激活在notFull上等待的线程
        if (c + 1 < capacity)
            notFull.signal();
    } finally {
        putLock.unlock();
    }
    //如果队列为空，激活一个获取元素的线程
    if (c == 0)
        signalNotEmpty();
}

private void enqueue(Node<E> node) {
    last = last.next = node;
}

private void signalNotEmpty() {
    final ReentrantLock takeLock = this.takeLock;
    takeLock.lock();
    try {
        //激活一个等待获取元素的线程
        notEmpty.signal();
    } finally {
        takeLock.unlock();
    }
}
```
 
##### 2、取出元素（出队）

 对于LinkedBlockingQueue，取出元素的方法有poll、peek、 take   
 对于take方法，当队列元素为空时，当前线程会被阻塞直到有元素被添加或线程被interrupt。

 
```
public E take() throws InterruptedException {
    E x;
    int c = -1;
    final AtomicInteger count = this.count;
    final ReentrantLock takeLock = this.takeLock;
    //对出队锁进行加锁
    takeLock.lockInterruptibly();
    try {
        //如果队列为空，则将线程加入到notEmpty等待队列
        while (count.get() == 0)
            notEmpty.await();
        //入队，相当于添加到链表尾部
        x = dequeue();
        //将队列元素计数器减1
        c = count.getAndDecrement();
        //如果队列还有其它元素，则通知一个需要获取元素的线程
        if (c > 1)
            notEmpty.signal();
    } finally {
        takeLock.unlock();
    }
    //如果队列已满，激活一个加入元素的线程
    if (c == capacity)
        signalNotFull();
    return x;
}

private void signalNotFull() {
    final ReentrantLock putLock = this.putLock;
    putLock.lock();
    try {
        //激活所有加入元素的线程
        notFull.signal();
    } finally {
        putLock.unlock();
    }
}
```
 
##### 3、清空队列

 可以调用clear方法来清空整个队列

 
```
public void clear() {
    //同时对两个锁加锁
    fullyLock();
    try {
        //从链表头部开始依次删除
        for (Node<E> p, h = head; (p = h.next) != null; h = p) {
            h.next = h;
            p.item = null;
        }
        head = last;
        //如果队列以前是满的，则将计数器设为0并且激活等待将元素入队的线程
        if (count.getAndSet(0) == capacity)
            notFull.signal();
    } finally {
        fullyUnlock();
    }
}
```
 清空队列方法首先会同时将入队锁和出队锁加上，然后从链表头部依次清除元素，清除完毕后，如果队列之前是满的，那么会通知其中一个等待将元素入队的线程。

 
```
void fullyLock() {
    putLock.lock();
    takeLock.lock();
}

void fullyUnlock() {
    takeLock.unlock();
    putLock.unlock();
}
```
 这里加锁和解锁顺序不能颠倒，否则有可能造成死锁。

 
##### 总结

 LinkedBlockingQueue从整体上来看实现并不复杂，它通过了一个入队一个出队锁来保证至多有两个线程来修改队列，并获取对应的Condition来等待、唤醒线程。

   
  