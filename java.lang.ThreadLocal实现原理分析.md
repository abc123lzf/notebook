---
title: java.lang.ThreadLocal实现原理分析
date: 2018-08-23 16:09:44
tags: CSDN迁移
---
 版权声明：尊重原创，转载请注明出处 [ https://blog.csdn.net/abc123lzf/article/details/81978210]( https://blog.csdn.net/abc123lzf/article/details/81978210)   
  ### []()一、简介

 `ThreadLocal` 隶属 `java.lang` 包，表示线程私有变量，也可叫做线程本地变量。它为单个线程单独创立了一个副本，每个线程只可访问属于自己的变量，不可访问和修改别的线程所属的变量。  
  `ThreadLocal` 属于一个泛型类，泛型参数为变量的类型，可以通过重写 `initialValue` 方法来实现对该变量初始值的设置。  
  `ThreadLocal` 提供了以下API实现对变量的控制：

 
```
public T get(); //返回该变量
public void set(T value); //设置该变量
public void remove(); //移除该变量

```
 例如下面的代码：

 
```
public class Test {

	private static ThreadLocal<Integer> tl = new ThreadLocal<Integer>() {
		//重写设置初始值的方法
		@Override
		protected Integer initialValue() {
	        return 0;
	    }
	};

	public static void main(String[] args) {
		System.out.println(tl.get());
		tl.set(tl.get() + 1);
		System.out.println(tl.get());
		
		Thread t = new Thread() {
			public void run() {
				System.out.println(tl.get());
			}
		};
		
		t.start();
	}
}

```
 输出为：

 
```
0
1
0

```
 看到这里想必大家理解了ThreadLocal的使用方法及其作用了吧。

 
### []()二、源码分析

 `ThreadLocal` 的主要方法有：

 
```
public T get(); //返回该变量
public void set(T value); //设置该变量
public void remove(); //移除该变量
protected T initialValue(); //返回变量的初始值

```
 
##### []()(1)initialValue方法

 `initialValue` 方法的作用是为变量设置初始值，该方法的默认实现是：

 
```
protected T initialValue() {
	return null;
}

```
 如果想要为该变量设置一个初始值，只需重写该方法即可，例如：

 
```
@Override
protected String initialValue() {
	return "hello world";
}

```
 `ThreadLocal` 对象中很多方法的实现会调用该方法来设置初始值。

 
##### []()(2)get方法分析

 想要了解 `ThreadLocal` 运作机制，我们可以从 `get` 方法作为切入点分析。

 
```
public T get() {
	Thread t = Thread.currentThread();
	//获取当前线程对象t对应的ThreadLocalMap
	ThreadLocalMap map = getMap(t);
	if (map != null) {
		//以当前ThreadLocal对象为键查找值
		ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
			@SuppressWarnings("unchecked")
			T result = (T)e.value;
			return result;
		}
	}
	return setInitialValue();
}

```
 首先是获取当前运行线程对象，然后根据该线程对象找到对应的 `ThreadLocalMap` （ `ThreadLocalMap` 是 `ThreadLocal` 的一个 `default` 访问权限内部类，其作用我们稍后分析）。如果找到了该线程对应的 `ThreadLocalMap` ，则通过当前 `ThreadLocal` 对象作为键查找 `Map` 中对应的 `Entry` （键值对）对象，如果查找结果不为 `null` ，则返回 `Entry` 对象的 `value` 。否则调用 `setInitialValue` 方法将当前 `ThreadLocal` 对象(this)和变量作为键值对存入 `ThreadLocalMap` 并返回变量。

 **getMap方法：**

 
```
ThreadLocalMap getMap(Thread t) {
	return t.threadLocals;
}

```
 该方法很简单，即返回Thread对象的成员变量threadLocals。  
 查看java.lang.Thread对象源码：

 
```
ThreadLocal.ThreadLocalMap threadLocals = null;

```
 相信很多人看到这里已经大致明白：每个 `Thread` 对象会持有一个 `ThreadLocalMap` ，当我们创建一个 `ThreadLocal` 对象时，该 `ThreadLocal` 对象本身作为键，变量作为值存入该 `ThreadLocalMap` 中，来实现线程的私有变量。

 **下面再来看setInitialValue方法的实现：**

 
```
private T setInitialValue() {
	T value = initialValue();
	Thread t = Thread.currentThread();
	ThreadLocalMap map = getMap(t);
	if (map != null)
		map.set(this, value);
	else
		createMap(t, value);
	return value;
}

void createMap(Thread t, T firstValue) {
	t.threadLocals = new ThreadLocalMap(this, firstValue);
}

```
 该方法作用是将 `ThreadLocal` 对象存入当前线程对象的 `ThreadLocalMap` 中，最后返回变量初始值。  
 该方法首先会调用 `initialValue` 方法初始化变量，然后获取当前 `Thread` 对象所属的 `ThreadLocalMap` （即成员变量 `threadLocals` ），如果 `Thread` 对象成员变量 `threadLocals` 不为空，则将当前 `ThreadLocal` 对象及其变量存入Map，否则会调用 `createMap` 方法为当前 `Thread` 对象新建一个 `ThreadLocalMap` 并存入当前 `ThreadLocal` 对象及其变量。

 
##### []()(3)ThreadLocal内部类：ThreadLocalMap

 `ThreadLocalMap` 负责存储 `ThreadLocal` 及其变量，即 `ThreadLocal` 对象本身作为键， `ThreadLocal` 存储的变量作为值。每个 `Thread` 对象在声明一个 `ThreadLocal` 后会持有一个 `ThreadLocalMap` 的引用，来实现 `ThreadLocal` 的功能。

 `ThreadLocalMap` 持有一个内部类 `Entry` ，类似于 `HashMap.Node` 类，负责保存键值对。

 
```
static class Entry extends WeakReference<ThreadLocal<?>> {
	Object value;
	Entry(ThreadLocal<?> k, Object v) {
		super(k);
		value = v;
	}
}

```
 `Entry` 继承了 `WeakReference` 类，使 `Entry` 的键为弱引用。

 看到这里很多人或许有这样一个疑问：**为什么Entry要继承WeakReference？**  
 既然将 `ThreadLocal` 声明为弱引用，那么自然会联想到和GC有关。

 如果不声明为弱引用，以最上面Test类的代码为例，当我们将上述 `ThreadLocal` 类型的静态变量 `tl` 设置为 `null` 时， `Thread` 对象成员变量 `threadLocals` 依然保留有一个 `ThreadLocalMap` ，该Map中持有保存该 `ThreadLocal` 的 `Entry` ，在这个线程运行期间无法GC，从而引发内存泄漏。所以， `Entry` 的键要声明为弱引用。  
 ![这里写图片描述](https://img-blog.csdn.net/20180823160659300?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FiYzEyM2x6Zg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)  
 当我们弃用一个 `ThreadLocal` 对象时，最好通过调用其 `remove` 方法而不是直接设为 `null` 。

 当向 `ThreadLocalMap` 添加一个 `ThreadLocal` 时，会获取 `ThreadLocal` 对象的 `threadLocalHashCode` 获取 `HashCode` 添加至Map中而不是调用 `hashCode()` 方法。  
  `ThreadLocalMap` 持有一个 `Entry` 数组，采用线性探测法处理哈希冲突，而不是 `HashMap` 的拉链法。  
 为了尽量避免哈希冲突， `ThreadLocal` 提供了 `hashCode` 的计算代码：

 
```
private static AtomicInteger nextHashCode = new AtomicInteger();

/**
* The difference between successively generated hash codes - turns
* implicit sequential thread-local IDs into near-optimally spread
* multiplicative hash values for power-of-two-sized tables.
*/
private static final int HASH_INCREMENT = 0x61c88647;

/**
* Returns the next hash code.
*/
private static int nextHashCode() {
	return nextHashCode.getAndAdd(HASH_INCREMENT);
}

```
 每当新建一个 `ThreadLocal` 对象，会调用 `nextHashCode` 获取自己的 `hashCode`   
 关于魔数 `0x61c88647` ，有兴趣的可以百度，这里不再赘述。

 
##### []()(4)set方法分析

 
```
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
	if (map != null)
		map.set(this, value);
	else
		createMap(t, value);
}

```
 与 `get` 方法类似， `set` 方法首先会获取当前运行的 `Thread` 对象并通过该对象获取对应的 `ThreadLocalMap` ，如果map为空，则为当前 `Thread` 对象新建一个 `ThreadLocalMap` ，否则直接将 `value` 放入 `map` 中。

 
##### []()(5)remove方法

 
```
public void remove() {
	ThreadLocalMap m = getMap(Thread.currentThread());
	if (m != null)
		m.remove(this);
}

```
 同样，获取当前 `Thread` 对应的 `ThreadLocalMap` ，然后调用 `ThreadLocalMap` 的 `remove` 方法移除 `ThreadLocal` 对象，无需通过弱引用机制对该 `ThreadLocal` 对象进行GC。

 **如果本文有错误或对文章有疑问，欢迎在评论区评论。**

   
  