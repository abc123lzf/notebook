---
title: java.util.LinkedList源码解析
date: 2018-08-30 17:31:29
tags: CSDN迁移
---
 版权声明：尊重原创，转载请注明出处 [ https://blog.csdn.net/abc123lzf/article/details/82192852]( https://blog.csdn.net/abc123lzf/article/details/82192852)   
  ### 一、概述

 LinkedList是一种基于双向链表的List实现，相比于ArrayList，它在添加元素的操作上更快。   
 下面是LinkedList的继承类图   
 ![这里写图片描述](https://img-blog.csdn.net/20180829203420696?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FiYzEyM2x6Zg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)   
 LinkedList除了可以作为List以外，还可用作双端队列

 
--------
 
### 二、源码分析

 首先来看LinkedList类定义和成员变量：

 
```
public class LinkedList<E> extends AbstractSequentialList<E>
    implements List<E>, Deque<E>, Cloneable, java.io.Serializable {
    transient int size = 0;
    transient Node<E> first;
    transient Node<E> last;
    //省略其它...
}
```
 LinkedList除了实现List、Deque接口外，也实现了Cloneable和Serializable接口   
 LinkedList只有三个实例变量：   
 int size：LinkedList包含的元素数量   
 Node<E> first：链表的头结点   
 Node<E> last：链表的尾结点   
 此外，父类AbstractList还包含一个modCount记录修改（写入、删除等操作）次数

 LinkedList用一个内部类Node来表示一个元素

 
```
private static class Node<E> {
    E item;
    Node<E> next;
    Node<E> prev;

    Node(Node<E> prev, E element, Node<E> next) {
            this.item = element;
            this.next = next;
            this.prev = prev;
    }
}
```
 Node有三个实例变量：   
 E item：保存元素的引用   
 Node<E> next：下一个Node结点的引用   
 Node<E> prev：上一个Node结点的引用

 **下面我们对几个比较常用的方法进行解析：**

 
--------
 
##### （1）构造方法

 LinkedList提供了两个构造方法：

 
```
public LinkedList() {
}

public LinkedList(Collection<? extends E> c) {
    this();
    addAll(c);
}
```
 对于第二个构造方法可以传入一个Collection集合，这个构造方法内部为调用addAll方法将集合传入的集合中所有的元素添加到LinkedList中。关于addAll方法的实现细节，我们待会再做讨论。

 
--------
 
##### （2）add方法

 add有两个重载方法：boolean add(E e)，add(int index, E element)   
 **boolean add(E e)**   
 add方法向LinkedList中的链表尾端添加元素，该方法始终返回true

 
```
public boolean add(E e) {
    linkLast(e);
    return true;
}

void linkLast(E e) {
    //将最后一个结点的引用赋给l
    final Node<E> l = last;
    //构建一个新结点，并将这个结点的prev指针指向l
    final Node<E> newNode = new Node<>(l, e, null);
    //将这个新的结点作为这个List的尾结点
    last = newNode;
    //如果这个List在添加元素前为空
    if (l == null)
        //将这个新结点作为首结点
        first = newNode;
    else
        //将原尾结点的next引用指向新结点
        l.next = newNode;
    size++;
    modCount++;
}
```
 linkLast方法功能是向链表尾端添加元素。该方法构造一个新的Node，然后作为LinkedList新的尾结点，如果这个时候LinkedList为空，那么还将这个新结点作为LinkedList的首结点，否则将原来的尾结点的next属性引用到这个新的结点。该方法的时间复杂度为O(1)

 **void add(int index, E element)**   
 该方法除了需要提供添加的元素引用，还需要提供一个int参数表示添加的位置（从头结点开始数）

 
```
public void add(int index, E element) {
    //判断index有没有越界(index从0计算，因为是添加操作所以index==size不会报错)
    //否则抛出IndexOutOfBoundsException异常
    checkPositionIndex(index);
    //当index==size时表示从尾端添加元素
    if (index == size)
        linkLast(element);
    //否则在原来位置为index结点之前添加元素
    else
        linkBefore(element, node(index));
}
//根据位置查找结点
Node<E> node(int index) {
    //如果index小于size / 2，那么从头结点开始查找
    if (index < (size >> 1)) {
        Node<E> x = first;
        for (int i = 0; i < index; i++)
            x = x.next;
        return x;
    //否则从尾结点开始查找
    } else {
        Node<E> x = last;
        for (int i = size - 1; i > index; i--)
            x = x.prev;
        return x;
    }
}
//在结点succ之前添加元素e
void linkBefore(E e, Node<E> succ) {
    final Node<E> pred = succ.prev;
    //构建新结点，将新结点的prev指向succ结点的前一个结点，将next指向succ
    final Node<E> newNode = new Node<>(pred, e, succ);
    //重新设置succ的prev指向新结点
    succ.prev = newNode;
    //如果succ就是头结点，那么将新结点作为List的头结点
    if (pred == null)
        first = newNode;
    //否则将原succ的prev节点的next指向新的结点
    else
        pred.next = newNode;
    size++;
    modCount++;
}
```
 可以看出该操作的时间复杂度为O(n)。

 
--------
 
##### （3）get、set方法

 **public E get(int index)**   
 get方法需要一个int参数，表示从头结点开始的元素位置

 
```
public E get(int index) {
    //index的范围只能是index>=0&&index<size
    checkElementIndex(index);
    return node(index).item;
}
```
 get方法本质上就是调用node方法获取到Node结点然后返回这个Node结点的item属性。   
 时间复杂度为O(n)。   
 注意在使用LinkedList遍历元素的时候，应当调用iterator迭代器遍历元素，而不应当使用for循环然后get(i)遍历，否则会使时间复杂度上升到O(n^2)，降低程序运行效率。对于ArrayList则可以使用for循环遍历，因为ArrayList内部实现是数组，其get方法时间复杂度只有O(1)。

 **public E set(int index, E element)**   
 set方法会将index位置上的Node的item属性替换成element，并返回被替换掉的元素引用

 
```
public E set(int index, E element) {
    //index的范围只能是index>=0&&index<size
    checkElementIndex(index);
    //获取这个位置上的结点
    Node<E> x = node(index);
    //替换掉x.item
    E oldVal = x.item;
    x.item = element;
    //返回原来的item
    return oldVal;
}
```
 时间复杂度为O()

 
--------
 
##### （4）remove方法

 remove有两个重载的方法：E remove(int)，boolean remove(Object)   
 **E remove(int)**   
 该方法会将index位置上的Node删除，并返回被删除的元素引用

 
```
public E remove(int index) {
    //index的范围只能是index>=0&&index<size
    checkElementIndex(index);
    //unlink方法负责删除这个结点
    return unlink(node(index));
}

E unlink(Node<E> x) {
    final E element = x.item;
    final Node<E> next = x.next;
    final Node<E> prev = x.prev;
    //如果这个结点的prev为null，那么这个结点就是List的头结点
    if (prev == null) {
        //将x的下一个结点作为List的头结点
        first = next;
    } else {
        //将x的上一个结点的next指向x的下一个结点
        prev.next = next;
        x.prev = null;
    }
    //如果这个结点的next为null，那么这个结点就是List的尾结点
    if (next == null) {
        //将x的上一个结点作为List的尾结点
        last = prev;
    } else {
        //将x的下一个结点的prev指向x的上一个结点
        next.prev = prev;
        x.next = null;
    }
    x.item = null;
    size--;
    modCount++;
    //返回被删除的元素
    return element;
}
```
 时间复杂度为O(n)

 **boolean remove(Object o)**   
 该方法删除从头结点开始和对象o相同的结点（通过equals判断而不是直接通过==判断）

 
```
public boolean remove(Object o) {
    //如果o为null，那么删除第一个item为null的Node
    if (o == null) {
        for (Node<E> x = first; x != null; x = x.next) {
            if (x.item == null) {
                unlink(x);
                return true;
            }
        }
    //否则根据equals判断是否相同
    } else {
        for (Node<E> x = first; x != null; x = x.next) {
            if (o.equals(x.item)) {
                unlink(x);
                return true;
            }
        }
    }
    //没有找到符合要求的结点则返回false
    return false;
}
```
 时间复杂度为O(n)

 
--------
 
##### （5）toArray方法

 toArray有两个重载方法：Object[] toArray()和<T> T[] toArray(T[] a)，这两个方法都是Collection接口的实现   
 **Object[] toArray()**

 
```
public Object[] toArray() {
    Object[] result = new Object[size];
    int i = 0;
    for (Node<E> x = first; x != null; x = x.next)
        result[i++] = x.item;
    return result;
}
```
 这个方法的实现很简单，就是构建一个size大小的Object数组，再遍历链表把元素添加到数组。

 **<T> T[] toArray(T[] a)**   
 toArray方法会尝试把List中所有的元素写入到数组a中并返回a，如果数组a的长度不够，那么这个方法会对a构建一个长度足够的新数组并将元素的引用拷贝到这个数组中然后返回。

 
```
@SuppressWarnings("unchecked")
public <T> T[] toArray(T[] a) {
    //如果a的长度不够，那么会对a新分配一个数组
    if (a.length < size)
        a = (T[])java.lang.reflect.Array.newInstance(a.getClass().getComponentType(), size);
    //同样类似于toArray()方法，遍历集合添加元素
    int i = 0;
    Object[] result = a;
    for (Node<E> x = first; x != null; x = x.next)
        result[i++] = x.item;
    //如果还有多余的空间，在尾部添加一个null表示结束
    if (a.length > size)
        a[size] = null;
    return a;
}
```
 
--------
 
##### （6）addAll方法

 addAll方法有两个重载的方法：boolean addAll(Collection<? extends E> c)和boolean addAll(int index, Collection<? extends E> c)，这些方法可以将集合c中所有的元素添加到LinkedList中。这两个重载的方法在实际实现上可以看成是一个方法。

 
```
public boolean addAll(Collection<? extends E> c) {
    //当index==size时表示从链表尾部添加元素
    return addAll(size, c);
}

public boolean addAll(int index, Collection<? extends E> c) {
    //index的范围可以为index>=0&&index<=size
    checkPositionIndex(index);
    //获取集合c包含所有元素引用的数组
    Object[] a = c.toArray();

    //如果集合c中没有元素，则直接返回false
    int numNew = a.length;
    if (numNew == 0)
        return false;

    Node<E> pred, succ;
    //如果从链表尾端添加元素，则将List尾结点引用赋值给pred,succ置为null
    if (index == size) {
        succ = null;
        pred = last;
    //如果不是从尾端添加元素，则将index位置的Node赋给succ, 再将这个Node的上一个结点引用赋给pred
    } else {
        succ = node(index);
        pred = succ.prev;
    }
    //遍历集合c中的元素，将每个元素构建一个Node并组装到链表中
    for (Object o : a) {
        @SuppressWarnings("unchecked") 
        E e = (E) o;
        Node<E> newNode = new Node<>(pred, e, null);
        if (pred == null)
            first = newNode;
        else
            pred.next = newNode;
        pred = newNode;
    }

    //如果是在链表尾端添加元素，则将List的尾结点置为pred
    if (succ == null) {
        last = pred;
    } else {
        pred.next = succ;
        succ.prev = pred;
    }
    size += numNew;
    modCount++;
    return true;
}
```
 
##### （7）iterator方法

 iterator方法用于返回一个遍历该集合元素的迭代器，这个方法来自Collection接口的定义。   
 iterator方法的实现在LinkedList的父类：AbstractSequentialList中，LinkedList并没有重写该方法：

 
```
public Iterator<E> iterator() {
    return listIterator();
}
```
 这个方法默认实现是调用listIterator方法，该方法返回ListIterator（这个接口继承了Iterator），这个方法的实现在AbstractList中：

 
```
public ListIterator<E> listIterator() {
    return listIterator(0);
}
```
 LinkedList重写了listIterator(int)方法：

 
```
public ListIterator<E> listIterator(int index) {
    checkPositionIndex(index);
    return new ListItr(index);
}
```
 该方法接受一个int参数，这个参数表示开始迭代的位置（从头结点开始计算）   
 首先会检查index是否越界，然后会构建一个内部类ListItr返回。   
 在了解ListItr之前，我们先来了解一下ListIterator接口（只列出Iterator没有的方法）

 
```
public interface ListIterator<E> extends Iterator<E> {
    //是否存在上一个元素
    boolean hasPrevious(); 
    //返回上一个遍历的元素
    E previous();
    //返回下一个元素的index
    int nextIndex(); 
    //返回上一个元素的index
    int previousIndex(); 
    //设置当前元素
    void set(E e);
    //在当前位置添加一个元素
    void add(E e);
}
```
 ListIterator相比于Iterator提供了更多的方法在迭代的时候操作集合，它可以在遍历到一个元素的时候访问上一个元素，也可以在这个替换掉这个元素，还可以添加一个新的元素。   
 对于每一个List，都可以调用listIterator方法获得ListIterator。

 下面来看ListItr类的声明、成员变量及构造方法：

 
```
private class ListItr implements ListIterator<E> {
    //上一次调用next返回的Node引用
    private Node<E> lastReturned;
    //即将遍历到下一个Node引用
    private Node<E> next;
    //下一个Node(next)的index位置
    private int nextIndex;
    //主要用于检测在遍历过程中没有调用非ListItr方法修改LinkedList
    private int expectedModCount = modCount;

    ListItr(int index) {
        //如果index==size代表没有元素可以遍历
        next = (index == size) ? null : node(index);
        nextIndex = index;
    }

    //省略其它方法...
}
```
 下面讲解ListItr的Iterator接口实现部分   
 Iterator接口实现部分：

 
```
public boolean hasNext() {
    //直接判断下一个Node的index是不是小于size
    return nextIndex < size;
}

public E next() {
    //检查List有没有修改过
    checkForComodification();
    //如果已经没有元素可供遍历了，则抛出NoSuchElementException
    if (!hasNext())
        throw new NoSuchElementException();

    lastReturned = next;
    next = next.next;
    nextIndex++;
    return lastReturned.item;
}

//移除最近一次调用next返回的元素
public void remove() {
    checkForComodification();
    if (lastReturned == null)
        throw new IllegalStateException();

    Node<E> lastNext = lastReturned.next;
    //该方法删除结点，并会将modCount的数值加1
    unlink(lastReturned);
    if (next == lastReturned)
        next = lastNext;
    else
        nextIndex--;
    lastReturned = null;
    //修改expectedModCount，保证和外部类的modCount一致
    expectedModCount++;
}

final void checkForComodification() {
    //如果ListItr的expectedModCount属性和modCount属性不一致就说明调用了非ListItr的方法修改了List
    if (modCount != expectedModCount)
        throw new ConcurrentModificationException();
}
```
 ListIterator的实现部分大同小异，这里就不再讲解了。

 
### 四、其它特性

 
##### 1、LinkedList的克隆

 LinkedList实现了Cloneable接口，重写了clone方法，并且为public权限。

 
```
public Object clone() {
    LinkedList<E> clone = superClone();

    clone.first = clone.last = null;
    clone.size = 0;
    clone.modCount = 0;

    for (Node<E> x = first; x != null; x = x.next)
        clone.add(x.item);

    return clone;
}

@SuppressWarnings("unchecked")
private LinkedList<E> superClone() {
    try {
        return (LinkedList<E>) super.clone();
    } catch (CloneNotSupportedException e) {
        //一般不会发生
        throw new InternalError(e);
    }
}
```
 clone方法首先调用Object默认的clone方法，这个默认的clone方法会将原来的LinkedList中的size属性、modCount属性、first引用和last引用赋值到克隆出的LinkedList中。如果只是这样克隆，那么会造成一个问题，如果克隆的LinkedList修改了first引用，那么原来的LinkedList也会被修改，因为这两个List的first、last引用都指向了一个Node。所以LinkedList除了调用默认的clone方法(本地方法)，还会通过一个for循环遍历原来的LinkedList中的链表，将其中的元素添加到克隆的LinkedList，由这个克隆的LinkedList自己构造新的Node，就可以避免上述问题。

 
##### 2、序列化

 LinkedList序列化策略和ArrayList差不多。同样，对于保存数据的结点：first、last都用了transient关键字修饰。LinkedList定义了readObject、writeObject方法实现了自己的序列化策略。

 
```
private void writeObject(java.io.ObjectOutputStream s) throws IOException {
    s.defaultWriteObject();
    //写入size变量
    s.writeInt(size);
    //遍历链表，只将元素的内容写入序列化文件
    for (Node<E> x = first; x != null; x = x.next)
        s.writeObject(x.item);
}

@SuppressWarnings("unchecked")
private void readObject(java.io.ObjectInputStream s) throws IOException, ClassNotFoundException {
    s.defaultReadObject();
    int size = s.readInt();
    //读取元素的内容并构造链表
    for (int i = 0; i < size; i++)
        linkLast((E)s.readObject());
}
```
 在序列化过程中，不会保存整个Node的信息，而是只记录size和元素的内容。

 
##### 3、和ArrayList的一个不同点

 查看ArrayList的源代码，发现ArrayList实现了一个接口：RandomAccess，这个接口是一个标记接口，表示该List支持快速随机访问。所以在调用Collections类的二分搜索等方法时，会采用for循环的方式遍历List。而对于LinkedList而言，则会采用一种迭代器方式遍历集合元素，原因在前文已经提到过了。

   
  