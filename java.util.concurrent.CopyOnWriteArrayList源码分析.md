---
title: java.util.concurrent.CopyOnWriteArrayList源码分析
date: 2018-09-20 15:33:53
tags: CSDN迁移
---
 版权声明：尊重原创，转载请注明出处 [ https://blog.csdn.net/abc123lzf/article/details/82775281]( https://blog.csdn.net/abc123lzf/article/details/82775281)   
  ### []()一、引言

 CopyOnWriteArrayList是Java并发工具包中可支持并发操作的List集合类。当我们修改CopyOnWriteArrayList的元素时，CopyOnWriteArrayList会将内部的数组复制一份，然后进行修改，修改完成后再将保存数组的引用指向这个修改好的数组。这样做的好处是我们可以对容器进行并发的读操作而无需对整个容器加锁，它采用的是一种读、写分离的并发解决方案。

 
--------
 
### []()二、源码分析

 首先来看CopyOnWriteArrayList的类定义和成员变量：

 
```
public class CopyOnWriteArrayList<E>
    implements List<E>, RandomAccess, Cloneable, java.io.Serializable {

	final transient ReentrantLock lock = new ReentrantLock();
	private transient volatile Object[] array;
 }

```
 CopyOnWriteArrayList和ArrayList一样实现了List、RandomAccess、Cloneable、Serializable接口。  
 它只有两个成员变量：重入锁和存储对象的数组。

 
##### []()1、CopyOnWriteArrayList的构造

 CopyOnWriteArrayList提供了三种public构造方法：

 
     方法签名                                                         | 说明                      
     ------------------------------------------------------------ | ------------------------ 
     public CopyOnWriteArrayList()                                | 构造一个初始大小为0的List         
     public CopyOnWriteArrayList(Collection&lt;? extends E&gt; c) | 构造一个List，将集合c中所有元素作为初始元素
     public CopyOnWriteArrayList(E[] toCopyIn)                    | 构造一个List，将数组c中所有元素作为初始元素


```
public CopyOnWriteArrayList() {
	setArray(new Object[0]);
}

final void setArray(Object[] a) {
	array = a;
}

```
 CopyOnWriteArrayList不像ArrayList那样初始的时候构造一个长度为10的数组，而是直接构造一个长度为0的数组。在CopyOnWriteArrayList中，数组的长度就是包含的元素的数量。

 
```
public CopyOnWriteArrayList(Collection<? extends E> c) {
        Object[] elements;
        //如果集合c的类型就是CopyOnWriteArrayList，那么直接调用它的getArray方法获取数组
        if (c.getClass() == CopyOnWriteArrayList.class)
            elements = ((CopyOnWriteArrayList<?>)c).getArray();
        else {
        	//调用集合的toArray方法获取包含该集合所有元素的数组
            elements = c.toArray();
           	//如果elements的类型不为Object数组，那么调用Arrays的copyOf方法将类型调整为Object[]
            if (elements.getClass() != Object[].class)
                elements = Arrays.copyOf(elements, elements.length, Object[].class);
        }
        setArray(elements);
    }

```
 这个构造方法会将集合c中所有元素作为初始元素。  
 置于为什么在调用toArray方法后会检测是否为Object[]类型，注释中是这么解释的：

 
```
// c.toArray might (incorrectly) not return Object[] (see 6260652)

```
 大致意思是即使调用的是toArray方法，也不一定绝对返回的是Object[]类型，据说是Java的一个官方BUG，可以参考一下这篇博客：[https://blog.csdn.net/aitangyong/article/details/30274749](https://blog.csdn.net/aitangyong/article/details/30274749) ，这里就不再讨论这个问题了。

 
```
public CopyOnWriteArrayList(E[] toCopyIn) {
	//将toCopyIn所有引用拷贝一份
	setArray(Arrays.copyOf(toCopyIn, toCopyIn.length, Object[].class));
}

```
 这个构造方法比较简单，直接将数组toCopyIn元素的引用拷贝一份并加入到一个新的数组中，然后赋给成员变量array。

 
##### []()2、添加元素

 可以通过调用add、addIfAbsent、addAllAbsent、addAll等方法来添加或者修改元素。

 我们先从最基本的add方法开始分析：  
 add方法由两个重载的方法：add(E)和add(int,E)，前者扩大数组长度并向数组尾部添加元素，后者可以向指定的位置添加元素  
 add(E)源代码：

 
```
public boolean add(E e) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
    	//获取原数组
        Object[] elements = getArray();
        int len = elements.length;
        //构造一个为原数组长度+1长度的数组，并将原数组元素的引用拷贝到这个新数组中
        Object[] newElements = Arrays.copyOf(elements, len + 1);
        //将新元素加入到新数组的最后一个位置
        newElements[len] = e;
        //将原数组替换成新数组
        setArray(newElements);
        return true;
    } finally {
        lock.unlock();
    }
}

```
 但凡是调用修改数组元素的方法，都会首先加上锁来确保线程安全性。当读取元素时，则无需加上锁。  
 所以说，CopyOnWriteArrayList支持多个线程并行地读取元素，对于修改操作在同一时间只能支持一个线程。

 add(int,E)源代码：

 
```
public void add(int index, E element) {
	final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] elements = getArray();
        int len = elements.length;
        //index不能超出数组最大下标+1
        if (index > len || index < 0)
            throw new IndexOutOfBoundsException("Index: " + index + ", Size: " + len);
        //新数组
        Object[] newElements;
        //数组的移动距离
        int numMoved = len - index;
        //如果插入的位置为最后一个
        if (numMoved == 0)
            newElements = Arrays.copyOf(elements, len + 1);
        else {
            newElements = new Object[len + 1];
            //以index为分界线将两边元素分开，留出一个空位插入新的元素
            System.arraycopy(elements, 0, newElements, 0, index);
            System.arraycopy(elements, index, newElements, index + 1, numMoved);
        }
        //插入元素，此时index位置为null
        newElements[index] = element;
        //将原数组替换为新数组
        setArray(newElements);
    } finally {
        lock.unlock();
    }
}

```
 可以看出，CopyOnWriteArrayList对于添加元素会频繁地调用数组的复制操作。所以，CopyOnWriteArrayList比较适合在多线程环境下需要频繁读取集合元素而不是频繁的添加、修改元素的场景。

 
##### []()3、获取、迭代元素

 元素的获取有几种方法：可以调用get方法并传入下标获取元素，或者调用iterator获取迭代器遍历元素。  
 get方法获取：

 
```
public E get(int index) {
    return get(getArray(), index);
}

@SuppressWarnings("unchecked")
private E get(Object[] a, int index) {
    return (E) a[index];
}

```
 get方法很简单，传入数组下标index，直接从原数组中获取。

 迭代器遍历元素：  
 可以调用iterator、listIterator获取迭代器遍历元素

 
```
public Iterator<E> iterator() {
    return new COWIterator<E>(getArray(), 0);
}

public ListIterator<E> listIterator() {
    return new COWIterator<E>(getArray(), 0);
}

public ListIterator<E> listIterator(int index) {
    Object[] elements = getArray();
    int len = elements.length;
    if (index < 0 || index > len)
        throw new IndexOutOfBoundsException("Index: "+index);

    return new COWIterator<E>(elements, index);
}

```
 返回的迭代器都是CopyOnWriteArrayList的内部类COWIterator

 
```
static final class COWIterator<E> implements ListIterator<E> {
	//原数组引用
	private final Object[] snapshot;
	//下一个迭代元素的下标
	private int cursor;
	//构造时需要传入原数组引用，开始迭代的下标
	private COWIterator(Object[] elements, int initialCursor) {
        cursor = initialCursor;
        snapshot = elements;
    }

	public boolean hasNext() { return cursor < snapshot.length; }
	public boolean hasPrevious() { return cursor > 0; }
	
    @SuppressWarnings("unchecked")
    public E next() {
        if (! hasNext())
            throw new NoSuchElementException();
        return (E) snapshot[cursor++];
    }
    
    @SuppressWarnings("unchecked")
    public E previous() {
        if (! hasPrevious())
            throw new NoSuchElementException();
        return (E) snapshot[--cursor];
    }
	
	public int nextIndex() { return cursor; }
	public int previousIndex() { return cursor-1; }
	//不支持元素引用的修改操作
	public void remove() { throw new UnsupportedOperationException(); }
	public void set(E e) { throw new UnsupportedOperationException(); }
	public void add(E e) { throw new UnsupportedOperationException(); }
	
	//省略forEachRemaining方法
}

```
 CopyOnWriteArrayList的迭代器和ArrayList不同，ArrayList的迭代器实现类是非静态内部类，它的迭代是直接在ArrayList持有的数组上进行遍历操作，在迭代期间不允许通过调用非迭代器的方法进行修改元素，否则会抛出ConcurrentModificationException。而对于CopyOnWriteArrayList则不一样，当调用iterator或其它方法获取迭代器时，它迭代的数组是在构造迭代器时CopyOnWriteArrayList持有的数组，它保存了原数组的引用，即使该集合持有的元素后续发生改变，也并不会对迭代结果产生任何影响。并且，COWIterator不支持直接调用迭代器的方法进行修改元素。

 
##### []()总结

 1、CopyOnWriteArrayList是一个线程安全，读数据时无需加锁的List。  
 2、CopyOnWriteArrayList在进行修改操作时必须上锁，意味着同一时间只支持一个线程进行修改操作。  
 3、CopyOnWriteArrayList底层通过一个Object数组存储数据，且这个数组长度就是元素的数量，在添加或删除元素时都必须重新建立一个数组并进行复制操作，所以说，CopyOnWriteArrayList适合读取操作多修改操作少的情况。

   
  