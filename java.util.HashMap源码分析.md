---
title: java.util.HashMap源码分析
date: 2018-09-01 09:58:29
tags: CSDN迁移
---
 版权声明：尊重原创，转载请注明出处 [ https://blog.csdn.net/abc123lzf/article/details/82227321]( https://blog.csdn.net/abc123lzf/article/details/82227321)   
  ### 一、概述

 HashMap是Java中常用的Map实现，它提供了一种键到值的映射，可以高效地查找键值对。其内部是一种基于拉链法实现的哈希表。   
 ![这里写图片描述](https://images0.cnblogs.com/blog/631817/201502/271613146279675.x-png)   
 图片来自百度百科。

 HashMap的继承关系图：   
 ![这里写图片描述](https://img-blog.csdn.net/20180830200356331?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FiYzEyM2x6Zg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)   
 HashMap支持插入键为null，值为null的键值对，但最多只能保存一个键为null的键值对，可以保存多个值为null的键值对。   
 HashMap不是一个线程安全的类，如果需要并发操作Map应当使用ConcurrentHashMap。

 在学习HashMap源码之前，请先了解哈希表的原理和HashMap的基本使用。

 
--------
 
### 二、Map接口

 在了解HashMap前，我们先来回顾一下Map接口

 
```
public interface Map<K, V> {
    //返回这个Map持有的键值对数量
    int size();
    //这个Map的键值对数量是不是为0
    boolean isEmpty();
    //返回是否包含键key
    boolean containsKey(Object key);
    //返回是否包含值value
    boolean containsValue(Object value);
    //通过键key获取值，不存在则返回null
    V get(Object key);
    //将键值对加入到Map中，如果key在Map中已经存在，则会被替换并且返回这个旧值，不存在则返回null
    V put(K key, V value);
    //通过键key移除键值对，如果这个键值对存在则返回值，若不存在则返回null
    V remove(Object key);
    //添加一个Map中包含的所有键值对
    void putAll(Map<? extends K, ? extends V> m);
    //清除所有键值对
    void clear();
    //返回所有键值对中包含所有键的Set集合
    Set<K> keySet();
    //返回所有键值对中包含所有值的Collection集合
    Collection<V> values();
    //返回所有键值对Set集合
    Set<Map.Entry<K, V>> entrySet();

    //单个键值对接口定义
    public static interface Entry<K,V> {
        //获得键
        K getKey();
        //获得值
        V getValue();
        //设置一个新值并返回旧值
        V setValue(V value);
        //判断两个键值对是否一致
        boolean equals(Object o);
        int hashCode();
        //省略静态方法(JDK 1.8新增)...
    }

    boolean equals(Object o);
    int hashCode();

    //以下方法为JDK1.8添加的默认方法，其具体实现省略以减少篇幅
    //通过键key查找值，如果存在返回对应的value，不存在返回defaultValue
    default V getOrDefault(Object key, V defaultValue) { /*...*/ }
    //传入一个BiConsumer函数实现，并对每一个键值对执行函数中的操作
    default void forEach(BiConsumer<? super K, ? super V> action) {/*...*/}
    //传入一个BiFunction函数实现，并对每一个键值对的值替换成function的返回值（function接收两个参数：键和值）
    default void replaceAll(BiFunction<? super K, ? super V, ? extends V> function) {/*...*/}
    //只有当Map中不存在键key才会插入这个键值对
    default V putIfAbsent(K key, V value) {/*...*/}
    //删除键值对，当这个键值对存在且值也等于value时，删除成功返回true
    default boolean remove(Object key, Object value) {/*...*/}
    //替换键值对的值，只有当oldValue等于newValue且Map中包含了这个键值对才会替换
    default boolean replace(K key, V oldValue, V newValue) {/*...*/}
    //替换键值对中的值为value，返回旧值，若不存在返回null
    default V replace(K key, V value) {/*...*/}
    //当键key对应的键值对不存在时，执行函数mappingFunction中的操作(key作参数)
    //返回值作为键值对的值，然后将这个键值对添加到Map中
    default V computeIfAbsent(K key, Function<? super K, ? extends V> mappingFunction) {/*...*/}
    //和上述方法相反，当键存在时才会执行函数的操作并添加到Map
    default V computeIfPresent(K key,
            BiFunction<? super K, ? super V, ? extends V> remappingFunction){/*...*/}
    //将键值对作为参数传入函数，如果返回值为null且键值对存在则移除这个键值对，如果返回值不为null，则调用put添加该键值对
    default V compute(K key,
            BiFunction<? super K, ? super V, ? extends V> remappingFunction){/*...*/}

    default V merge(K key, V value,
            BiFunction<? super V, ? super V, ? extends V> remappingFunction){/*...*/}
}
```
 JDK1.8t提供了很多默认方法，这些方法可以传入函数实现，可以直接对这些元素调用函数中的方法，而不必向以前那样取出一个值再执行操作。

 
--------
 
### 三、源码分析

 我们先来看HashMap的类定义和静态常量

 
```
public class HashMap<K,V> extends AbstractMap<K,V>
    implements Map<K,V>, Cloneable, Serializable {
    //构建Map时默认的桶大小：16
    static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; 
    //最大的桶大小：2^30
    static final int MAXIMUM_CAPACITY = 1 << 30;
    //默认负载因子，当Map中包含的元素/Map桶大小 > 0.75时，会扩大桶的大小
    static final float DEFAULT_LOAD_FACTOR = 0.75f;
    //当一个桶包含的元素大于等于8且桶的数量大于64时，会将链表转换成红黑树(JDK1.8引入)
    static final int TREEIFY_THRESHOLD = 8;
    //在调整桶大小时，如果一个桶元素小于等于6，则采用链表(JDK 1.8)
    static final int UNTREEIFY_THRESHOLD = 6;
    //当桶的大小小于64时，即使某个桶的链表超过8也不会转换成红黑树(JDK1.8)
    static final int MIN_TREEIFY_CAPACITY = 64;
}
```
 HashMap继承了AbstractMap，实现了Cloneable、Serializable接口，和很多Java集合类一样。   
 在JDK1.8中HashMap做了一次大幅度的更新，就是当一个桶中的元素大于等于8且桶的数量大于64时，会将这个桶中的链表转换成红黑树，可以有效地减少查找的时间开销。转换成红黑树后，如果这个桶的元素又小于等于6，则HashMap会自动将红黑树转换回链表。   
 这样，HashMap在JDK1.8以前最坏的添加、查找操作时间复杂度为O(N)，更新到JDK1.8以后，HashMap最坏添加、查找操作时间复杂度降低到O(log N)   
 


--------
   
 HashMap的实例变量：

 
```
//存储键值对的数组
transient Node<K,V>[] table;
//键值对Set缓存，在频繁调用entrySet方法时可以减少时间开销
transient Set<Map.Entry<K,V>> entrySet;
//键值对数量
transient int size;
//Map进行修改的次数
transient int modCount;
//当size大于这个数时，HashMap会进行resize操作，threshold = capacity * loadFactor
//在下文我们称它为临界值
int threshold;
//负载因子，默认为0.75
final float loadFactor;
//父类AbstractMap中的变量，用于keySet方法，作为缓存使用
transient Set<K> keySet;
//父类AbstractMap中的变量，用于values方法，作为缓存使用
transient Collection<V> values;
```
 实例变量table是HashMap的桶数组，通过一个内部类Node来记录键值对的信息。   
 HashMap.Node类实现了Map.Entry接口，它维持了一个链表，当遇到哈希碰撞时就通过链表来保存这个元素。 在JDK1.8中，除了有链表实现的Node，还有红黑树的Node实现：HashMap.TreeNode，TreeNode间接继承了HashMap.Node类。

 下面是HashMap.Node类源码：

 
```
static class Node<K,V> implements Map.Entry<K,V> {
    //hash值，这个值等于key.hashCode()
    final int hash;
    //键值对
    final K key;
    V value;
    //指向下一个链表结点
    Node<K,V> next;

    Node(int hash, K key, V value, Node<K,V> next) {
        this.hash = hash;
        this.key = key;
        this.value = value;
        this.next = next;
    }

    public final K getKey()        { return key; }
    public final V getValue()      { return value; }
    public final String toString() { return key + "=" + value; }
    //注意不要和成员变量hash混淆，这个方法是整个键值对的hashCode
    public final int hashCode() {
        return Objects.hashCode(key) ^ Objects.hashCode(value);
    }
    //当且仅当两个键值对的键和值相等时(equals方法或==)返回true
    public final boolean equals(Object o) {
        if (o == this)
            return true;
        if (o instanceof Map.Entry) {
            Map.Entry<?,?> e = (Map.Entry<?,?>)o;
            if (Objects.equals(key, e.getKey()) && Objects.equals(value, e.getValue()))
                return true;
        }
        return false;
    }
}
```
 
--------
 
##### （1）HashMap核心方法一：putVal方法

 putVal方法是HashMap内部的方法，很多Map接口实现方法都调用了这个方法来更改或添加键值对。   
 下面是putVal方法的源代码，为了方便阅读，我对源码进行了一些调整（因为源代码实在是有些乱）。

 参数解析：   
 int hash: key的hash值（调用HashMap内部方法hash计算出来的hash值，不是key.hashCode()）   
 K key: 键的引用   
 V value：值的引用   
 boolean onlyIfAbsent：当这个值为true时（例如putIfAbsent方法），如果Map中存在这个键，那么就不会更改对应的值   
 boolean evict：这个值为false时，表示这个HashMap处于一个构造阶段

 
```
final V putVal(int hash, K key, V value, boolean onlyIfAbsent, boolean evict) {
    //首先将HashMap的桶数组引用赋给tab
    Node<K,V>[] tab = table;
    int n;

    //如果table为null或者table的长度为0，那么进行resize操作，并将新table的长度赋给n
    //在调用无参构造器构造HashMap时table会为null
    if (tab == null || (n = tab.length) == 0) {
        tab = resize();
        n = tab.length;
    }
    //定位桶的位置
    int i = (n - 1) & hash;
    Node<K,V> p = tab[i];

    //如果这个桶是空的，那么直接添加这个键值对
    if (p == null) {
        tab[i] = newNode(hash, key, value, null);
    //如果这个桶不是空的，说明遇到了哈希冲突
    } else {
        Node<K,V> e; 
        K k = p.key;
        //如果参数key满足这个桶中的第一个键值对
        if (p.hash == hash && (k == key || (key != null && key.equals(k))))
            e = p;
        //如果不满足，则继续在这个桶中查找
        //首先判断是不是红黑树
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        //不是红黑树则是链表
        else {
            //利用for循环遍历链表直到找到了相等的key或者遍历到了链表的最后
            for (int binCount = 0; ;binCount++) {
                e = p.next;
                //e为null时遍历到了链表最后
                if (e == null) {
                    //构建一个新的键值对
                    p.next = newNode(hash, key, value, null);
                    //如果现在这个链表长度大于等于8并且桶的长度大于64，那么转换为红黑树
                    if (binCount >= TREEIFY_THRESHOLD - 1)
                        treeifyBin(tab, hash);
                    break;
                }
                k = e.key;
                //如果找到了相等的key
                if (e.hash == hash && (k == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        //e不为null时说明Map中存在相等的key
        if (e != null) { 
            V oldValue = e.value;
            //如果参数onlyIfAbsent为false，那么替换这个值
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e); //这个方法在HashMap中实现为空，是用在HashMap子类：LinkedHashMap上的
            //返回旧值
            return oldValue;
        }
    }
    modCount++; //修改次数加1
    size++;
    //如果size大于临界值，那么进行resize操作
    if (size > threshold)
        resize();
    afterNodeInsertion(evict); //同样实现为空，用于LinkedHashMap重写
    return null;
}
```
 归纳起来：putVal方法执行步骤为：   
 1、首先判断table是不是为null或者table的长度为0，否则进行resize操作   
 2、计算参数key的hash值定位桶的位置   
 3、如果桶为空则直接插入元素，如果不为空，则判断这个桶的第一个键值对的键是否和参数key相同，若相同则根据参数onlyIfAbsent决定是否替换这个键值对的值，若不相同，则判断这个Node是否属于TreeNode，属于TreeNode则调用HashMap的treeifyBin插入元素，否则遍历Node保存的链表，并依次比对，如果没有找到相等的键，则在这个链表尾插入键值对。   
 4、最后，如果添加了键值对而没有替换，则将modCount计数器和size加1，并检查是否需要resize。

 在调用putVal方法时，首先需要给出一个键的hash值。HashMap一般通过内部的静态方法hash来计算键的hashCode：

 
```
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```
 这个方法计算hash值的策略是：   
 1、如果key为null，则直接返回哈希值null。   
 2、如果不为null，则将key的hashCode返回值h和h无符号右移位16进行异或运算（相当于让h的低16位和高16位进行xor运算）然后返回。

 在putVal方法定位桶的位置的时候，运用到了一个数学规律：X % length = X & (length - 1)，当length为2的幂次时成立。   
 这就是为什么HashMap的桶长度永远是2的幂次，因为在定位桶位置的时候，位运算效率要比模运算要高。

 我们以一个key为”hello”的字符串来分析这个过程（以桶的长度为16为例）：   
 ![这里写图片描述](https://img-blog.csdn.net/20180831105227202?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FiYzEyM2x6Zg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)   
 由于table长度都是2^n，因此桶的下标仅和hash的低n位相关，其高位在X&(n-1)都被与操作置为0了，所以就相当于高位没有参与运算。JDK的设计者们应该是为了减少哈希冲突的概率（虽然现在很多常用类的hashCode方法设计的很好了），决定将一个key.hashCode方法返回值的高16位和低16为进行一个异或运算来减少哈希冲突概率，反正这种位运算并不会造成多大的时间开销。

 
##### （2）HashMap核心方法二：resize

 resize方法同样是HashMap内部的方法，这个方法会扩大HashMap的桶长度以减少哈希碰撞。当HashMap检测到size大于桶长度乘以负载因子时会被调用。

 
```
final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;
    //oldCap为table长度
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    //旧的临界值
    int oldThr = threshold;
    //分别代表新的table长度，新的扩容最低size大小
    int newCap, newThr = 0;

    //如果table长度大于0
    if (oldCap > 0) {
        //如果table长度大于最大桶大小(1 << 30)，则不进行扩容，直接返回table
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        //将table长度的2倍赋给newCap，作为本次扩容的大小
        } else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY 
                && oldCap >= DEFAULT_INITIAL_CAPACITY) {
            //当newCap小于1<<30并且oldCap>=16时将临界值扩大到原来的2倍
            newThr = oldThr << 1; 
        }
    //如果table长度等于0且旧临界值 > 0,新的扩容大小不变
    } else if (oldThr > 0) {
        newCap = oldThr;
    } else {               
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }

    //如果没有设置新的临界值
    if (newThr == 0) {
        //如果新的table长度小于1 << 30，那么设置新的临界值为ft，否则为Integer.MAX_VALUE
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ? (int)ft : Integer.MAX_VALUE);
    }

    //将新的扩容最小size大小赋值到成员变量
    threshold = newThr;
    //以newCap为长度构建Node数组并赋值到table成员变量
    @SuppressWarnings({"rawtypes","unchecked"})
    Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;

    //如果原table不为null
    if (oldTab != null) {
        //遍历原table的桶
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            //如果这个桶存在键值对
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null;
                //如果这个桶只有一个键值对，那么将这个键值对的引用赋值到新table
                if (e.next == null)
                    newTab[e.hash & (newCap - 1)] = e;
                //判定这个桶是不是红黑树实现的
                else if (e instanceof TreeNode)
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                //如果不是红黑树则是链表
                else {
                    //这里可能会构建两个链表
                    //loHead和loTail表示一个键的hash值和oldCap进行与运算等于0时的链表
                    Node<K,V> loHead = null, loTail = null;
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    //这个循环负责在新table上构建链表
                    do {
                        next = e.next;
                        //e.hash&oldCap当等于0时表示无须改变table的下标
                        //e.hash&oldCap只会存在两个结果：0和oldCap
                        if ((e.hash & oldCap) == 0) {
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }
                        //如果不为0则需要改变下标：旧table数组长度加上原来的下标
                        else {
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    //如果loTail(包含所有e.hash&oldCap等于0的键值对)
                    //对应的链表不为null，则将这个链表添加到newTab[j]位置
                    if (loTail != null) {
                        loTail.next = null;
                        newTab[j] = loHead;
                    }
                    //如果hiTail(包含所有e.hash&oldCap等于oldCap的键值对)
                    //对应的链表不为null，则将这个链表添加到newTab[j+oldCap]位置
                    if (hiTail != null) {
                        hiTail.next = null;
                        //旧table数组长度加上原来的下标
                        newTab[j + oldCap] = hiHead;
                    }
                }
            }
        }
    }
    return newTab;
}
```
 总结一下resize操作过程：   
 1、首先根据HashMap的实际情况设置新的扩容大小，分一下几种情况：   
 （1） 如果table为null，那么会将扩容大小设置为16，并根据负载因子设置临界值   
 （2）如果table不为null，那么会将扩容大小设置为原来的2倍，除非大小已经超出1<<30（如果超出该方法直接返回旧的table）   
 2、根据扩容大小构建一个新的Node数组并直接赋给成员变量table，接着遍历旧的table数组，将里面所有的键值对逐一添加到新的table中并返回该新table的引用。

 很多人可能会问：为什么e.hash & oldCap等于0时不需要改变下标，而大于0时就要改变下标为原数组长度加上旧的下标。

 有这么一个规律：   
 如果hash & oldCap == 0   
 hash & (oldCap - 1) == hash & (newCap - 1);   
 如果hash & oldCap == oldCap   
 hash & (oldCap - 1) + oldCap == hash & (newCap - 1);   
 用代码表示：

 
```
static boolean index(int hash, int oldCap) {
    int newCap = oldCap << 1;
    if((hash & oldCap) == 0)
        return (hash & (oldCap - 1)) == (hash & (newCap - 1));
    return (hash & oldCap - 1) + oldCap == (hash & (newCap - 1));
}
```
 当oldCap为2的幂次时，这个方法永远返回true。这里就不证明了（其实证明也很简单）。

 这么做有一大好处：   
 因为HashMap的扩容策略为2的幂次，即为1 << n。   
 也就是说，e.hash & oldCap等于0还是等于oldCap只和key的第n位相关，这样就有很大概率将这个链表拆分为两半。   
 例如下面这个例子（以hash值为10和116，oldCap为16作为示范）：   
 ![这里写图片描述](https://img-blog.csdn.net/20180831173433539?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FiYzEyM2x6Zg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)   
 注意颜色为红色的位，只有当第n位的两个位都为1的时候，e.hash & oldCap才会等于16，否则等于0。   
 这样做有很大概率可以将原来的一个链表拆分成两个长度相差不大的链表，以保证HashMap查找效率。

 
##### （3）HashMap核心方法三：getNode

 getNode方法是HashMap内部方法，其功能是根据键来查找键值对Node。   
 **为了方便阅读，我对源代码进行了修改。**

 
```
final Node<K,V> getNode(int hash, Object key) {
    Node<K,V>[] tab = table; 
    int n = (table == null) ? 0 : table.length;
    //如果table为空则直接返回null
    if(table == null || n == 0)
        return null;

    //定位桶的位置
    Node<K,V> first = table[(n - 1) & hash]; 
    if(first == null)
        return null;
    //获取这个桶的第一个键值对的key
    K k = first.key;
    //如果参数key和这个键值对的key相等，则直接返回这个Node
    if (first.hash == hash && (k == key || (key != null && key.equals(k))))
        return first;
    //获取这个桶第二个Node
    Node<K,V> e = first.next; 
    if (e != null) {
        //如果是红黑树则调用getTreeNode方法
        if (first instanceof TreeNode)
            return ((TreeNode<K,V>)first).getTreeNode(hash, key);
        //遍历链表直到找到符合要求的键值对
        do {
            if (e.hash == hash &&((k = e.key) == key || (key != null && key.equals(k))))
                return e;
        } while ((e = e.next) != null);
    }
    return null;
}
```
 removeNode方法就不介绍了，大致流程都差不多。

 
##### （4）HashMap的构造方法

 HashMap提供了4种public构造方法   
 **public HashMap()**

 
```
public HashMap() {
    this.loadFactor = DEFAULT_LOAD_FACTOR;
}
```
 HashMap提供了无参构造方法，该方法默认指定负载因子为0.75，并将其它所有int型变量设置为0，table默认为null

 **public HashMap(int, float)**

 
```
public HashMap(int initialCapacity, float loadFactor) {
    if (initialCapacity < 0)
        throw new IllegalArgumentException("Illegal initial capacity: " + initialCapacity);
    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;
    if (loadFactor <= 0 || Float.isNaN(loadFactor))
        throw new IllegalArgumentException("Illegal load factor: " + loadFactor);
    this.loadFactor = loadFactor;
    //该方法返回大于等于initialCapacity的最小2次幂数
    this.threshold = tableSizeFor(initialCapacity);
}

static final int tableSizeFor(int cap) {
    int n = cap - 1;
    n |= n >>> 1;
    n |= n >>> 2;
    n |= n >>> 4;
    n |= n >>> 8;
    n |= n >>> 16;
    return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}
```
 这个构造方法可以预先指定桶的大小和自定义负载因子。如果传入非法参数则会抛出IllegalArgumentException异常。   
 如果指定的桶的大小不是2次方数，则会构造方法会选择一个大于等于initialCapacity的最小二次幂数

 **public HashMap(int)**

 
```
public HashMap(int initialCapacity) {
    this(initialCapacity, DEFAULT_LOAD_FACTOR);
}
```
 这个方法同样可以指定桶的大小，负载因子为默认的0.75   
 在使用HashMap的时候，应尽量调用此构造方法构造HashMap，因为HashMap的resize操作是一个比较耗时的操作。

 **public HashMap(Map<? extends K, ? extends V> m)**

 
```
public HashMap(Map<? extends K, ? extends V> m) {
    this.loadFactor = DEFAULT_LOAD_FACTOR;
    putMapEntries(m, false);
}
//putAll、clone方法也会调用这个方法
//evict为false时代表为构造阶段
final void putMapEntries(Map<? extends K, ? extends V> m, boolean evict) {
    int s = m.size();
    if (s > 0) {
        if (table == null) {
            float ft = ((float)s / loadFactor) + 1.0F;
            int t = ((ft < (float)MAXIMUM_CAPACITY) ? (int)ft : MAXIMUM_CAPACITY);
            if (t > threshold)
                threshold = tableSizeFor(t);
        } else if (s > threshold)
            resize();
        //通过一个for循环将所有的键值对添加到table中
        for (Map.Entry<? extends K, ? extends V> e : m.entrySet()) {
            K key = e.getKey();
            V value = e.getValue();
            putVal(hash(key), key, value, false, evict);
        }
    }
}
```
 这个构造方法可以构造一个包含这个Map里面所有键值对的HashMap。

 
##### （5）entrySet方法

 entrySet方法是Map接口中定义的方法，这个方法返回一个Set<Map.Entry<K,V>>，里面包含这个Map所有的键值对。

 
```
public Set<Map.Entry<K,V>> entrySet() {
    Set<Map.Entry<K,V>> es;
    return (es = entrySet) == null ? (entrySet = new EntrySet()) : es;
}
```
 HashMap首先会检查它的成员变量entrySet（这个成员变量用于缓存一个EntrySet对象，避免重复创建该对象）是否为null，如果为null则构建一个EntrySet对象，否则直接返回成员变量entrySet。   
 EntrySet是HashMap的一个非静态内部类。它提供了Set所有方法的实现。

 
```
final class EntrySet extends AbstractSet<Map.Entry<K,V>> {
    public final int size()                 { return size; }
    public final void clear()               { HashMap.this.clear(); }
    public final Iterator<Map.Entry<K,V>> iterator() {
        return new EntryIterator();
    }
    public final boolean contains(Object o) {
        if (!(o instanceof Map.Entry))
            return false;
        Map.Entry<?,?> e = (Map.Entry<?,?>) o;
        Object key = e.getKey();
        Node<K,V> candidate = getNode(hash(key), key);
        return candidate != null && candidate.equals(e);
    }
    public final boolean remove(Object o) {
        if (o instanceof Map.Entry) {
            Map.Entry<?,?> e = (Map.Entry<?,?>) o;
            Object key = e.getKey();
            Object value = e.getValue();
            return removeNode(hash(key), key, value, true, true) != null;
        }
        return false;
    }
    //省略forEach、spliterator方法
}
```
 EntrySet实际上就是调用外部类HashMap中的方法来实现Set接口中的功能。   
 对于EntrySet的iterator方法，同样是构建了一个非静态内部类EntryIterator。   
 和其它非线程安全的集合类一样，如果在迭代元素的时候有其它线程修改了Map，则会抛出ConcurrentModificationException异常。   
 遍历Map相当于遍历table数组，在HashMap较大的情况下，如非必要应当尽量少遍历整个HashMap。

 
### 三、序列化和克隆

 
##### （1）序列化

 和很多Java集合类一样，HashMap对存储键值对的结构添加了transient关键字并通过自定义writeObject和readObject来实现序列化操作   
 HashMap有以下成员变量添加了transient关键字：   
 table数组、EntrySet缓存、size、modCount。

 
```
private void writeObject(ObjectOutputStream s) throws IOException {
    int buckets = capacity();
    s.defaultWriteObject();
    //写入table数组的长度
    s.writeInt(buckets);
    //写入键值对数量size
    s.writeInt(size);
    //调用这个方法将键值对数据写入
    internalWriteEntries(s);
}

void internalWriteEntries(java.io.ObjectOutputStream s) throws IOException {
    Node<K,V>[] tab;
    //遍历table数组
    if (size > 0 && (tab = table) != null) {
        for (int i = 0; i < tab.length; ++i) {
            //写入键和值
            for (Node<K,V> e = tab[i]; e != null; e = e.next) {
                s.writeObject(e.key);
                s.writeObject(e.value);
            }
        }
    }
}
```
 HashMap序列化后的数据包括：resize临界值threshold、负载因子loadFactor、table数组的长度、键值对数量size、键值对数据。

 
```
private void readObject(io.ObjectInputStream s) throws IOException, ClassNotFoundException {
    //这个方法会读入loadFactor、 threshold
    s.defaultReadObject();
    //将HashMap中成员变量设置为null或0(除了loadFactor)
    reinitialize();

    if (loadFactor <= 0 || Float.isNaN(loadFactor))
        throw new InvalidObjectException("Illegal load factor: " + loadFactor);
    //读入table数组长度
    s.readInt();
    //读入键值对数量size
    int mappings = s.readInt();

    if (mappings < 0)
        throw new InvalidObjectException("Illegal mappings count: " + mappings); 
    else if (mappings > 0) {
        //将loadFactor作为负载因子，但loadFactor不会小于0.25，不会大于4.0
        float lf = Math.min(Math.max(0.25f, loadFactor), 4.0f);
        float fc = (float)mappings / lf + 1.0f;
        //将大于等于fc的最小2次幂数作为桶的长度，桶的长度最小16，最大1<<30
        int cap = ((fc < DEFAULT_INITIAL_CAPACITY) ?
                       DEFAULT_INITIAL_CAPACITY :
                       (fc >= MAXIMUM_CAPACITY) ?
                       MAXIMUM_CAPACITY :
                       tableSizeFor((int)fc));
        float ft = (float)cap * lf;
        threshold = ((cap < MAXIMUM_CAPACITY && ft < MAXIMUM_CAPACITY) ?
                         (int)ft : Integer.MAX_VALUE);
        //构建长度为cap的桶
        @SuppressWarnings({"rawtypes","unchecked"})
        Node<K,V>[] tab = (Node<K,V>[])new Node[cap];
        table = tab;
        //将键值对读入并调用putVal方法添加
        for (int i = 0; i < mappings; i++) {
            @SuppressWarnings("unchecked")
            K key = (K) s.readObject();
            @SuppressWarnings("unchecked")
            V value = (V) s.readObject();
            putVal(hash(key), key, value, false, false);
        }
    }
}

void reinitialize() {
    table = null;
    entrySet = null;
    keySet = null;
    values = null;
    modCount = 0;
    threshold = 0;
    size = 0;
}
```
 在反序列化HashMap的时候，并不会按照之前的桶长度重新构建桶。

 
##### （2）克隆

 
```
@SuppressWarnings("unchecked")
@Override
public Object clone() {
    HashMap<K,V> result;
    try {
        result = (HashMap<K,V>)super.clone();
    } catch (CloneNotSupportedException e) {
        //一般不会发生
        throw new InternalError(e);
    }
    //将除了threshold之外的变量置为null或0
    result.reinitialize();
    //将原HashMap中的元素的键值对引用复制一份到克隆的HashMap
    result.putMapEntries(this, false);
    return result;
}
```
 在克隆的时候应当对里面的元素的复制一份出来。如果不这样做，两个HashMap会共用一个table数组，会造成调用了其中一个方法后另一个HashMap的状态也会随之改变。

   
  