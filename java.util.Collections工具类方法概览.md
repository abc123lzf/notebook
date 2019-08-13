---
title: java.util.Collections工具类方法概览
date: 2018-09-06 20:34:04
tags: CSDN迁移
---
 版权声明：尊重原创，转载请注明出处 [ https://blog.csdn.net/abc123lzf/article/details/82455190]( https://blog.csdn.net/abc123lzf/article/details/82455190)   
  本文以JDK1.8的Collections源码进行解析

 
### 一、引言

 java.util.Collections工具类提供了很多操作集合的方法，如果你还不了解这些方法的具体作用，那么了解其使用方法还是很有必要，可以有效的简化代码和增加其可读性   
 Collections类中的方法可分为一下几种类型：   
 **1、搜索集合：**   
 二分搜索元素：

 
```
//将List集合对key进行二分搜索，返回key的下标
public static <T> int binarySearch(List<? extends Comparable<? super T>> list, T key);
//将List集合根据外比较器c的规则对key进行二分搜索，返回key的下标
public static <T> int binarySearch(List<? extends T> list, T key, Comparator<? super T> c);
```
 查找最小、最大、相等元素：

 
```
//返回集合coll中最小的元素
public static <T extends Object & Comparable<? super T>> T min(Collection<? extends T> coll);
//根据外比较器comp选出coll中最小的元素
public static <T> T min(Collection<? extends T> coll, Comparator<? super T> comp);
//选出集合coll中的最大元素
public static <T extends Object & Comparable<? super T>> T max(Collection<? extends T> coll);
//根据外比较器comp选出集合coll中的最大元素
public static <T> T max(Collection<? extends T> coll, Comparator<? super T> comp);
//返回一个集合c中所有等于o的元素数量
public static int frequency(Collection<?> c, Object o);

```
 查找子集、交集：

 
```
//获取target在source中的第一个子集位置，不存在返回-1
public static int indexOfSubList(List<?> source, List<?> target);
//获取target在source中的最后一个子集位置，不存在返回-1
public static int lastIndexOfSubList(List<?> source, List<?> target);
//返回c1和c2集合是否有交集
public static boolean disjoint(Collection<?> c1, Collection<?> c2);
```
 **2、修改集合元素、排列方式**   
 排序集合：

 
```
//对List集合排序，并且List集合中的元素必须要实现Comparable接口
public static <T extends Comparable<? super T>> void sort(List<T> list);
//对List集合中的元素和c对比进行排序
public static <T> void sort(List<T> list, Comparator<? super T> c);
```
 替换集合中的元素：

 
```
//将List集合中的第i个元素和第j个元素交换
public static void swap(List<?> list, int i, int j);
//将List集合中所有的元素替换成obj
public static <T> void fill(List<? super T> list, T obj);
//将list集合中所有等于oldVal的元素替换成newVal
public static <T> boolean replaceAll(List<T> list, T oldVal, T newVal);
```
 随机打乱集合：

 
```
//将List集合中的元素随机打乱
public static void shuffle(List<?> list);
//根据Random将List集合中元素随机打乱
public static void shuffle(List<?> list, Random rnd);
```
 倒转、旋转集合：

 
```
//倒转List集合中的元素
public static void reverse(List<?> list);
//将list集合中的元素旋转distance个距离
public static void rotate(List<?> list, int distance);
```
 添加元素：

 
```
//向集合c中添加多个elements元素，返回是否全部添加成功
public static <T> boolean addAll(Collection<? super T> c, T... elements);
```
 **3、将集合包装为线程安全集合**

 
```
//将集合c封装为一个线程安全的集合并返回
public static <T> Collection<T> synchronizedCollection(Collection<T> c);
//将Set集合c封装为一个线程安全的集合并返回
public static <T> Set<T> synchronizedSet(Set<T> s);
//将SortedSet集合c封装为一个线程安全的集合并返回
public static <T> SortedSet<T> synchronizedSortedSet(SortedSet<T> s);
//将NavigableSet集合s封装为一个线程安全的集合并返回
public static <T> NavigableSet<T> synchronizedNavigableSet(NavigableSet<T> s);
//将NavigableSet集合s封装为一个线程安全的集合并返回
public static <T> List<T> synchronizedList(List<T> list);
//将Map封装为一个线程安全的Map并返回
public static <K,V> Map<K,V> synchronizedMap(Map<K,V> m);
//将SortedMap封装为一个线程安全的SortedMap并返回
public static <K,V> SortedMap<K,V> synchronizedSortedMap(SortedMap<K,V> m);
////将NavigableMap封装为一个线程安全的NavigableMap并返回
public static <K,V> NavigableMap<K,V> synchronizedNavigableMap(NavigableMap<K,V> m);
```
 **4、将集合包装为只读集合**

 
```
//将集合c封装为一个只读的集合并返回
public static <T> Collection<T> unmodifiableCollection(Collection<? extends T> c);
//将Set集合s封装为一个只读的Set并返回
public static <T> Set<T> unmodifiableSet(Set<? extends T> s);
//将SortedSet集合s封装为一个只读的SortedSet并返回
public static <T> SortedSet<T> unmodifiableSortedSet(SortedSet<T> s);
//将NavigableSet集合s封装为一个只读的NavigableSet并返回
public static <T> NavigableSet<T> unmodifiableNavigableSet(NavigableSet<T> s);
//将List集合s封装为一个只读的List并返回
public static <T> List<T> unmodifiableList(List<? extends T> list);
//将Map m封装为一个只读的Map并返回
public static <K,V> Map<K,V> unmodifiableMap(Map<? extends K, ? extends V> m);
//将SortedMap封装为一个只读的SortedMap并返回
public static <K,V> SortedMap<K,V> unmodifiableSortedMap(SortedMap<K, ? extends V> m);
//将NavigableMap封装为一个只读的NavigableMap并返回
public static <K,V> NavigableMap<K,V> unmodifiableNavigableMap(NavigableMap<K, ? extends V> m);
```
 **5、将集合包装为类型安全的集合**

 
```
//将集合c封装成一个类型安全的Collection集合，其类型只可为type或type子类
public static <E> Collection<E> checkedCollection(Collection<E> c, Class<E> type);
//将队列queue封装为一个类型安全的对列，其类型只可为type或type子类
public static <E> Queue<E> checkedQueue(Queue<E> queue, Class<E> type);
//将Set集合封装为一个类型安全的Set集合，其类型只可为type或type子类
public static <E> Set<E> checkedSet(Set<E> s, Class<E> type);
//将SortedSet集合封装为一个类型安全的SortedSet集合，其类型只可为type或type子类
public static <E> SortedSet<E> checkedSortedSet(SortedSet<E> s, Class<E> type);
//将NavigableSet集合封装为一个类型安全的NavigableSet集合，其类型可为type或type子类
public static <E> NavigableSet<E> checkedNavigableSet(NavigableSet<E> s, Class<E> type);
//将Map封装为一个类型安全的Map，其键类型只可为keyType及其子类，其值类型只可为valueType及其子类
public static <K, V> Map<K, V> checkedMap(Map<K, V> m, Class<K> keyType, Class<V> valueType);
//将SortedMap封装为一个类型安全的SortedMap，其键类型只可为keyType及其子类，其值类型只可为valueType及其子类
public static <K,V> SortedMap<K,V> checkedSortedMap(SortedMap<K, V> m, 
        Class<K> keyType, Class<V> valueType);
//将NavigableMap封装为一个类型安全的NavigableMap，其键类型只可为keyType及其子类，其值类型只可为valueType及其子类
public static <K,V> NavigableMap<K,V> checkedNavigableMap(NavigableMap<K, V> m,
        Class<K> keyType, Class<V> valueType);
```
 **6、返回一个简单的集合**   
 返回空的集合：

 
```
//返回一个空的迭代器
public static <T> Iterator<T> emptyIterator();
//返回一个空的List迭代器
public static <T> ListIterator<T> emptyListIterator();
//返回一个Enumeration迭代器
public static <T> Enumeration<T> emptyEnumeration();
//返回一个空的Set集合
public static final <T> Set<T> emptySet();
//返回一个空的SortedSet集合
public static <E> SortedSet<E> emptySortedSet();
//返回一个空的NavigableSet集合
public static <E> NavigableSet<E> emptyNavigableSet();
//返回一个空的List集合
public static final <T> List<T> emptyList();
//返回一个空的Map
public static final <K,V> Map<K,V> emptyMap();
//返回一个空的SortedMap
public static final <K,V> SortedMap<K,V> emptySortedMap();
//返回一个空的NavigableMap集合
public static final <K,V> NavigableMap<K,V> emptyNavigableMap();
```
 返回只包含一个元素的集合：

 
```
//返回一个只有一个元素o的Set集合
public static <T> Set<T> singleton(T o);
//返回一个只有一个元素o的Set集合
public static <T> List<T> singletonList(T o);
//返回一个只有一个键值对key-value的Map
public static <K,V> Map<K,V> singletonMap(K key, V value);
```
 返回一个长度为n但是元素全是o的集合：

 
```
//返回一个长度为n但所有元素都是o的List集合
public static <T> List<T> nCopies(int n, T o);
```
 **7、其它**   
 返回Comparator比较器：

 
```
//返回一个反向的外比较器
public static <T> Comparator<T> reverseOrder();
//根据cmp比较方式返回一个和cmp相反的外比较器
public static <T> Comparator<T> reverseOrder(Comparator<T> cmp);
```
 复制集合元素：

 
```
//将迭代器中迭代出来的所有元素添加到一个ArrayList并返回
public static <T> ArrayList<T> list(Enumeration<T> e);
//将src中的元素按顺序复制到dest中
public static <T> void copy(List<? super T> dest, List<? extends T> src);
```
 返回Enumeration迭代器：

 
```
//根据集合c的元素返回一个Enumeration迭代器
public static <T> Enumeration<T> enumeration(final Collection<T> c);
```
 双向队列转换为单向队列：

 
```
//将Deque双向队列转换成Queue
public static <T> Queue<T> asLifoQueue(Deque<T> deque);
```
 提取Map的键集合：

 
```
//将Map<E,Boolean>类型的map中的所有键提取到Set并返回
public static <E> Set<E> newSetFromMap(Map<E, Boolean> map);
```
 
--------
 
### 二、源码解析

 关于Collections类的源码我们只挑几个比较有代表性的方法解析：

 
##### （1）二分搜索：binarySearch

 二分搜索实现了以O(log N)的时间复杂度来查找有序集合中的元素。   
 Collections提供了两个重载的binarySearch方法，这里就只讲解<T> int binarySearch(List<? extends Comparable<? super T>>, T key)，元素key

 
```
public static <T> int binarySearch(List<? extends Comparable<? super T>> list, T key) {
    if (list instanceof RandomAccess || list.size()<BINARYSEARCH_THRESHOLD)
        return Collections.indexedBinarySearch(list, key);
    else
        return Collections.iteratorBinarySearch(list, key);
}
```
 首先如果集合中的元素数量小于BINARYSEARCH_THRESHOLD（大小为5000）的话，那么统一采用indexedBinarySearch法进行二分搜索，即通过get(i)来获取集合的元素。   
 其次如果List实现了RandomAccess接口（这个接口是一个标记接口，表示该集合支持快速随机访问，ArrayList实现了该接口，LinkedList没有实现），那么即使元素大于5000，也采用indexedBinarySearch搜索，否则采用迭代器来访问元素进行搜索。

 
```
private static <T> int indexedBinarySearch(List<? extends Comparable<? super T>> list, T key) {
    int low = 0;
    int high = list.size()-1;

    while (low <= high) {
        //mid为(低位+高位)2
        int mid = (low + high) >>> 1;
        Comparable<? super T> midVal = list.get(mid);
        int cmp = midVal.compareTo(key);
        //如果midVal < key
        if (cmp < 0)
            low = mid + 1;
        //如果midVal > key
        else if (cmp > 0)
            high = mid - 1;
        //等于直接返回位置
        else
            return mid;
    }
    //没有找到key
    return -(low + 1);
}
```
 Collections的二分搜索采用了循环而非递归的实现。   
 我们以下面一个ArrayList中的数组为例，查找的key为11，其过程为：   
 ![这里写图片描述](https://img-blog.csdn.net/20180906173942367?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FiYzEyM2x6Zg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

 
```
private static <T> int iteratorBinarySearch(List<? extends Comparable<? super T>> list, T key) {
    int low = 0;
    int high = list.size()-1;
    ListIterator<? extends Comparable<? super T>> i = list.listIterator();

    while (low <= high) {
        int mid = (low + high) >>> 1;
        Comparable<? super T> midVal = get(i, mid);
        int cmp = midVal.compareTo(key);

        if (cmp < 0)
            low = mid + 1;
        else if (cmp > 0)
            high = mid - 1;
        else
            return mid;
    }
    return -(low + 1);
}

private static <T> T get(ListIterator<? extends T> i, int index) {
    T obj = null;
    int pos = i.nextIndex();
    if (pos <= index) {
        do {
            obj = i.next();
        } while (pos++ < index);
    } else {
        do {
            obj = i.previous();
        } while (--pos > index);
    }
    return obj;
}
```
 iteratorBinarySearch和indexedBinarySearch的方法没有太大差别，仅仅只是获取元素的方式不同，前者是通过调用List的get方法，后者则是通过List的listIterator获取元素。

 
##### （2）倒转集合：reverse

 
```
@SuppressWarnings({"rawtypes", "unchecked"})
public static void reverse(List<?> list) {
    int size = list.size();
    if (size < REVERSE_THRESHOLD || list instanceof RandomAccess) {
        //i的范围为0~size/2，j为size/2+1 ~ size
        for (int i=0, mid=size>>1, j=size-1; i<mid; i++, j--)
            swap(list, i, j);
    } else {
        //获取两个迭代器，fwd初始指针为0，rev初始指针为size-1
        ListIterator fwd = list.listIterator();
        ListIterator rev = list.listIterator(size);
        //采用同样的策略交换元素
        for (int i=0, mid=list.size()>>1; i<mid; i++) {
            Object tmp = fwd.next();
            fwd.set(rev.previous());
            rev.set(tmp);
        }
    }
}

@SuppressWarnings({"rawtypes", "unchecked"})
public static void swap(List<?> list, int i, int j) {
    final List l = list;
    l.set(i, l.set(j, l.get(i)));
}
```
 reverse方法和binarySearch在选择获取元素的方式上一样，在交换元素时，时间复杂度为O(N/2)：   
 ![这里写图片描述](https://img-blog.csdn.net/20180906175933903?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FiYzEyM2x6Zg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

 
##### （3）将集合包装为只读集合：unmodifiableCollection

 
```
public static <T> Collection<T> unmodifiableCollection(Collection<? extends T> c) {
    return new UnmodifiableCollection<>(c);
}
```
 这个方法返回的实际上是Collections的一个内部类：

 
```
static class UnmodifiableCollection<E> implements Collection<E>, Serializable {
    final Collection<? extends E> c;

    UnmodifiableCollection(Collection<? extends E> c) {
        if (c==null)
            throw new NullPointerException();
        this.c = c;
    }
    public int size()                   {return c.size();}
    public boolean isEmpty()            {return c.isEmpty();}
    public boolean contains(Object o)   {return c.contains(o);}
    //....
    public boolean add(E e) {
        throw new UnsupportedOperationException();
    }
    //...
}
```
 可以看到，UnmodifiableCollection采用了典型的装饰者设计模式，对于正常的读操作，都是调用的原Collection对象的方法，对于修改操作，则抛出UnsupportedOperationException异常。   
 其实像其它类似的方法，包括将集合包装为一个线程安全的集合，返回一个只有1个元素的集合，都是返回的Collections的内部类。   
 所以在使用这些方法的时候不要进行强制转换：

 
```
//不要这样，会抛出ClassCastException
ArrayList<Object> arr = (ArrayList<Object>)Collections.unmodifiableList(new ArrayList<Object>());
```
   
  