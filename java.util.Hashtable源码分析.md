---
title: java.util.Hashtable源码分析
date: 2018-09-01 20:15:50
tags: CSDN迁移
---
 版权声明：尊重原创，转载请注明出处 [ https://blog.csdn.net/abc123lzf/article/details/82284981]( https://blog.csdn.net/abc123lzf/article/details/82284981)   
  ### 一、引言

 Hashtable在JDK中是一个古老的Map实现类，早在JDK1.0就已经存在，其内部实现。在新的代码中已经不推荐再使用Hashtable存储键值对，而是使用HashMap（无须多线程操作）或ConcurrentHashMap（需要多线程操作）。   
 Hashtable是一个线程安全的Map类，其大多数方法都加了synchronized关键字，意味着同一时间只能有一个线程对它的实例进行访问，而不是像ConcurrentHashMap使用了分段锁技术，可以支持多线程并发修改、访问。

 Hashtable的继承关系类图：   
 ![这里写图片描述](https://img-blog.csdn.net/2018090110491296?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FiYzEyM2x6Zg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)   
 Hashtable的父类是Dictionary，这个父类同样是一个JDK1.0就已经存在的抽象类（Map接口于JDK1.2引入）   
 Hashtable有一个常用的子类：Properties（JDK1.0就已存在），这个类负责保存键和值都是String类型的键值对，一般用来存放系统配置信息（如调用System.getProperties()），也可用来读取properties配置文件并将信息载入到这个类的实例中。

 在阅读Hashtable的源码前，建议先阅读HashMap的源代码。   
 可以参考我的博客：[https://blog.csdn.net/abc123lzf/article/details/82227321](https://blog.csdn.net/abc123lzf/article/details/82227321)   
 因为已经不再推荐使用Hashtable，所以这篇文章只会大致分析Hashtable与HashMap的一些异同点

 
--------
 
### 二、源码分析

 下面是Hashtable的类定义、实例变量和静态常量。

 
```
public class Hashtable<K,V> extends Dictionary<K,V>
    implements Map<K,V>, Cloneable, Serializable {
    //table最大长度
    private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;
    //存储键值对的数组
    private transient Entry<?,?>[] table;
    //键值对的数量
    private transient int count;
    //当count大于这个数时，会对table进行扩容，我们称其为临界值
    private int threshold;
    //负载因子，默认0.75
    private float loadFactor;
    //修改次数，主要用于检测在迭代器遍历元素时是否调用外部方法修改Hashtable
    private transient int modCount = 0;
    //键缓存，用于存储keySet方法返回的内部类对象
    private transient volatile Set<K> keySet;
    //entrySet缓存，该Set包含所有键值对
    private transient volatile Set<Map.Entry<K,V>> entrySet;
    //值缓存，包含所有键值对中的值
    private transient volatile Collection<V> values;
}
```
 Hashtable的成员变量和HashMap差不多，都有一个临界值、负载因子和modCount。   
 Hashtable使用一个内部类Entry代表一个键值对，Entry继承了Map.Entry接口。

 
```
private static class Entry<K,V> implements Map.Entry<K,V> {
    final int hash; //键的hashCode
    final K key; //键
    V value; //值
    Entry<K,V> next; //下一个元素，通过链表来解决哈希冲突

    protected Entry(int hash, K key, V value, Entry<K,V> next) {
        this.hash = hash;
        this.key =  key;
        this.value = value;
        this.next = next;
    }
    //克隆Entry，包括Entry的next属性
    @SuppressWarnings("unchecked")
    protected Object clone() {
            return new Entry<>(hash, key, value,
                    (next==null ? null : (Entry<K,V>) next.clone()));
    }
    //获取键和值的方法
    public K getKey() { return key; }
    public V getValue() { return value; }
    //设置新的值并返回旧值
    public V setValue(V value) {
        if (value == null)
            throw new NullPointerException();
        V oldValue = this.value;
        this.value = value;
        return oldValue;
    }
    //通过键和值对象的equals判断两个键值对是否相等
    public boolean equals(Object o) {
        if (!(o instanceof Map.Entry))
            return false;
        Map.Entry<?,?> e = (Map.Entry<?,?>)o;

        return (key==null ? e.getKey()==null : key.equals(e.getKey())) &&
               (value==null ? e.getValue()==null : value.equals(e.getValue()));
   }
   //键和值的hashCode进行按位异或运算，注意不要和成员变量hash混淆
   public int hashCode() {
       return hash ^ Objects.hashCode(value);
   }
   //省略toString方法
}
```
 Hashtable.Entry和HashMap.Node的实现大致相同。区别在于Hashtable.Entry重写了clone方法，在hashCode方法的实现上，HashMap会重新调用键和值的hashCode方法并进行异或运算。

 
--------
 
##### （1）构造方法

 和HashMap一样，Hashtable提供了4个public构造方法且参数类型相同。

 
```
public Hashtable() {
    this(11, 0.75f);
}

//initialCapacity为table(桶)长度
public Hashtable(int initialCapacity) {
    this(initialCapacity, 0.75f);
}

//loadFactor为负载因子
public Hashtable(int initialCapacity, float loadFactor) {
    if (initialCapacity < 0)
        throw new IllegalArgumentException("Illegal Capacity: "+ initialCapacity);
    if (loadFactor <= 0 || Float.isNaN(loadFactor))
        throw new IllegalArgumentException("Illegal Load: "+loadFactor);

    if (initialCapacity==0)
        initialCapacity = 1;
    this.loadFactor = loadFactor;
    table = new Entry<?,?>[initialCapacity];
    //将临界值设置为负载因子乘以桶大小(table数组长度)
    threshold = (int)Math.min(initialCapacity * loadFactor, MAX_ARRAY_SIZE + 1);
}
```
 在构造Hashtable的时候，可以指定桶的大小和负载因子。和HashMap不一样的地方是：HashMap的桶大小只能为2次幂，而Hashtable可以指定桶的长度为1~Integer.MAX_VALUE - 8范围内的任意整数，对于Hashtable而言最好将桶的长度设置为素数，以减少哈希冲突概率。

 对于无参构造方法，Hashtable默认将桶大小设置为11（HashMap为16），负载因子设置为0.75（和HashMap一样），并且Hashtable会分配一个长度最小为1的Entry数组给table变量，而HashMap在调用无参构造方法的时候不会分配数组给table，只有在调用了put等方法后才会分配。

 
```
public Hashtable(Map<? extends K, ? extends V> t) {
    this(Math.max(2*t.size(), 11), 0.75f);
    putAll(t);
}
```
 和HashMap一样，构造Hashtable的时候可以指定一个Map，可以把这个Map中所有的键值对**引用**复制到这个Hashtable中。

 
##### （2）put方法

 Hashtable不支持将键为null或值为null的键值对添加到Hashtable，尝试添加这样的键值对会抛出NullPointerException异常。

 
```
public synchronized V put(K key, V value) {
    if (value == null) {
        throw new NullPointerException();
    }

    Entry<?,?> tab[] = table;
    //通过将key的hashCode和table长度进行模运算定位桶的位置
    int hash = key.hashCode();
    int index = (hash & 0x7FFFFFFF) % tab.length;
    @SuppressWarnings("unchecked")
    Entry<K,V> entry = (Entry<K,V>)tab[index];
    //遍历这个桶中的链表，查找有没有相同的key
    for(; entry != null ; entry = entry.next) {
        //如果这个键已经存在于table，那么这个键值对的值会被替换，然后返回旧值
        if ((entry.hash == hash) && entry.key.equals(key)) {
            V old = entry.value;
            entry.value = value;
            return old;
        }
    }
    //调用addEntry方法添加元素
    addEntry(hash, key, value, index);
    return null;
}

private void addEntry(int hash, K key, V value, int index) {
    modCount++;
    Entry<?,?> tab[] = table;
    //如果键值对数量超过临界值那么进行rehash操作
    if (count >= threshold) {
        rehash(); //rehash方法策略我们稍作讨论
        tab = table;
        //重新定位桶的位置
        hash = key.hashCode();
        index = (hash & 0x7FFFFFFF) % tab.length;
    }
    @SuppressWarnings("unchecked")
    Entry<K,V> e = (Entry<K,V>) tab[index];
    //如果这个桶没有其它键值对，那么直接插入到数组(此时e为null)，如果有其它键值对，那么插入到链表头部
    tab[index] = new Entry<>(hash, key, value, e);
    count++;
}
```
 Hashtable在定位桶的时候，先将key的哈希值和0x7FFFFFFF进行与运算，再进行模运算。   
 0x7FFFFFFF代表int类型的最大值（二进制码0111 1111 1111 1111 1111 1111 1111 1111），如果key的哈希值为负数（最高位为1），那么会将其转换为一个正数，防止模运算算出负数导致ArrayIndexOutOfBoundsException异常。

 **Hashtable和HashMap在解决哈希冲突的时候有以下几个不同点：**   
 1、Hashtable解决哈希冲突的策略是在这个桶中构造一个链表，并插入到链表的头部。而HashMap是插入到链表尾端（当这个链表长度小于8的时候）。   
 2、当一个桶中的链表长度大于8的时候且table数组长度大于64的时候，HashMap会将这个链表转换为红黑树，将查找最坏时间复杂度由O(n)降到O(log n)

 
##### （3）内部方法：rehash

 当Hashtable检测到键值对数量大于等于临界值时，会调用这个方法扩大桶的大小以减少哈希冲突。

 
```
@SuppressWarnings("unchecked")
protected void rehash() {
    int oldCapacity = table.length;
    Entry<?,?>[] oldMap = table;

    //新的table长度为原来的2倍+1,如果大于MAX_ARRAY_SIZE则不进行扩容
    int newCapacity = (oldCapacity << 1) + 1;
    if (newCapacity - MAX_ARRAY_SIZE > 0) {
        if (oldCapacity == MAX_ARRAY_SIZE)
            return;
        newCapacity = MAX_ARRAY_SIZE;
    }

    Entry<?,?>[] newMap = new Entry<?,?>[newCapacity];
    modCount++;
    //新的临界值为新的桶大小乘以负载因子
    threshold = (int)Math.min(newCapacity * loadFactor, MAX_ARRAY_SIZE + 1);
    //将新的Entry数组赋给table成员变量
    table = newMap;

    //遍历旧的table
    for (int i = oldCapacity ; i-- > 0 ;) {
        //遍历这个桶中所有的键值对
        for (Entry<K,V> old = (Entry<K,V>)oldMap[i] ; old != null ; ) {
            Entry<K,V> e = old;
            old = old.next;
            //定位到新的table数组，如果存在哈希冲突则添加到链表头部
            int index = (e.hash & 0x7FFFFFFF) % newCapacity;
            e.next = (Entry<K,V>)newMap[index];
            newMap[index] = e;
        }
    }
}
```
 Hashtable的resize大小为桶大小乘以2加上1，而HashMap为桶大小的2倍（永远为2的次幂）。

 
##### （4）keySet、entrySet、values方法

 
```
public Set<K> keySet() {
    if (keySet == null)
        keySet = Collections.synchronizedSet(new KeySet(), this);
    return keySet;
}

public Set<Map.Entry<K,V>> entrySet() {
    if (entrySet==null)
        entrySet = Collections.synchronizedSet(new EntrySet(), this);
    return entrySet;
}

public Collection<V> values() {
    if (values==null)
        values = Collections.synchronizedCollection(new ValueCollection(), this);
    return values;
}
```
 KeySet、EntrySet、ValueCollection都是Hashtable的非静态内部类。keySet、entrySet、values方法通过Collections的synchronizedX方法将其包装成线程安全的类并返回，这些类本质上都是通过调用了外部类的方法来实现其功能，如果将返回的集合中的键值对进行修改，那么Hashtable中的键值对也会随之发生改变，在HashMap中也是类似的原理。

 
### 三、序列化和反序列化策略

 Hashtable实现了writeObject方法和readObject方法实现了自己的序列化和反序列化策略   
 序列化：

 
```
private void writeObject(ObjectOutputStream s) throws IOException {
    Entry<Object, Object> entryStack = null;

    synchronized (this) {
        //写入负载因子和临界值
        s.defaultWriteObject();
        //写入table长度
        s.writeInt(table.length);
        //写入键值对数量
        s.writeInt(count);
        //将Hashtable中所有的键值对组成一个链表放入entryStack中
        for (int index = 0; index < table.length; index++) {
            Entry<?,?> entry = table[index];
            while (entry != null) {
                entryStack = new Entry<>(0, entry.key, entry.value, entryStack);
                entry = entry.next;
            }
        }
    }
    //依次写入键和值
    while (entryStack != null) {
        s.writeObject(entryStack.key);
        s.writeObject(entryStack.value);
        entryStack = entryStack.next;
    }
}
```
 反序列化：

 
```
private void readObject(ObjectInputStream s) throws IOException, ClassNotFoundException {
    s.defaultReadObject();

    if (loadFactor <= 0 || Float.isNaN(loadFactor))
        throw new StreamCorruptedException("Illegal Load: " + loadFactor);
    //读入原table长度
    int origlength = s.readInt();
    //读入键值对数量
    int elements = s.readInt();

    if (elements < 0)
        throw new StreamCorruptedException("Illegal # of Elements: " + elements);
    //比较原table长度和键值对数量/负载因子+1大小，以较大的为origlenth
    origlength = Math.max(origlength, (int)(elements / loadFactor) + 1);
    int length = (int)((elements + elements / 20) / loadFactor) + 3;
    if (length > elements && (length & 1) == 0)
        length--;
    //比较length和origlength长度，以较小者作为table的长度
    length = Math.min(length, origlength);
    table = new Entry<?,?>[length];
    threshold = (int)Math.min(length * loadFactor, MAX_ARRAY_SIZE + 1);
    count = 0;

    //还原键值对
    for (; elements > 0; elements--) {
        @SuppressWarnings("unchecked")
        K key = (K)s.readObject();
        @SuppressWarnings("unchecked")
        V value = (V)s.readObject();
        reconstitutionPut(table, key, value);
    }
}

private void reconstitutionPut(Entry<?,?>[] tab, K key, V value) 
        throws StreamCorruptedException {
    if (value == null) {
        throw new java.io.StreamCorruptedException();
    }
    //定位桶的位置
    int hash = key.hashCode();
    int index = (hash & 0x7FFFFFFF) % tab.length;

    //如果存在了相同的键，则说明对象流有问题
    for (Entry<?,?> e = tab[index] ; e != null ; e = e.next) {
        if ((e.hash == hash) && e.key.equals(key)) {
            throw new java.io.StreamCorruptedException();
        }
    }

    @SuppressWarnings("unchecked")
    Entry<K,V> e = (Entry<K,V>)tab[index];
    //重新构造Entry对象
    tab[index] = new Entry<>(hash, key, value, e);
    count++;
}
```
   
  