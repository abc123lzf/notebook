---
title: 深入分析Java中的枚举类（JDK 1.8）
date: 2018-08-27 10:52:26
tags: CSDN迁移
---
 版权声明：尊重原创，转载请注明出处 [ https://blog.csdn.net/abc123lzf/article/details/82084893]( https://blog.csdn.net/abc123lzf/article/details/82084893)   
  ### []()一、概述

 在分析之前，先来回顾下枚举类的基础知识点  
 枚举类一般用于一个类只可能拥有有限个实例，比如季节只可拥有春夏秋冬，性别只有男女  
 枚举类和普通类有以下几个不同点：  
 1、枚举类不能指定继承的父类（因为继承了 `java.lang.Enum` 类），但是可以实现多个接口，枚举类默认实现了 `Comparable` 接口和 `Serializable` 接口  
 2、枚举类的构造方法的访问权限只可为 `private`   
 3、枚举类的实例必须显式列出。

 
### []()二、枚举类的真正实现

 我们以下列代码为例来分析枚举类：

 
```
public enum Season {
	SPRING("春天"), SUMMER("夏天"), AUTUMN("秋天"), WINTER("冬天");
	
	private final String chinese;
	
	private Season(String chinese) {
		this.chinese = chinese;
	}

	public String getChinese() {
		return chinese;
	}
}

```
 其实上述代码相当于：

 
```
public final class Season extends Enum<Season> {
	public static final Season SPRING;
	public static final Season SUMMER;
	public static final Season AUTUMN;
	public static final Season WINTER;
	private static final Season[] ENUM$VALUES;
	
	static {
		SPRING = new Season("SPRING", 0, "春天");
		SUMMER = new Season("SUMMER", 1, "夏天");
		AUTUMN = new Season("AUTUMN", 2, "秋天");
		WINTER = new Season("WINTER", 3, "冬天");
		ENUM$VALUES = new Season[]{SPRING, SUMMER, AUTUMN, WINTER}
	}
	
	private final String chinese;
	
	private Season(String name, int ordinal, String chinese) {
		super(name, ordinal);
		this.chinese = chinese;
	}

	public String getChinese() {
		return chinese;
	}
	
	public static Season[] values() {
		Season[] arr = new Session[ENUM$VALUES.length];
		System.arraycopy(ENUM$VALUES, 0, arr, 0, arr.length);
		return arr;
	}
	
	public static Season valueOf(String name) {
		return Enum.valueOf(Season.class, name);
	}
}

```
 在编译过程中，编译器会对枚举类进行以下处理：  
 1、增加一个静态初始化块和一个静态数组，静态初始化块负责构造枚举类的实例并将其保存到数组中。  
 2、修改当前枚举类的构造方法，向构造方法前面增加两个参数： `name` ,  `ordinal` 。分别代表枚举类的名称和初始化顺序，并且使用 `super` 调用父类 `java.lang.Enum` 的构造方法。  
 3、添加两个静态方法：  
  `public static Season[] values()`   
 该方法返回一个数组，包含了该枚举类所有的实例（并不是返回枚举类自己保存实例的数组，而是通过对其复制，以确保实例不被篡改）。  
  `public static Season valueOf(String name)`   
 该方法通过当前枚举类的名称返回对应的实例。比如 `valueOf("SPRING")` 返回 `Season.SPRING` ）。

 
### []()三、java.lang.Enum源码解析

 看完枚举类实际实现后，我们再来分析Enum类的源码。

 
```
public abstract class Enum<E extends Enum<E>>
        implements Comparable<E>, Serializable {
	//枚举的名称，以上面Season.SPRING为例，这里name就为"SPRING"
    private final String name;

    public final String name() {
        return name;
    }
	//枚举的顺序，从0开始，比如SPRING为0，SUMMER为1
    private final int ordinal;

    public final int ordinal() {
        return ordinal;
    }
	//只可由子类访问，即枚举类
    protected Enum(String name, int ordinal) {
        this.name = name;
        this.ordinal = ordinal;
    }

    public String toString() {
        return name;
    }
	//实现和Object一样，只不过加了final防止重写，下面的hashCode也一样
    public final boolean equals(Object other) {
        return this == other;
    }

    public final int hashCode() {
        return super.hashCode();
    }
	//只可和同一个枚举类的枚举比较
    public final int compareTo(E o) {
        Enum<?> other = (Enum<?>)o;
        Enum<E> self = this;
        if (self.getClass() != other.getClass() &&
            self.getDeclaringClass() != other.getDeclaringClass())
            throw new ClassCastException();
        //通过比较枚举的顺序判定大小
        return self.ordinal - other.ordinal;
    }

    @SuppressWarnings("unchecked")
    public final Class<E> getDeclaringClass() {
        Class<?> clazz = getClass();
        Class<?> zuper = clazz.getSuperclass();
        return (zuper == Enum.class) ? (Class<E>)clazz : (Class<E>)zuper;
    }

    public static <T extends Enum<T>> T valueOf(Class<T> enumType, String name) {
        T result = enumType.enumConstantDirectory().get(name);
        if (result != null)
            return result;
        if (name == null)
            throw new NullPointerException("Name is null");
        throw new IllegalArgumentException(
            "No enum constant " + enumType.getCanonicalName() + "." + name);
    }
	//不可重写finalize
    protected final void finalize() { }

    private void readObject(ObjectInputStream in) throws IOException,
        ClassNotFoundException {
        throw new InvalidObjectException("can't deserialize enum");
    }

    private void readObjectNoData() throws ObjectStreamException {
        throw new InvalidObjectException("can't deserialize enum");
    }
}

```
 ** `Enum` 类有两个成员变量： `name` 和 `ordinal` **，前面提过了分别代表枚举类的名称和初始化顺序。这两个字段被 `final` 关键字修饰，所以一个枚举类在构造时必须调用其父类 `Enum` 的构造方法完成赋值操作，前面同样也说过了编译器会自动修改子类的构造方法。

 ** `Enum` 类重写了 `Object` 类的四个方法： `finalize` ， `toString` ,  `hashCode` ,  `equals` ， `clone` 方法**  
 其中， `finalize` ， `hashCode` ,  `equals` ， `clone` 方法被 `final` 关键字修饰，枚举类不可重写这些方法。下面来分析这些方法被 `final` 关键字修饰的原因：  
 1、 `finalize` ：由于枚举类的实例被 `static final` 变量强引用，所以不可被垃圾回收，自然重写 `finalize` 就没有任何意义了。  
 2、 `hashCode` ：在 `Enum` 类中该方法的实现就是调用 `Object` 里的 `hashCode` 方法，枚举类的实例是不变的，并且重写这个方法并不会比 `Object` 的 `hashCode` 本地实现更快，如果你想深入了解 `Object` 里的 `hashCode` 方法实现原理，可以参考这篇博客：  
 [https://blog.csdn.net/xusiwei1236/article/details/45152201。](https://blog.csdn.net/xusiwei1236/article/details/45152201%E3%80%82)  
 3、 `equals` ：枚举类的实例是不变的，所以直接用 `==` 判断是否相等就可以了  
 4、 `clone` ：枚举类的实例不可被克隆

 `Enum` 类的 `toString` 方法默认返回枚举类的名称（即 `name` 属性），子类可以重写该方法。

 ** `Enum` 实现了 `Comparable` 接口**  
 一个枚举类实例之间是可比较大小的，不同枚举类的实例不可比较大小。  
  `compareTo` 方法的实现是通过 `ordinal` 成员变量（即枚举类实例初始化顺序）判断大小，且该方法被 `final` 修饰，不可被重写。

 **静态方法 `valueOf` 解析**  
  `Enum` 提供了静态方法 `valueOf` ，使其可以通过枚举类对应 `Class` 类和实例的名称获取到枚举实例。  
 比如调用 `Enum.valueOf(Season.class, "SPRING")` 返回的是 `Season.SPRING`   
 如果传入的 `name` 为 `null` ，则会抛出 `NullPointerException` 异常，如果没有找到对应的实例，则会抛出 `IllegalArgumentException` 异常。

 该方法通过反射获取实例。

 
```
T result = enumType.enumConstantDirectory().get(name);

```
 继续分析：

 
```
public final class Class<T> implements java.io.Serializable, GenericDeclaration,
		Type, AnnotatedElement{

	private volatile transient Map<String, T> enumConstantDirectory = null;

	Map<String, T> enumConstantDirectory() {
        if (enumConstantDirectory == null) {
            T[] universe = getEnumConstantsShared();
            if (universe == null)
                throw new IllegalArgumentException(getName() + " is not an enum type");
            Map<String, T> m = new HashMap<>(2 * universe.length);
            for (T constant : universe)
                m.put(((Enum<?>)constant).name(), constant);
            enumConstantDirectory = m;
        }
        return enumConstantDirectory;
	}
	
    private volatile transient T[] enumConstants = null;
    
	T[] getEnumConstantsShared() {
        if (enumConstants == null) {
            if (!isEnum()) return null;
            try {
	            //注意这里
                final Method values = getMethod("values");
                java.security.AccessController.doPrivileged(
                    new java.security.PrivilegedAction<Void>() {
                        public Void run() {
                                values.setAccessible(true);
                                return null;
                            }
                        });
                @SuppressWarnings("unchecked")
                T[] temporaryConstants = (T[])values.invoke(null);
                enumConstants = temporaryConstants;
            } catch (InvocationTargetException | NoSuchMethodException |
                   IllegalAccessException ex) { return null; }
        }
        return enumConstants;
    }
}

```
 `getEnumConstantsShared` 方法实际上是调用了 `Season` 枚举类的 `values` 方法返回了保存当前枚举类所有实例类的数组，再将这个数组赋值给 `Class` 对象的 `enumConstants` 成员变量并返回给 `enumConstantDirectory` 方法。 `enumConstantDirectory` 方法将这个数组里的实例以键值对形式（枚举名为键，对应的对象为值）保存到 `enumConstantDirectory` 这个Map中，并返回这个Map，然后直接调用Map的get方法并传入 `name` 就可以获取到我们想要的枚举对象了。

 **Enum的序列化**  
 虽然说 `Enum` 实现了 `Serializable` 接口，但实际上枚举类的序列化只是将名称(成员变量 `name` )输出，反序列化则是默认调用 `Enum` 静态方法 `valueOf` 。  
 可以看下面这一段测试代码：

 
```
public class Test {
	public static void main(String[] arg) 
			throws IOException, ClassNotFoundException {
		FileOutputStream fos = new FileOutputStream("D:\\out.txt");
		ObjectOutputStream oos = new ObjectOutputStream(fos);
		oos.writeObject(Season.SPRING);
		oos.close();
		FileInputStream fis = new FileInputStream("D:\\out.txt");
		ObjectInputStream ois = new ObjectInputStream(fis);
		Object obj = ois.readObject();
		System.out.println(obj.getClass().getName());
		System.out.println(obj == Season.SPRING);
	}
}

```
 序列化文件out.txt内容为：  
 ![这里写图片描述](https://img-blog.csdn.net/2018082710194893?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FiYzEyM2x6Zg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)  
 序列化文件只保存了 `name` 属性的值和对象类型

 运行上述代码，输出结果为：  
 com.test.Season  
 true  
 可以看出反序列化只是通过序列化文件中 `name` 属性调用 `valueOf` 返回 `Season` 类已经构造好的实例而已。

   
  