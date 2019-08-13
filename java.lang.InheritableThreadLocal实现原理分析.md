---
title: java.lang.InheritableThreadLocal实现原理分析
date: 2018-08-24 15:09:37
tags: CSDN迁移
---
 版权声明：尊重原创，转载请注明出处 [ https://blog.csdn.net/abc123lzf/article/details/82016143]( https://blog.csdn.net/abc123lzf/article/details/82016143)   
  在了解InheritableThreadLocal之前，需要先理解ThreadLocal的实现原理，关于ThreadLocal文章可以参考我的博客：   
 [https://blog.csdn.net/abc123lzf/article/details/81978210](https://blog.csdn.net/abc123lzf/article/details/81978210)

 
### 一、前言

 在了解InheritableThreadLocal前，我们先来回顾下InheritableThreadLocal的使用方法   
 先看下面这个例子：

 
```
public class Test {

    static ThreadLocal<Integer> itl = new InheritableThreadLocal<Integer>() {
        @Override
        protected Integer initialValue() {
            return 0;
        }
    };

    static class Print implements Runnable {
        @Override
        public void run() {
            System.out.println(Thread.currentThread().getName() + " " + itl.get());
        }
    }

    public static void main(String[] args) {
        System.out.println(Thread.currentThread().getName() + " " + itl.get());
        itl.set(itl.get() + 1);
        new Thread(new Print()).start();
    }
}
```
 运行后的输出结果为：   
 main 0   
 Thread-0 1

 InheritableThreadLocal可以使子线程**在创建的时候**获取到父线程的线程私有变量的值。需要注意的是，在子线程创建之后，如果子线程修改了变量的值，父线程是无法获知子线程修改了该变量的。所以，InheritableThreadLocal和ThreadLocal的区别在于创建子线程时的操作。

 
### 二、实现原理分析

 先看InheritableThreadLocal的源码：

 
```
public class InheritableThreadLocal<T> extends ThreadLocal<T> {

    protected T childValue(T parentValue) {
        return parentValue;
    }

    ThreadLocalMap getMap(Thread t) {
       return t.inheritableThreadLocals;
    }

    void createMap(Thread t, T firstValue) {
        t.inheritableThreadLocals = new ThreadLocalMap(this, firstValue);
    }
}
```
 InheritableThreadLocal继承了ThreadLocal类，重写了ThreadLocal的三个方法：childValue、getMap、createMap方法。

 同样，我们先从get方法开始分析：

 
```
public T get() {
    Thread t = Thread.currentThread();
    //获取当前线程对象t的inheritableThreadLocals成员变量
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        //根据当前InheritableThreadLocal为键获取Entry
        ThreadLocalMap.Entry e = map.getEntry(this);
        //如果查找结果不为null则返回
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    //否则设置初始值
    return setInitialValue();
}
```
 和ThreadLocal不同的是getMap，在ThreadLocal类的getMap实现中，返回的是当前Thread对象的threadLocals成员变量，而在InheritableThreadLocal中，由于重写了getMap方法，返回的是inheritableThreadLocals成员变量，它同样是Thread对象的ThreadLocalMap类型的成员变量。即：

 
```
/* ThreadLocal values pertaining to this thread. This map is maintained
* by the ThreadLocal class. */
ThreadLocal.ThreadLocalMap threadLocals = null;

/*
* InheritableThreadLocal values pertaining to this thread. This map is
* maintained by the InheritableThreadLocal class.
*/
ThreadLocal.ThreadLocalMap inheritableThreadLocals = null;
```
 这就是为什么InheritableThreadLocal要重写getMap方法和createMap方法了，因为在获取Map和创建Map时赋值的变量是inheritableThreadLocals而不是threadLocals。

 先回顾下：InheritableThreadLocal和ThreadLocal的区别在于创建子线程时的操作。   
 所以，我们现在来关注Thread的构造方法。

 查看Thread类的源码，虽然Thread类提供了很多构造器，但其实都调用了一个方法：init：

 
```
private void init(ThreadGroup g, Runnable target, String name,
                      long stackSize, AccessControlContext acc,
                      boolean inheritThreadLocals) {
    if (name == null) {
        throw new NullPointerException("name cannot be null");
    }

    this.name = name;
    //获取父线程对象
    Thread parent = currentThread();
    SecurityManager security = System.getSecurityManager();
    if (g == null) {

        if (security != null) {
            g = security.getThreadGroup();
        }

        if (g == null) {
            g = parent.getThreadGroup();
        }
    }

    g.checkAccess();

    if (security != null) {
        if (isCCLOverridden(getClass())) {
            security.checkPermission(SUBCLASS_IMPLEMENTATION_PERMISSION);
        }
    }

    g.addUnstarted();

    this.group = g;
    this.daemon = parent.isDaemon();
    this.priority = parent.getPriority();

    if (security == null || isCCLOverridden(parent.getClass()))
        this.contextClassLoader = parent.getContextClassLoader();
    else
        this.contextClassLoader = parent.contextClassLoader;

    this.inheritedAccessControlContext =
    acc != null ? acc : AccessController.getContext();
    this.target = target;
    setPriority(priority);
    //注意这里
    if (inheritThreadLocals && parent.inheritableThreadLocals != null)
        this.inheritableThreadLocals = ThreadLocal.createInheritedMap(parent.inheritableThreadLocals);

    this.stackSize = stackSize;

    tid = nextThreadID();
}
```
 在init方法中，会首先获取构造当前线程的线程对象（即父线程）。   
 再来看这一部分的代码：

 
```
if (inheritThreadLocals && parent.inheritableThreadLocals != null)
        this.inheritableThreadLocals = ThreadLocal.createInheritedMap(parent.inheritableThreadLocals);
```
 首先这里会判断是否支持inheritThreadLocals（这里的inheritThreadLocals是一个boolean类型的局部变量，一般情况下都会为true）和判断父线程的inheritableThreadLocals成员变量是否为null，如果满足条件则调用ThreadLocal的静态方法createInheritedMap并将父线程的inheritableThreadLocals作为参数传入该方法，以此来创建一个ThreadLocalMap并赋值给子线程的inheritableThreadLocals成员变量。

 下面来看ThreadLocal静态方法createInheritedMap

 
```
static ThreadLocalMap createInheritedMap(ThreadLocalMap parentMap) {
    return new ThreadLocalMap(parentMap);
}
```
 该静态方法构造了一个新的ThreadLocalMap

 下面是上述静态方法调用的ThreadLocalMap的构造器：

 
```
private ThreadLocalMap(ThreadLocalMap parentMap) {
    Entry[] parentTable = parentMap.table;
    int len = parentTable.length;
    setThreshold(len);
    table = new Entry[len];

    for (int j = 0; j < len; j++) {
        Entry e = parentTable[j];
        if (e != null) {
            @SuppressWarnings("unchecked")
            ThreadLocal<Object> key = (ThreadLocal<Object>) e.get();
            if (key != null) {
                Object value = key.childValue(e.value);
                Entry c = new Entry(key, value);
                int h = key.threadLocalHashCode & (len - 1);
                while (table[h] != null)
                    h = nextIndex(h, len);
                table[h] = c;
                size++;
            }
        }
    }
}
```
 上述构造方法首先获取父线程inheritableThreadLocals成员变量所有的Entry，通过一个for循环，**拷贝**(浅拷贝)该Entry并添加到子线程成员变量inheritableThreadLocals的ThreadLocalMap中。

 最后，再来看看InheritableThreadLocal的childValue方法吧   
 childValue默认实现是返回传入的变量，如果需要根据父线程变量的值返回子线程的初始值，则需要重写该方法，该方法会在构造ThreadLocalMap时拷贝Entry过程中调用（就是上面那个构造方法）。

 如果文章有错误或者对该文章有疑问，欢迎在评论区留言。

   
  