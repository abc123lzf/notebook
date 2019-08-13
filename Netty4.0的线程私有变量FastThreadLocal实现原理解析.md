---
title: Netty4.0的线程私有变量FastThreadLocal实现原理解析
date: 2018-11-01 18:19:31
tags: CSDN迁移
---
 版权声明：尊重原创，转载请注明出处 [ https://blog.csdn.net/abc123lzf/article/details/83591922]( https://blog.csdn.net/abc123lzf/article/details/83591922)   
  ### []()一、引言

 Netty实现了自己的线程私有变量机制FastThreadLocal，相比于JDK的ThreadLocal，Netty的FastThreadLocal采用了一种常量索引而不是通过哈希表来存储变量，它能够始终以O(1)的速度来查找对应的变量。

 Netty实现了java.lang.Thread子类FastThreadLocalThread来保证能够使用FastThreadLocal，FastThreadLocalThread持有一个InternalThreadLocalMap实例，类似于JDK的ThreadLocalMap，InternalThreadLocalMap存储了这个线程持有的所有线程私有变量，以FastThreadLocal为键，对应的变量为值。

 在阅读本篇文章前，建议读者首先了解JDK的ThreadLocal的实现原理，可以参考我的博客：  
 [https://blog.csdn.net/abc123lzf/article/details/81978210](https://blog.csdn.net/abc123lzf/article/details/81978210)

 使用示例：

 
```
public class Test {	
	static final FastThreadLocal<String> LOCAL = new FastThreadLocal<String>() {
		protected String initialValue() {
			return "hello";
		}
	};
	
	public static void main(String[] args) throws InterruptedException, IOException {	
		System.out.println(LOCAL.get());
		LOCAL.set("bbbb");
		FastThreadLocalThread thread = new FastThreadLocalThread(()->{
			System.out.println(LOCAL.get());
			LOCAL.set("aaaa");
			System.out.println(LOCAL.get());
		});
		thread.start();
	}
}

```
 输出结果为：

 
```
hello
hello
aaaa

```
 
--------
 
### []()二、源码解析

 FastThreadLocal提供了以下API：

 
     方法签名                                               | 作用                                                   
     -------------------------------------------------- | ----------------------------------------------------- 
     public final V get()                               | 获取当前线程持有变量的值                                         
     public final V get(InternalThreadLocalMap)         | 获取指定InternalThreadLocalMap实例对应的变量值                   
     public final void set(V)                           | 设置当前线程持有变量的值                                         
     public final void set(InternalThreadLocalMap, V)   | 设置制定的InternalThreadLocalMap实例对应的变量值                  
     public final boolean isSet()                       | 判断这个变量是否在当前线程持有的InternalThreadLocalMap已经设置过          
     public final boolean isSet(InternalThreadLocalMap) | 判断指定的InternalThreadLocalMap是否存在当前FastThreadLocal对应的变量
     public final void remove()                         | 从当前线程的InternalThreadLocalMap中移除这个变量                  

使用FastThreadLocal可以重写它的initialValue和onRemoval方法，前者可以用来设置一个初始变量值，后者则会在调用remove方法移除这个变量的时候调用。

 FastThreadLocal仅持有两个实例变量：

 
```
private final int index;
private final int cleanerFlagIndex;

```
 
##### []()1、初始化阶段

 
```
public FastThreadLocal() {
    index = InternalThreadLocalMap.nextVariableIndex();
    cleanerFlagIndex = InternalThreadLocalMap.nextVariableIndex();
}

```
 每个FastThreadLocal实例在都有唯一的index和cleanerFlagIndex，通过AtomicInteger的getAndIncrease方法获取。index为当前变量在

 
##### []()2、get方法

 get方法用于获取当前线程持有的变量的值

 
```
@SuppressWarnings("unchecked")
public final V get() {
    InternalThreadLocalMap threadLocalMap = InternalThreadLocalMap.get();
    Object v = threadLocalMap.indexedVariable(index);
    if (v != InternalThreadLocalMap.UNSET)
        return (V) v;
    V value = initialize(threadLocalMap);
    registerCleaner(threadLocalMap);
    return value;
}

```
 **get方法可以分为以下几个步骤**

  
  * 获取当前线程(current Thread)持有的InternalThreadLocalMap  这里通过调用InternalThreadLocalMap的静态方法get获取：

 
```
public static InternalThreadLocalMap get() {
    Thread thread = Thread.currentThread();
    //如果当前线程属于FastThreadLocalThread，那么直接获取返回
    if (thread instanceof FastThreadLocalThread) {
        return fastGet((FastThreadLocalThread) thread);
    } else {
        return slowGet();
    }
}

```
 get方法首先获取当前线程对象，然后判定当前线程是不是FastThreadLocalThread类型，如果是，就调用静态方法fastGet来获取InternalThreadLocalMap：

 
```
private static InternalThreadLocalMap fastGet(FastThreadLocalThread thread) {
    InternalThreadLocalMap threadLocalMap = thread.threadLocalMap();
    if (threadLocalMap == null)
        thread.setThreadLocalMap(threadLocalMap = new InternalThreadLocalMap());
    return threadLocalMap;
}

```
 由于每个FastThreadLocalThread对象都持有一个InternalThreadLocalMap实例，所以只需要调用它的threadLocalMap方法获取就行。如果结果为null，那么此时就实例化一个InternalThreadLocalMap并赋给这个FastThreadLocalThread对象，然后方法返回结束。  
 InternalThreadLocalMap的构造方法如下：

 
```
private InternalThreadLocalMap() {
    super(newIndexedVariableTable());
}

public static final Object UNSET = new Object();
private static Object[] newIndexedVariableTable() {
    Object[] array = new Object[32];
    Arrays.fill(array, UNSET);
    return array;
}

```
 InternalThreadLocalMap的构造方法首先调用静态方法newIndexedVariableTable构造一个长度为32的Object数组，并将这个数组的元素全部设置为UNSET，代表尚未设置。然后，调用InternalThreadLocalMap的父类UnpaddedInternalThreadLocalMap的构造方法：

 
```
UnpaddedInternalThreadLocalMap(Object[] indexedVariables) {
    this.indexedVariables = indexedVariables;
}

```
 父类的构造方法则是将这个Object数组赋给成员变量indexedVariables，这个数组就负责存储FastThreadLocal对应的值。

 如果当前线程不属于FastThreadLocalThread，就会调用slowGet方法来获取当前线程持有的InternalThreadLocalMap：

 
```
private static InternalThreadLocalMap slowGet() {
    ThreadLocal<InternalThreadLocalMap> slowThreadLocalMap = UnpaddedInternalThreadLocalMap.slowThreadLocalMap;
    InternalThreadLocalMap ret = slowThreadLocalMap.get();
    if (ret == null) {
        ret = new InternalThreadLocalMap();
        slowThreadLocalMap.set(ret);
    }
    return ret;
}

```
 slowGet的策略是将InternalThreadLocalMap实例通过到java.lang.ThreadLocal实例存储到java.lang.Thread的java.lang.ThreadLocal.ThreadLocalMap中。

  
  * 获取到当前线程的InternalThreadLocalMap后，再调用它的indexedVariable方法，并把当前FastThreadLocal的index成员变量作为参数传入，来获取变量的值  
```
public Object indexedVariable(int index) {
    Object[] lookup = indexedVariables;
    return index < lookup.length? lookup[index] : UNSET;
}

```
 如果index小于这个Object数组的长度，那么直接返回这个数组index位置的元素，否则返回UNSET。

  
  * 如果获取到的变量不是UNSET，那么这个变量就是我们需要的结果，get方法随之返回这个结果并结束。否则，调用initialize方法初始化变量  
```
private V initialize(InternalThreadLocalMap threadLocalMap) {
    V v = null;
    try {
        v = initialValue();
    } catch (Exception e) {
        PlatformDependent.throwException(e);
    }
    threadLocalMap.setIndexedVariable(index, v);
    addToVariablesToRemove(threadLocalMap, this);
    return v;
}

```
 initialize方法中，将initialValue方法的返回值作为变量的初始值。前面已经提到过，initialValue是一个非final的protected方法，默认返回null，可以用来重写来定义变量初始值。如果在initialValue有任何异常发生，那么即使是非运行时异常也会将其强行抛出（通过sun.misc.Unsafe实现）。

 设置完初始值后，接下来的任务就是将初始值存储在该线程持有的InternalThreadLocalMap中，通过调用InternalThreadLocalMap的setIndexedVariable方法来是实现：

 
```
public boolean setIndexedVariable(int index, Object value) {
    Object[] lookup = indexedVariables;
    if (index < lookup.length) {
        Object oldValue = lookup[index];
        lookup[index] = value;
        return oldValue == UNSET;
    } else {
        expandIndexedVariableTableAndSet(index, value);
        return true;
    }
}

```
 如果index小于数组indexedVariables的长度，那么直接将变量添加到这个数组的index中。否则调用expandIndexedVariableTableAndSet方法：

 
```
private void expandIndexedVariableTableAndSet(int index, Object value) {
    Object[] oldArray = indexedVariables;
    final int oldCapacity = oldArray.length;
    int newCapacity = index;
    //设为原长度的2倍，并保证为长度为2次幂
    newCapacity |= newCapacity >>>  1;
    newCapacity |= newCapacity >>>  2;
    newCapacity |= newCapacity >>>  4;
    newCapacity |= newCapacity >>>  8;
    newCapacity |= newCapacity >>> 16;
    newCapacity ++;
	//构造一个新的数组，并将旧数组的元素复制进来
    Object[] newArray = Arrays.copyOf(oldArray, newCapacity);
    //将剩余元素设为UNSET
    Arrays.fill(newArray, oldCapacity, newArray.length, UNSET);
    newArray[index] = value; //设置新变量
    indexedVariables = newArray; //新数组替代旧数组
}

```
 expandIndexedVariableTableAndSet将原来负责保存变量的Object数组扩大到原来的2倍，并将新变量插入到该数组中。

 随后，initialize会调用静态方法addToVariablesToRemove：

 
```
@SuppressWarnings("unchecked")
private static void addToVariablesToRemove(InternalThreadLocalMap threadLocalMap, FastThreadLocal<?> variable) {
	//获取Object数组0位置上的元素
    Object v = threadLocalMap.indexedVariable(variablesToRemoveIndex);
    Set<FastThreadLocal<?>> variablesToRemove;
    //如果这个元素为null或者为UNSET
    if (v == InternalThreadLocalMap.UNSET || v == null) {
    	//构造一个Set集合，将IdentityHashMap的键集合作为一个Set集合，然后将这个Set放置于Object数组的0号位
        variablesToRemove = Collections.newSetFromMap(new IdentityHashMap<FastThreadLocal<?>, Boolean>());
        threadLocalMap.setIndexedVariable(variablesToRemoveIndex, variablesToRemove);
    } else {
        variablesToRemove = (Set<FastThreadLocal<?>>) v;
    }
    variablesToRemove.add(variable); //将变量添加到一个
}

```
 addToVariablesToRemove方法首先获取Object数组indexedVariables的variablesToRemoveIndex位元素  
 variablesToRemoveIndex是一个int型变量，在类加载阶段初始化：

 
```
private static final int variablesToRemoveIndex = InternalThreadLocalMap.nextVariableIndex();

```
 一般来说，variablesToRemoveIndex恒为0。

 如果数组indexedVariables的0号位元素为null或者为UNSET，那么构造一个IdentityHashMap（这个Map判断键相同的方式是通过==判断，而不是equals方法），并以FastThreadLocal为键，并通过Collections工具类的newSetFromMap将IdentityHashMap键集合作为一个Set集合，并将这个Set集合放入数组indexedVariables的0号位。然后，将当前的FastThreadLocal添加到这个集合中。

 所以，**对于每个InternalThreadLocalMap的indexedVariables数组，它的第一个元素都是这个Set集合**。

 经过上述一系列步骤后，initialize方法才算执行完成。

  
  * 调用registerCleaner方法尝试向ObjectCleaner注册一个回调任务，当线程死亡的时候，能够及时回收这个线程持有的InternalThreadLocalMap  
```
private void registerCleaner(final InternalThreadLocalMap threadLocalMap) {
    Thread current = Thread.currentThread();
    //如果在构造FastThreadLocalThread的时候指定了Runnable，或者cleanerFlagIndex位置的元素不为UNSET，那么方法退出
    if (FastThreadLocalThread.willCleanupFastThreadLocals(current) ||
        threadLocalMap.indexedVariable(cleanerFlagIndex) != InternalThreadLocalMap.UNSET) {
        return;
    }
    //将数组indexedVariables的cleanerFlagIndex位元素设为true
    threadLocalMap.setIndexedVariable(cleanerFlagIndex, Boolean.TRUE);
    ObjectCleaner.register(current, new Runnable() {
        @Override
        public void run() {
            remove(threadLocalMap);
        }
    });
}

```
 如果当前线程不属于FastThreadLocalThread，或者在构造FastThreadLocalThread时指定了Runnable任务，或者cleanerFlagIndex位置上的元素不为UNSET，registerCleaner方法随即会返回结束。

 如果满足上述条件，registerCleaner方法将数组indexedVariables的cleanerFlagIndex位设为布尔类型的true，然后向ObjectCleaner注册一个回调任务，该任务负责清除InternalThreadLocalMap中的引用，会在线程死亡后被触发执行。

 ObjectCleaner将线程对象被一个弱引用对象WeakReference所引用，然后将这个弱引用对象保存到一个Set集合中，当线程死亡并且失去其它引用后，在下次系统进行垃圾回收的时候，这个线程对象随即会被GC，然后，它的弱引用对象会被加入到引用队列。ObjectCleaner维护的一个线程会不断尝试获取引用队列中的Reference对象，当获取到这个Reference对象后，就会执行我们注册的回调任务（回调任务就保存在引用对象中）。  
 关于Java的Reference和ReferenceQueue机制，可以参考我的博客：[https://blog.csdn.net/abc123lzf/article/details/82381719](https://blog.csdn.net/abc123lzf/article/details/82381719)

 至此，get方法执行完毕。

 看完上述代码，可以总结出indexedVariables数组的结构如下：  
 （1）对于构造阶段未指定Runnable任务的FastThreadLocal对象：  
 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20181101181316668.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FiYzEyM2x6Zg==,size_16,color_FFFFFF,t_70)

 （2）对于构造阶段指定Runnable任务的FastThreadLocal对象：  
 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20181101181400905.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FiYzEyM2x6Zg==,size_16,color_FFFFFF,t_70)

 （3）对于一般的Thread对象：  
 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20181101181624983.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FiYzEyM2x6Zg==,size_16,color_FFFFFF,t_70)

 
##### []()2、set方法

 set方法用于设置线程私有的变量值。当传入的值为InternalThreadLocalMap.UNSET时，代表将这个变量从InternalThreadLocalMap中删除。

 
```
public final void set(V value) {
    if (value != InternalThreadLocalMap.UNSET) {
        InternalThreadLocalMap threadLocalMap = InternalThreadLocalMap.get();
        if (setKnownNotUnset(threadLocalMap, value)) {
            registerCleaner(threadLocalMap);
        }
    } else {
        remove();
    }
}

```
 set方法分为以下几个步骤：

  
  * 获取当前线程持有的InternalThreadLocalMap实例。  刚才已经分析过这个方法，这里不再赘述

  
  * 将这个变量添加到这个InternalThreadLocalMap中  
```
private boolean setKnownNotUnset(InternalThreadLocalMap threadLocalMap, V value) {
    if (threadLocalMap.setIndexedVariable(index, value)) {
        addToVariablesToRemove(threadLocalMap, this);
        return true;
    }
    return false;
}

```
 这里调用setIndexedVariable方法将变量插入到indexedVariable数组的index位置中，设置成功后，再调用addToVariablesToRemove方法设置存储FastThreaLocal的Set集合（如果尚未设置的话）。  
 上面这两个方法同样都已经分析过。

  
  * 向ObjectCleaner注册一个回调任务，参考get方法  
##### []()3、remove方法

 remove方法可以将变量从InternalThreadLocalMap移除。

 
```
public final void remove() {
    remove(InternalThreadLocalMap.getIfSet());
}
@SuppressWarnings("unchecked")
public final void remove(InternalThreadLocalMap threadLocalMap) {
    if (threadLocalMap == null)
        return;
    Object v = threadLocalMap.removeIndexedVariable(index);
    removeFromVariablesToRemove(threadLocalMap, this);
    if (v != InternalThreadLocalMap.UNSET) {
        try {
            onRemoval((V) v);
        } catch (Exception e) {
            PlatformDependent.throwException(e);
        }
    }
}

```
 remove方法分为以下几个步骤：

  
  * 获取当前线程对象所属的InternalThreadLocalMap  
```
public static InternalThreadLocalMap getIfSet() {
    Thread thread = Thread.currentThread();
    if (thread instanceof FastThreadLocalThread)
        return ((FastThreadLocalThread) thread).threadLocalMap();
    return slowThreadLocalMap.get();
}

```
 如果当前线程为FastThreadLocalThread，那么直接通过它的threadLocalMap方法获取。否则通过JDK的ThreadLocal获取。

  
  * 将这个变量从InternalThreadLocalMap持有的indexedVariables数组移除  
```
public Object removeIndexedVariable(int index) {
    Object[] lookup = indexedVariables;
    if (index < lookup.length) {
        Object v = lookup[index];
        lookup[index] = UNSET;
        return v;
    } else {
        return UNSET;
    }
}

```
 该方法将这个变量所在的数组index位设为UNSET，然后返回这个变量的值。

  
  * 将indexedVariables数组的0号位的Set集合中移除当前的FastThreadLocal对象  
```
private static void removeFromVariablesToRemove(
        InternalThreadLocalMap threadLocalMap, FastThreadLocal<?> variable) {
    Object v = threadLocalMap.indexedVariable(variablesToRemoveIndex);
    if (v == InternalThreadLocalMap.UNSET || v == null)
        return;
    @SuppressWarnings("unchecked")
    Set<FastThreadLocal<?>> variablesToRemove = (Set<FastThreadLocal<?>>) v;
    variablesToRemove.remove(variable); //移除这个FastThreadLocal
}

```
  
  * 最后调用onRemoval方法并传入变量的值。该方法默认实现为空，可以重写它来实现自己的任务逻辑。    
  