---
title: 浅析Java中的String、StringBuilder和StringBuffer源码
date: 2018-08-25 15:09:06
tags: CSDN迁移
---
 版权声明：尊重原创，转载请注明出处 [ https://blog.csdn.net/abc123lzf/article/details/82026715]( https://blog.csdn.net/abc123lzf/article/details/82026715)   
  `String` 、 `StringBuilder` 、 `StringBuffer` 算得上是一个老生常谈的问题了，在实际使用中会经常用到这些类，了解这些类的使用和原理是十分必要的，本文将根据JDK1.8的源码分析。

 
### []()一、String类

 先来看 `String` 类的源码：

 
```
public final class String
    implements java.io.Serializable, Comparable<String>, CharSequence {
	//存储字符串字符的char数组
	private final char value[];
	//hashCode缓存，只有当第一次调用了hashCode方法时才会被赋值，以后调用该方法时直接返回该变量
    private int hash;

	//省略所有的方法和静态变量
}

```
 单凭上述源码可以总结出：  
 1、 `String` 类不可被继承：该类被final修饰。  
 2、 `String` 类是不可变的： `value[]` 被 `final` 修饰，可以将 `String` 看成是对 `char` 数组的一个封装。  
 3、 `String` 类可以序列化，可以和其他 `String` 比较其大小：实现了 `Comparable` 接口

 `String` 类提供了大量的API处理字符串，但是无论怎么操作，原有的 `String` 都是不变的，调用类似 `substring` 、 `concat` 方法都会返回一个全新的 `String` 对象。

 下面是一个经典的问题：

 
```
public class Test {
	public static void main(String[] args) {
		String a = "abc";
		String b = new String("abc");
		String c = new String("abc");
		String d = b.intern();
		System.out.println(a == b);
		System.out.println(b == c);
		System.out.println(a == d);
	}
}

```
 运行上述代码，输出结果是什么？  
 答案是：  
 false  
 false  
 true

 这是为什么呢？  
 在JVM中，为了减少内存的消耗，存在一个称为**字符串常量池**的区域。当我们通过 `String a = "abc"` 这样创建一个字符串对象时，JVM会首先在字符串常量池中寻找这个字符串，若存在，则将 `a` 直接指向该字符串，若不存在，则在常量池中创建该字符串并将 `a` 指向它，所以 `a==d` 返回 `true` 。当我们通过 `String b = new String("abc")` 这样创建字符串时，情况就不一样了，它会在堆区创建该字符串对象并使 `b` 指向它，同样调用 `String c = new String("abc")` 时，也会在堆区再创建一个 `String` 对象并使 `c` 指向它，JVM并不会判断堆区是否存在其它相同的 `String` 对象，所以 `b==c` 返回 `false` 。当调用 `String d = b.intern();` 时， `intern` 方法（该方法为 `native` 方法）会在字符串常量池中查找是否存在该字符串对象，如果存在，则将 `d` 指向该常量池中的字符串对象，如果不存在则在常量池中创建该字符串并指向它，所以 `a==d` 返回 `true` 。  
 ![这里写图片描述](https://img-blog.csdn.net/20180825003333270?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FiYzEyM2x6Zg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

 
### []()二、StringBuilder

 `StringBuilder` 经常用在需要对字符串进行大量操作的地方，它可以非常高效的对字符串进行拼接等操作。  
 可能会有人问：我同样可以用+号来拼接字符串啊，为什么还要用到 `StringBuilder` ?  
 如果你提出了这样一个疑问，请看下面代码：

 
```
public class Test {
	public static void main(String[] args) {
		String s = "";
		long st = System.currentTimeMillis();
		for(int i = 0; i < 100000; i++) {
			s += "a";
		}
		long ed = System.currentTimeMillis();
		System.out.println("时间：" + (ed - st) + "毫秒");
		
		st = System.currentTimeMillis();
		StringBuilder sb = new StringBuilder();
		for(int i = 0; i < 100000; i++) {
			sb.append("a");
		}
		ed = System.currentTimeMillis();
		System.out.println("时间：" + (ed - st) + "毫秒");
	}
}

```
 运行上述代码，输出：  
 时间：2735毫秒  
 时间：2毫秒

 可以看出，在大量对字符串进行连接操作的情况下， `StringBuilder` 优势非常明显。下面我们根据上述例子，来逐步了解 `StringBuilder` 。  
 先贴出 `StringBuilder` 的源码：

 
```
public final class StringBuilder extends AbstractStringBuilder
    implements java.io.Serializable, CharSequence {
    
    public StringBuilder() {
        super(16);
    }

	public StringBuilder(int capacity) {
        super(capacity);
    }

	public StringBuilder(String str) {
        super(str.length() + 16);
        append(str);
    }

	public StringBuilder(CharSequence seq) {
        this(seq.length() + 16);
        append(seq);
    }
    //省略其他方法
}

```
 `StringBuilder` 继承了 `AbstractStringBuilder` ，和 `String` 一样实现了 `CharSequence` 和 `Serializable` 接口。  
  `StringBuilder` 提供了4个构造器，无参构造器默认创建一个长度为16的 `char` 数组，作为存储字符串的缓冲区。用户也可以传入一个 `int` 参数指定该缓冲区的大小，也同样可以传入一个初始字符串。字符串存储的实现在其父类 `AbstractStringBuilder` 类中。  
 下面来看AbstractStringBuilder源码：

 
```
abstract class AbstractStringBuilder implements Appendable, CharSequence {
	//存储字符串的char数组
    char[] value;
    
	//字符串的长度
    int count;

    AbstractStringBuilder() {
    }
    
    AbstractStringBuilder(int capacity) {
        value = new char[capacity];
    }
	
	public AbstractStringBuilder append(String str) {
		//如果str为null，则直接往后拼接"null"
        if (str == null)
            return appendNull();
        int len = str.length();
        //确保value数组的长度足够，不会越界
        ensureCapacityInternal(count + len);
        //将str从value[count]开始拼接
        str.getChars(0, len, value, count);
        //更新数组长度
        count += len;
        return this;
    }

	//扩展value数组的长度，minimumCapacity的值为新char数组的长度
	private void ensureCapacityInternal(int minimumCapacity) {
        if (minimumCapacity - value.length > 0) {
	        //将旧数组的值拷贝到新数组
            value = Arrays.copyOf(value,
                    newCapacity(minimumCapacity));
        }
    }
	//省略其它方法...
}

```
 和 `String` 不同的是， `value` 数组的长度并不代表字符串的实际长度，而是采用一个变量 `count` 记录字符串长度。每当调用 `append` 、 `insert` 、 `reverse` 方法时，会首先判断 `value` 长度是否充足，如果长度不够，则会重新分配一个新的 `char` 数组并将旧数组拷贝进去，再进行拼接。如果长度足够，则直接将需要拼接的字符串加在数组后面，再修改 `count` 的值即可。  
 当我们需要获取 `StringBuilder` 的字符串时，直接调用 `toString` 即可， `toString` 方法在 `StringBuilder` 中默认是将 `value` 数组从0~count截取并返回一个全新的 `String` 对象。

 了解 `StringBuilder` 拼接原理后，我们再来分析为什么在大量的拼接操作时String速度会明显慢于 `StringBuilder` 。

 
```
public class Test {
	public static void main(String[] args) {
		String s = "";
		for(int i = 0; i < 100000; i++) {
			s += "a";
		}
		System.out.print(s);
	}
}

```
 将上述代码编译成class文件，通过eclipse自带的class文件分析工具得到 `main` 方法的字节码：

 
```
public static void main(java.lang.String[] args);
     0  ldc <String ""> [16]
     2  astore_1 [s]
     3  iconst_0
     4  istore_2 [i]
     5  goto 31
     8  new java.lang.StringBuilder [18]
    11  dup
    12  aload_1 [s]
    13  invokestatic java.lang.String.valueOf(java.lang.Object) : java.lang.String [20]
    16  invokespecial java.lang.StringBuilder(java.lang.String) [26]
    19  ldc <String "a"> [29]
    21  invokevirtual java.lang.StringBuilder.append(java.lang.String) : java.lang.StringBuilder [31]
    24  invokevirtual java.lang.StringBuilder.toString() : java.lang.String [35]
    27  astore_1 [s]
    28  iinc 2 1 [i]
    31  iload_2 [i]
    32  ldc <Integer 100000> [39]
    34  if_icmplt 8
    37  getstatic java.lang.System.out : java.io.PrintStream [40]
    40  aload_1 [s]
    41  invokevirtual java.io.PrintStream.print(java.lang.String) : void [46]
    44  return
      Line numbers:
        [pc: 0, line: 6]
        [pc: 3, line: 7]
        [pc: 8, line: 8]
        [pc: 28, line: 7]
        [pc: 37, line: 10]
        [pc: 44, line: 11]
      Local variable table:
        [pc: 0, pc: 45] local: args index: 0 type: java.lang.String[]
        [pc: 3, pc: 45] local: s index: 1 type: java.lang.String
        [pc: 5, pc: 37] local: i index: 2 type: int
      Stack map table: number of frames 2
        [pc: 8, append: {java.lang.String, int}]
        [pc: 31, same]

```
 分析字节码，可以看出字符串+=操作实际上是一个语法糖，在每次循环中，都会创建一个 `StringBuilder` 对象（指令8），再调用 `String.valueOf(s)` 返回字符串对象（指令12、13），接着调用 `StringBuilder` 的 `append` 方法对字符串 `"a"` 进行拼接（指令19、21），最后再调用 `StringBuilder` 的 `toString` 方法赋值给变量 `s` （指令24、27），单次循环完成。

 所以事实上，上述代码其实可以看成：

 
```
public class Test {
	public static void main(String[] args) {
		String s = "";
		for(int i = 0; i < 100000; i++) {
			StringBuilder sb = new StringBuilder(s);
			sb.append("a");
			s = sb.toString();
		}
		System.out.print(s);
	}
}

```
 每一次循环都会创建一个 `StringBuilder` 对象并将字符串 `s` 拷贝到自己的 `value` 数组，拼接好一个字符串后再调用 `toString` 再次创建一个 `String` 对象赋值给 `a` ，这样频繁的创建对象和字符串的复制会给堆内存带来比较大的压力，这就是为什么大量的拼接操作中 `StringBuilder` 效率要比 `String` 好。

 
### []()三、StringBuffer

 
```
public final class StringBuffer extends AbstractStringBuilder
    implements java.io.Serializable, CharSequence {
    
	private transient char[] toStringCache;
	//省略其它方法
}

```
 `StringBuffer` 和 `StringBuilder` 一样继承了 `AbstractStringBuilder` ，实现了 `CharSequence` 和 `Serializable` 接口。  
  `StringBuffer` 和 `StringBuilder` 功能一样，不同点在于 `StringBuffer` 中的方法大都用了 `synchronized` 修饰，它可以在多线程环境下安全的使用，而 `StringBuilder` 不是线程安全的。

 `StringBuffer` 比 `StringBuilder` 多了一个 `toStringCache` 缓冲区。

 
```
@Override
public synchronized String toString() {
	if (toStringCache == null) {
		toStringCache = Arrays.copyOfRange(value, 0, count);
    }
    return new String(toStringCache, true);
}

```
 这样在并发环境下大量调用 `toString` 方法时，可以减少很多数组复制操作。当然每次更新 `value` 数组时都会将 `toStringCache` 设为 `null` ，保证 `toString` 返的是最新的字符串。

 相比于 `StringBuilder` ，在单线程环境下我们应当尽量使用 `StringBuilder` ，它减少了加锁的步骤，效率比 `StringBuffer` 更快。而在多线程环境下，应当首先考虑的是线程安全，所以我们应当使用 `StringBuffer` 。

   
  