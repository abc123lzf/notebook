---
title: java.util.ArrayList源码解析
date: 2018-08-29 19:39:11
tags: CSDN迁移
---
 版权声明：尊重原创，转载请注明出处 [ https://blog.csdn.net/abc123lzf/article/details/82154383]( https://blog.csdn.net/abc123lzf/article/details/82154383)   
  本文将根据JDK 1.8版本的ArrayList解析

 
### []()一、概述

 `ArrayList` 是Java一个很常用的集合类，它相当于一个动态数组，内部的数组大小可以根据元素实际情况自动分配，也可以自己分配大小。  
 在使用 `ArrayList` 的时候，应注意 `ArrayList` 并不是线程安全的，如果需要多线程并发操作应当使用 `CopyOnWriteArrayList` (读远大于写的情况)，或者使用 `Collections` 工具类的 `synchronizedList` 方法将其包装。

 下面是 `ArrayList`  UML类图  
 ![这里写图片描述](https://img-blog.csdn.net/20180828204450684?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FiYzEyM2x6Zg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)  
  `ArrayList` 继承了 `AbstractList` 抽象类，实现了 `RandomAccess` 、 `Serializable` 、 `Cloneable` 接口，说明 `ArrayList` 支持快速随机访问、支持克隆和序列化操作。

 
### []()二、源码解析

 
#### []()1、实例变量

 `ArrayList` 只有两个实例变量：

 
```
transient Object[] elementData;
private int size;

```
 `elementData` 负责保存该集合持有的元素，size保存该集合的持有的元素个数（不一定等于 `elementData.length` ）  
 除此之外，AbstractList还持有一个实例变量：

 
```
protected transient int modCount = 0;

```
 `modCount` 用来记录这个List的修改次数

 
#### []()2、构造方法

 ArrayList提供了三个public构造方法：

 
```
private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};
public ArrayList() {
	this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
}

```
 这个构造方法默认将 `elementData` 初始化为一个空数组，当调用了 `add` 方法时，会默认分配一个长度为10的数组给 `elementData` 。

 
```
public ArrayList(int initialCapacity) {
	if (initialCapacity > 0) {
		this.elementData = new Object[initialCapacity];
    } else if (initialCapacity == 0) {
        this.elementData = EMPTY_ELEMENTDATA;
    } else {
            throw new IllegalArgumentException("Illegal Capacity: "+
					initialCapacity);
	}
}

```
 该构造方法可以传入一个 `int` 参数，代表初始分配的数组大小，如果传入的参数小于0会抛出 `IllegalArgumentException` 异常。

 
```
public ArrayList(Collection<? extends E> c) {
    elementData = c.toArray();
    if ((size = elementData.length) != 0) {
		if (elementData.getClass() != Object[].class)
	        elementData = Arrays.copyOf(elementData, size, Object[].class);
    } else {
		this.elementData = EMPTY_ELEMENTDATA;
	}
}

```
 可以传入一个 `Collection` 集合，该构造方法会将这个集合里的所有元素作为 `ArrayList` 的初始元素。  
 首先该方法会调用 `toArray` 方法（ `<T> T[] toArray()` ）获得该集合所有元素的引用副本，如果该集合不为空且数组类型不为 `Object[]` ，则将这些元素的引用复制到 `elementData` 数组。

 
#### []()3、方法分析

 
##### []()（1）size、isEmpty方法

 该方法直接返回成员变量 `size` 

 
```
public int size() {
	return size;
}

```
 其实现和父类 `AbstractList` 相同  
 同样 `isEmpty` 方法也是，直接判断 `size` 是不是为0

 
##### []()（2）add方法

 `add` 方法的作用是向集合添加元素， `ArrayList` 中有两个重载的方法：  
  `public boolean add(E e);` 和 `public void add(int index, E element);`   
 **add(E e)方法详解**

 
```
public boolean add(E e) {
    ensureCapacityInternal(size + 1);
    elementData[size++] = e;
	return true;
}

private void ensureCapacityInternal(int minCapacity) {
    if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
		minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
    }
	ensureExplicitCapacity(minCapacity);
}

private void ensureExplicitCapacity(int minCapacity) {
    modCount++;
	if (minCapacity - elementData.length > 0)
        grow(minCapacity);
}

private void grow(int minCapacity) {
	int oldCapacity = elementData.length;
	//相当于oldCapacity的1.5倍
    int newCapacity = oldCapacity + (oldCapacity >> 1);
    if (newCapacity - minCapacity < 0)
        newCapacity = minCapacity;
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);
    elementData = Arrays.copyOf(elementData, newCapacity);
}

private static int hugeCapacity(int minCapacity) {
	//小于0代表minCapacity溢出
	if (minCapacity < 0)
        throw new OutOfMemoryError();
    return (minCapacity > MAX_ARRAY_SIZE) ? Integer.MAX_VALUE : MAX_ARRAY_SIZE;
}

```
 在调用 `add` 方法真正添加元素之前，会首先检测 `elementData` 数组的长度是否足够，具体的执行逻辑是：  
 首先调用 `ensureCapacityInternal` 方法，如果 `elementData` 指向了 `DEFAULTCAPACITY_EMPTY_ELEMENTDATA` ，即空数组（通过无参构造器构造 `ArrayList` ，或指定初始化大小为0构造，或指定的初始化集合没有元素时构造， `elementData` 会指向该元素），那么会默认给 `elementData` 分配一个长度为10的数组。接着调用 `ensureExplicitCapacity` 方法，该方法首先增加计数器 `modCount` ，接着判断数组空间大小是否足够（即添加元素后数组会不会越界），如果不够，则调用 `grow` 方法。 `grow` 方法的作用时给 `elementData` 分配一个新的数组并将旧的数组拷贝到这个新数组中，默认分配大小为原数组的1.5倍。如果分配的数组过大（超过 `Integer.MAX_VALUE - 8` ，一般都不会那么大），则调用 `hugeCapacity` 静态方法 ，如果 `minCapacity` 介于 `Integer.MAX_VALUE - 8` 到 `Integer.MAX_VALUE` ，则直接分配一个 `Integer.MAX_VALUE` 大小的数组，否则抛出 `OutOfMemoryError` 。

 **void add(int index, E element)**  
 index为插入元素的位置

 
```
public void add(int index, E element) {
	rangeCheckForAdd(index);
    ensureCapacityInternal(size + 1);
    System.arraycopy(elementData, index, elementData, index + 1, size - index);
    elementData[index] = element;
    size++;
}
private void rangeCheckForAdd(int index) {
	if (index > size || index < 0)
        throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
}
private String outOfBoundsMsg(int index) {
	return "Index: " + index + ", Size: " + size;
}

```
 该方法首先会检查数组下标，如果 `index` 越界则抛出 `IndexOutOfBoundsException` 异常。接着按 `add(E e)` 方法一样检查数组长度是否足够。然后将数组从 `index` 开始右移一个位置，再将目标元素插入到 `elementData[index]` 中。

 
#### []()（3）remove方法

 `remove` 有两个重载的方法： `E remove(int)` 和 `boolean remove(Object o)`   
 **E remove(int)**  
 该方法需要传入一个 `int` 参数，代表需要移除的数组元素下标，返回的是删除的元素

 
```
public E remove(int index) {
	rangeCheck(index);
    modCount++;
    E oldValue = elementData(index);

    int numMoved = size - index - 1;
    if (numMoved > 0)
		System.arraycopy(elementData, index+1, elementData, index, numMoved);
    elementData[--size] = null;

    return oldValue;
}
@SuppressWarnings("unchecked")
E elementData(int index) {
	return (E) elementData[index];
}

```
 同样，首先检查数组下标是否越界。然后调用 `System.arraycopy` 将需要删除的元素后面所有的数组元素往前移一个位置，最后显式调用 `elementData[--size] = null;` 来通知GC：空间不足时可以将此对象进行回收。

 **boolean remove(Object)**  
 该方法传入一个 `Object` 对象，来删除集合中调用 `equals` 方法返回 `true` 的对象。如果有任意一个元素满足条件被删除则直接返回 `true` 。

 
```
public boolean remove(Object o) {
	if (o == null) {
        for (int index = 0; index < size; index++)
            if (elementData[index] == null) {
                fastRemove(index);
                return true;
            }
        } else {
		for (int index = 0; index < size; index++)
			if (o.equals(elementData[index])) {
				fastRemove(index);
                return true;
		}
    }
	return false;
}

private void fastRemove(int index) {
    modCount++;
    int numMoved = size - index - 1;
    if (numMoved > 0)
	    System.arraycopy(elementData, index + 1, elementData, index, numMoved);
	elementData[--size] = null;
}

```
 该方法首先判断传入的 `Object` 是不是为 `null` ，如果为 `null` ，则从数组下标0开始搜索第一个为 `null` 的元素，找到后调用私有方法 `fastRemove` （和上面 `remove` 方法同样的策略）移动数组并删除，然后返回 `true` 。如果不为 `null` ，则同样遍历数组，删除第一个 `equals` 方法返回 `true` 对象，返回 `true` 。如果没有找到符合条件的对象，返回 `false` 。

 
##### []()（4）get、set方法

 **E get(int)**  
 get方法需要传入一个int参数，代表下标号

 
```
public E get(int index) {
	rangeCheck(index);
    return elementData(index);
}

```
 `get` 方法首先检查参数index是否越界，否则抛出异常。然后调用 `elementData` 方法直接返回元素。

 **E set(int, E)**  
 set方法需要传入两个参数，添加的元素和位置

 
```
public E set(int index, E element) {
    rangeCheck(index);
    E oldValue = elementData(index);
	elementData[index] = element;
	return oldValue;
}

```
 这个方法不像 `add` 方法那样会移出一个位置插入元素， `set` 方法会直接在数组的 `index` 位置放入元素，如果之前这个位置已经存在一个元素则会被替换，最后返回那个被替换掉的元素。

 
##### []()（5）contains、indexOf、lastIndexOf方法

 **int indexOf(Object)**  
 该方法返回第一个和Object相等的元素所在的数组下标，如果不存在返回-1

 
```
public int indexOf(Object o) {
	if (o == null) {
		for (int i = 0; i < size; i++)
            if (elementData[i]==null)
                return i;
    } else {
        for (int i = 0; i < size; i++)
            if (o.equals(elementData[i]))
                return i;
    }
    return -1;
}

```
 **boolean contains(Object o)**  
 该方法判断数组中是否存在相同的元素

 
```
public boolean contains(Object o) {
	return indexOf(o) >= 0;
}

```
 **int lastIndexOf(Object)**  
 该方法返回该数组最后一个和参数相等的对象

 
```
public int lastIndexOf(Object o) {
	if (o == null) {
		for (int i = size-1; i >= 0; i--)
			if (elementData[i]==null)
				return i;
    } else {
        for (int i = size-1; i >= 0; i--)
            if (o.equals(elementData[i]))
                return i;
    }
    return -1;
}

```
 从元素尾部开始遍历数组，遇到第一个满足条件的数组元素直接返回其下标。

 
##### []()（6）toArray方法

 `toArray` 有两个重载方法： `Object[] toArray()` 和 `<T> T toArray(T[])`   
 这两个方法的作用就是将数组内所有元素的引用拷贝到一个新的数组中并返回  
 ** `Object[] toArray()` **

 
```
public Object[] toArray() {
	return Arrays.copyOf(elementData, size);
}

```
 该方法直接调用 `Arrays.copyOf` 方法拷贝，该数组包含这个集合中所有的元素的引用。

 ** `<T> T toArray(T[])` **  
 该方法需要传入一个参数： `T[]` 类型的数组，代表需要拷贝的数组

 
```
@SuppressWarnings("unchecked")
public <T> T[] toArray(T[] a) {
	if (a.length < size)
        return (T[]) Arrays.copyOf(elementData, size, a.getClass());
    System.arraycopy(elementData, 0, a, 0, size);
    if (a.length > size)
        a[size] = null;
    return a;
}

```
 首先判断判断传入的数组 `a` 能不能存放下该集合所有的元素，如果不够，则调用 `Arrays.copyOf` 其中一个重载方法创建一个新的长度为 `size` 的数组并将集合中所有的元素的引用拷贝进去然后返回。如果足够，调用 `System.arraycopy` 直接拷贝进传入的数组 `a` 。接着，如果满足数组 `a` 的长度大于集合中所有元素的数量，则将数组尾部置为 `null` 作为标记。最后返回拷贝好的数组 `a` 。

 
##### []()（7）trimToSize、ensureCapacity方法

 这两个方法是 `ArrayList` 特有的。主要用于控制数组的长度  
 **void trimToSize()**  
 该方法用于缩减数组长度以减少内存消耗

 
```
public void trimToSize() {
	modCount++;
    if (size < elementData.length) {
        elementData = (size == 0) ? EMPTY_ELEMENTDATA : Arrays.copyOf(elementData, size);
	}
}

```
 调用该方法后， `elementData` 的长度和元素的数量一致。  
 因为 `ArrayList` 只会自动扩容而不会自动缩小长度，所以在必要的时候应当调用 `trimToSize` 控制好长度避免内存浪费

 **void ensureCapacity(int)**  
 该方法需要传入一个int参数

 
```
public void ensureCapacity(int minCapacity) {
    int minExpand = (elementData != DEFAULTCAPACITY_EMPTY_ELEMENTDATA) ? 0 : DEFAULT_CAPACITY;
    if (minCapacity > minExpand)
        ensureExplicitCapacity(minCapacity);
}

```
 该方法会让数组分配一个指定大小为 `minCapacity` 长度的数组，如果 `minCapacity` 小于 `elementData` 的数组长度，那么会被忽略。  
 如果通过无参构造器构造 `ArrayList` 之后再调用该方法，那么最少也会分配一个长度为10的数组（即使传入的参数小于10）。

 
##### []()（8）addAll方法

 `addAll` 有两个重载的方法： `boolean addAll(Collection<? extends E>)` 和 `boolean addAll(int, Collection<? extends E>)`   
 **boolean addAll(Collection<? extends E> c)**  
 该方法将传入的集合c中所有的元素添加到 `elementData` 的尾部，并返回true（除非 `c` 没有任何元素）

 
```
public boolean addAll(Collection<? extends E> c) {
    Object[] a = c.toArray();
    int numNew = a.length;
    ensureCapacityInternal(size + numNew);  // Increments modCount
    System.arraycopy(a, 0, elementData, size, numNew);
    size += numNew;
	return numNew != 0;
}

```
 首先调用集合 `c` 的 `toArray` 方法获取到这个集合包含的所有对象，然后调用 `ensureCapacityInternal` 方法确保 `elementData` 有足够的空间，最后再把元素添加到 `elementData` 。  
 **boolean addAll(int, Collection<? extends E>)**  
 这个方法将传入的集合中所有的元素从指定的数组下标添加。

 
```
public boolean addAll(int index, Collection<? extends E> c) {
	//检查index是否越界
	rangeCheckForAdd(index);

    Object[] a = c.toArray();
    int numNew = a.length;
    //确保elementData长度足够
    ensureCapacityInternal(size + numNew);  // Increments modCount
	
	//移动数组元素
    int numMoved = size - index;
    if (numMoved > 0)
        System.arraycopy(elementData, index, elementData, index + numNew, numMoved);

	//将数组a(包含集合c的元素)从elementData的index处开始添加
    System.arraycopy(a, 0, elementData, index, numNew);
    size += numNew;
    //返回true，除非a.length == 0
    return numNew != 0;
}

```
 
##### []()（9）removeAll、retainAll方法

 **boolean removeAll(Collection<?> c)**  
 该方法删除 `elementData` 与集合 `c` 的交集部分

 
```
public boolean removeAll(Collection<?> c) {
	//集合c不可为null，否则抛出NullPointerException
	Objects.requireNonNull(c);
    return batchRemove(c, false);
}

private boolean batchRemove(Collection<?> c, boolean complement) {
    final Object[] elementData = this.elementData;
    int r = 0, w = 0;
    boolean modified = false;
    try {
	    //遍历elementData元素，将集合c中没有的元素依次赋值到elementData
        for (; r < size; r++)
            if (c.contains(elementData[r]) == complement)
                elementData[w++] = elementData[r];
    //finally语句主要是防止c.contains有异常抛出时能保证elementData数据的完整性
    } finally {
	    //如果没有遍历完elementData
        if (r != size) {
	        //将没有遍历到的元素复制到elementData
            System.arraycopy(elementData, r, elementData, w, size - r);
            //size - r的大小等于没有遍历到的元素
            w += size - r;
        }
        //如果有元素被删除
        if (w != size) {
	        //将多余元素设为null
            for (int i = w; i < size; i++)
	            elementData[i] = null;
            modCount += size - w;
            size = w;
            modified = true;
        }
    }
    return modified;
}

```
 如果没有理解上面这个算法可以看图解：  
 ![这里写图片描述](https://img-blog.csdn.net/2018082916531417?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FiYzEyM2x6Zg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)  
 如果Collection c为 `ArrayList` ，那么该算法的时间复杂度为 `O(n^2)` 。

 **boolean retainAll(Collection<?> c)**  
 该方法删除elementData和集合c的差集部分

 
```
public boolean retainAll(Collection<?> c) {
	//集合c不可为null，否则抛出NullPointerException
    Objects.requireNonNull(c);
	return batchRemove(c, true);
}

```
 同样调用了私有方法 `batchRemove` ，只不过 `complement` 参数为 `true` 

 
##### []()（10）iterator方法

 `iterator` 方法继承自 `Collection` 接口，用于返回该元素的迭代器用于遍历集合内的元素

 
```
 public Iterator<E> iterator() {
	return new Itr();
}

```
 `ArrayList` 的 `iterator` 方法实现是返回内部类 `Itr` ， `Itr` 类实现了 `Iterator` 接口

 
```
private class Itr implements Iterator<E> {
	int cursor;
    int lastRet = -1; 
    int expectedModCount = modCount;
    //省略其它方法...
}

```
 `Itr` 类有三个成员变量：  
  `cursor` 变量用于记录下一个迭代的元素的数组下标  
  `lastRet` 变量用于记录上一次返回的元素的数组下标  
  `expectedModCount` 则是 `modCount` 的值，主要用来检测在迭代器使用期间有没有修改过 `ArrayList` ，修改了之后如果调用迭代器内的 `next` 方法，则会抛出 `ConcurrentModificationException` 异常。

 在了解Itr源码之前，我们先来回顾下 `Iterator` 接口

 
```
public interface Iterator<E> {
	//是否还有下一个元素，如果返回false则代表迭代完成
    boolean hasNext();
	//返回下一个元素
    E next();
	//移除最后一个调用next返回的元素，默认实现为不支持此操作
    default void remove() {
        throw new UnsupportedOperationException("remove");
    }
	//JDK 1.8引入的方法，Consumer为函数式接口，调用该方法并传入一个Consumer函数可以自动
	//为每一个元素执行函数中定义的操作
    default void forEachRemaining(Consumer<? super E> action) {
        Objects.requireNonNull(action);
        while (hasNext())
            action.accept(next());
    }
}

```
 现在我们再来看 `Itr` 对这些方法的实现

 
```
@SuppressWarnings("unchecked")
public E next() {
	//检查ArrayList有没有被修改过
	checkForComodification();
	//获取需要返回的数组下标
    int i = cursor;
    //如果迭代完成则抛出异常
    if (i >= size)
        throw new NoSuchElementException();
    Object[] elementData = ArrayList.this.elementData;
    //如果越界则抛出异常
    if (i >= elementData.length)
        throw new ConcurrentModificationException();
    cursor = i + 1;
    return (E) elementData[lastRet = i];
}

public void remove() {
	//保证没有连续两次调用remove方法或没有调用过next方法
	if (lastRet < 0)
		throw new IllegalStateException();
	//检查ArrayList有没有被修改过
	checkForComodification();
    try {
	    //调用ArrayList实例的remove方法移除
		ArrayList.this.remove(lastRet);
		//将cursor减1
        cursor = lastRet;
        //防止连续两次调用此方法
        lastRet = -1;
        //移除一个对象后，modCount会自增1
        expectedModCount = modCount;
    } catch (IndexOutOfBoundsException ex) {
		throw new ConcurrentModificationException();
    }
}

@Override @SuppressWarnings("unchecked")
public void forEachRemaining(Consumer<? super E> consumer) {
	Objects.requireNonNull(consumer);
    final int size = ArrayList.this.size;
    int i = cursor;
    if (i >= size) {
	    return;
    }
    final Object[] elementData = ArrayList.this.elementData;
    if (i >= elementData.length) {
	    throw new ConcurrentModificationException();
    }
    //从cursor开始执行consumer中的accpet方法直到遍历完成
    while (i != size && modCount == expectedModCount) {
	    consumer.accept((E) elementData[i++]);
    } 
    //更新cursor、lastRet的值
    cursor = i;
    lastRet = i - 1;
    checkForComodification();
}

```
 关于迭代器的方法还有 `listIterator()` 和 `listIterator(int)` ，具体实现也大同小异，这里就不再详细讨论了。

 
### []()三、其他要点

 
##### []()（1）ArrayList的序列化

 先来回顾下：  
 在序列化和反序列化过程中需要特殊处理的类必须使用下列准确签名来实现特殊方法：  
  `private void writeObject(java.io.ObjectOutputStream out) throws IOException`   
  `private void readObject(java.io.ObjectInputStream in) throws IOException, ClassNotFoundException`   
  `writeObject` 用来写入信息， `readObject` 用于读取信息

 `ArrayList` 存储对象的 `elementData` 是用 `transient` 修饰的，那为什么在反序列化的时候仍可将其读出呢，答案就在 `writeObject` 方法中。

 
```
private void writeObject(ObjectOutputStream s) throws IOException{
	int expectedModCount = modCount;
    s.defaultWriteObject();
    s.writeInt(size);
 
    for (int i=0; i<size; i++)
		s.writeObject(elementData[i]);
	//防止在序列化过程中有尝试修改ArrayList的行为
    if (modCount != expectedModCount) {
        throw new ConcurrentModificationException();
    }
}

```
 可以看出， `writeObject` 通过一个 `for` 循环将 `elementData` 里面所有的元素写入序列化文件。

 
```
private void readObject(java.io.ObjectInputStream s) throws IOException, ClassNotFoundException {
        elementData = EMPTY_ELEMENTDATA;
        s.defaultReadObject();
        s.readInt(); 
        if (size > 0) {
            ensureCapacityInternal(size);
            Object[] a = elementData;
            for (int i=0; i<size; i++) {
                a[i] = s.readObject();
		}
	}
}

```
 在读序列化文件的时候，先读出 `size` 元素的值，再根据 `size` 分配足够大的数组，然后通过 `for` 循环将数组中的元素读入。

 `ArrayList` 通过这种方式读取的好处是可以节省内存空间，因为在读的时候会根据元素的实际大小分配数组，而不会预留空间（除非小于元素数量小于10）。

 
##### []()（2）ArrayList的RandomAccess

 `ArrayList` 实现类 `RandomAccess` 接口（是一个标记接口），说明它支持快速访问。  
  `RandomAccess` 接口主要用在 `Collections` 工具类上， `Collections` 提供了大量静态方法操作集合，在需要遍历元素的时候，会根据一个集合是否实现了 `RandomAccess` 接口来采取用 `for` 循环遍历还是用迭代器遍历。对于 `ArrayList` 来说，采用for循环遍历更快，对于 `LinkedList` 来说，采用迭代器遍历更快。  
 可以参考这篇博客：[https://blog.csdn.net/weixin_39148512/article/details/79234817](https://blog.csdn.net/weixin_39148512/article/details/79234817)

 如果本文有错误或者对有任何疑问，欢迎在评论区留言。

   
  