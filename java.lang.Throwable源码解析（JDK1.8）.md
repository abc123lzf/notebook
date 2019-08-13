---
title: java.lang.Throwable源码解析（JDK1.8）
date: 2018-08-28 10:35:58
tags: CSDN迁移
---
 版权声明：尊重原创，转载请注明出处 [ https://blog.csdn.net/abc123lzf/article/details/82115048]( https://blog.csdn.net/abc123lzf/article/details/82115048)   
  ### []()一、基础知识复习

 `Throwable` 是所有异常类的父类，它提供了一系列API为我们反馈异常的信息。

 在分析 `Throwable` 源码前，我们先来回顾一下Java异常的基础知识：  
 1、Java提供了try…catch…finally…语法来实现异常处理机制，在 `try` 语句中一旦抛出一个异常对象（ `Throwable` 或其子类），如果找到了对应的 `catch` 语句块，都可以将其捕获并将异常对象作为形参传入 `catch` 语句块中进行处理，如果没有找到合适的 `catch` 语句块，则会使当前线程停止。无论有没有异常， `finally` 语句块都会执行（除了强制退出Java程序）。  
 2、Java将异常分为两类：异常（ `Exception` ）和错误（ `Error` ），它们都继承 `Throwable` 类。 `Error` 一般指与JVM相关的问题：如崩溃、虚拟机内部错误等，这种错误无法恢复，会导致应用程序中断，一般情况下程序不需要捕获 `Error` 。 `Exception` 一般指程序可以预知的异常并且可以正常的处理，这种类型的异常需要捕获并处理。  
 3、Java的 `Exception` 还可以分为两类：Checked异常类和Runtime异常类，所有 `RuntimeException` 类及其子类被称为Runtime异常类，不是 `RuntimeException` 子类并且以 `Exception` 为父类的异常类为Checked异常类。对于Checked异常，程序必须由try…catch语句来捕获此异常，如果没有捕获则无法通过编译。而Runtime异常则不一样，Runtime异常无须显式声明抛出，当然也可以通过try…catch捕获。常见的 `RuntimeException` 子类有 `IndexOutOfBoundsException` ,  `NumberFormatException` 。  
 4、在Java 7以后的版本，可以将实现了 `Closeable` 接口的类（比如 `InputStream` ）放在try(…){}小括号中，这样可以在try语句块执行完毕后自动关闭它们，而不必在finally语句显式关闭。  
 5、子类方法声明抛出的异常类型应当是父类方法声明抛出的异常类型的子类或是同一个类，子类方法声明抛出的异常不允许比父类方法声明抛出的多。  
 6、不要在 `finally` 块调用 `return` ， `finally` 中的 `return` 会覆盖掉try…catch…的返回值，因为 `finally` 语句实际上是在 `return` 语句在真正返回之前执行。

 
### []()二、源码解析

 分析 `Exception` 和 `Error` 类源码，发现这些类都只是调用了父类 `Throwable` 的构造器。  
 既然 `Exception` 和 `Error` 继承了 `java.lang.Throwable` 类，所以了解 `Throwable` 类的源码是十分必要的

 
##### []()下面通过我们先通过成员变量来逐步了解 `Throwable` 类：

 
```
//静态变量
private static class SentinelHolder {
	public static final StackTraceElement STACK_TRACE_ELEMENT_SENTINEL =
		new StackTraceElement("", "", null, Integer.MIN_VALUE);
	public static final StackTraceElement[] STACK_TRACE_SENTINEL =
        new StackTraceElement[] {STACK_TRACE_ELEMENT_SENTINEL};
}

private static final StackTraceElement[] UNASSIGNED_STACK 
		= new StackTraceElement[0];
private static final List<Throwable> SUPPRESSED_SENTINEL =
        Collections.unmodifiableList(new ArrayList<Throwable>(0));
private static final String NULL_CAUSE_MESSAGE = "Cannot suppress a null exception.";
private static final String SELF_SUPPRESSION_MESSAGE = "Self-suppression not permitted";
private static final String CAUSE_CAPTION = "Caused by: ";
private static final String SUPPRESSED_CAPTION = "Suppressed: ";
private static final Throwable[] EMPTY_THROWABLE_ARRAY = new Throwable[0];

//实例变量
private transient Object backtrace;
private String detailMessage;
private Throwable cause = this;
private StackTraceElement[] stackTrace = UNASSIGNED_STACK;
private List<Throwable> suppressedExceptions = SUPPRESSED_SENTINEL;

```
 **实例变量：**  
 **（1） `Object backtrace` ** ：这个变量由native方法赋值，用来保存栈信息的轨迹（根据注释翻译的，具体作用不太清楚）。  
 **（2） `String detailMessage` ** ：描述这个异常的信息，比如 `new Exception("This is exception")` 传入的就是这个异常的描述信息，默认条件下可以调用 `getMessage` 或 `getLocalizedMessage` 方法获取。

 
```
public String getMessage() {
	return detailMessage;
}
public String getLocalizedMessage() {
	return getMessage();
}

```
 这两个方法都可以被子类重写，一般来说， `getLocalizedMessage` 用来返回本地化描述信息，如果子类有必要实现本地化错误描述信息的话可以重写这个方法。

 **（3） `Throwable cause` ** ：描述这个异常由哪个 `Throwable` 导致，默认是 `this` 。可通过构造器自定义。构造完成后，可以通过 `getCause` 方法访问：

 
```
public synchronized Throwable getCause() {
	return (cause == this ? null : cause);
}

```
 如果没有指定 `cause` ，则返回 `null` 。  
 在构造完成后，也可通过initCause方法修改：

 
```
public synchronized Throwable initCause(Throwable cause) {
	if (this.cause != this)
		throw new IllegalStateException("Can't overwrite cause with " +
                                            Objects.toString(cause, "a null"), this);
	if (cause == this)
        throw new IllegalArgumentException("Self-causation not permitted", this);
    this.cause = cause;
	return this;
}

```
 修改 `cause` 的前提是必须在构造方法中没有指定别的 `cause` （即默认条件下 `cause` 为 `this` ），否则会抛出 `IllegalStateException` 异常。  
 另外也不能修改 `cause` 为 `this` 。

 `cause` 一般用在这种情形中：

 
```
try {
	Integer.valueOf(str);
} catch(NumberFormatException e) {
	throw new MyException(e);
}

```
 相当于给 `NumberFormatException` 加了一层包装，以便更好地处理异常（因为只需要捕获到一种异常类型 `MyException` ）

 **（4） `StackTraceElement[] stackTrace` ** ：这个异常抛出位置的栈信息，每个 `StackTraceElement` 代表一个栈信息，默认指向静态常量 `UNASSIGNED_STACK` ，代表栈信息为空。  
 我们先来看 `StackTraceElement` 源代码

 
```
public final class StackTraceElement implements java.io.Serializable {

    private String declaringClass;
    private String methodName;
    private String fileName;
    private int lineNumber;

    public StackTraceElement(String declaringClass, String methodName,
                             String fileName, int lineNumber) {
        this.declaringClass = Objects.requireNonNull(declaringClass, "Declaring class is null");
        this.methodName = Objects.requireNonNull(methodName, "Method name is null");
        this.fileName = fileName;
        this.lineNumber = lineNumber;
    }

    public String getFileName() {
        return fileName;
    }

    public int getLineNumber() {
        return lineNumber;
    }

    public String getClassName() {
        return declaringClass;
    }

    public String getMethodName() {
        return methodName;
    }

    public boolean isNativeMethod() {
        return lineNumber == -2;
    }

    public String toString() {
        return getClassName() + "." + methodName +
            (isNativeMethod() ? "(Native Method)" :
             (fileName != null && lineNumber >= 0 ?
              "(" + fileName + ":" + lineNumber + ")" :
              (fileName != null ?  "("+fileName+")" : "(Unknown Source)")));
    }

    //省略equals、hashCode方法
}

```
 `StackTraceElement` 保存了一个栈区单个方法方法名、所属类名、代码行数（可为 `null` ，视调用的方法），文件名（可为 `null` ，视调用的方法）。虽然 `StackTraceElement` 提供了 `public` 构造器，但该类对象一般还是通过JVM构造的（通过调用 `Throwable` 实例的 `getStackTraceElement(int index)` 本地方法设置到 `stackTrace` 成员变量）。

 我们可以调用 `getStackTrace` 方法获取到 `stackTrace` 数组的副本，里面包含了栈区方法的信息。

 
```
public StackTraceElement[] getStackTrace() {
	return getOurStackTrace().clone();
}

//printStackTrace方法, writeObject(用于序列化)也会调用此方法获取栈信息
private synchronized StackTraceElement[] getOurStackTrace() {
    if (stackTrace == UNASSIGNED_STACK || (stackTrace == null && backtrace != null)) {
		int depth = getStackTraceDepth();
        stackTrace = new StackTraceElement[depth];
        for (int i=0; i < depth; i++)
			stackTrace[i] = getStackTraceElement(i);
    } else if (stackTrace == null) {
        return UNASSIGNED_STACK;
    }
	return stackTrace;
}

native int getStackTraceDepth();
native StackTraceElement getStackTraceElement(int index);

```
 栈信息、栈深度都是通过本地方法获取的，这里就不再讨论JVM相关的实现了。

 （5） `List<Throwable> suppressedExceptions`  : 这个是JDK 1.7引入的新特性。该 `List` 用来保存被屏蔽的异常对象，在try-catch语句中，如果 `try` 中抛出了异常，在执行流程转移到方法栈上一层之前， `finally` 语句块会执行，但是，如果在 `finally` 语句块中又抛出了一个异常，那么这个异常会覆盖掉之前抛出的异常，这点很像 `finally` 中 `return` 的覆盖。比如下面这个例子：

 
```
public class Test {
	static class MyException extends Exception {
		public MyException(String string, Exception e) {
			super(string, e);
		}
	}
	private static void test() throws MyException {
		try {
			Integer.valueOf("oh");
		} catch(NumberFormatException e) {
			throw new MyException("One", e); //在JDK1.7以前，这个异常会被覆盖掉
		} finally {
			try {
				Integer.valueOf("oh");
			} catch(NumberFormatException e) {
				throw new MyException("Two", e); //finally语句抛出异常
			}
		}
	}
	public static void main(String[] args) {
		try {
			test();
		} catch (MyException e) {
			e.printStackTrace();
		}
	}
}

```
 运行上述代码，控制台输出：

 
```
com.test.Test$MyException: Two
	at com.test.Test.test(Test.java:42)
	at com.test.Test.main(Test.java:49)
Caused by: java.lang.NumberFormatException: For input string: "oh"
	at java.lang.NumberFormatException.forInputString(NumberFormatException.java:65)
	at java.lang.Integer.parseInt(Integer.java:580)
	at java.lang.Integer.valueOf(Integer.java:766)
	at com.test.Test.test(Test.java:40)
	... 1 more

```
 发现没有，之前 `MyException("One", e)` 被第二个 `MyException` 覆盖掉了。  
 在JDK1.7以前，有两种解决方案：如果只关心第一个异常，那么我们可以再 `try` 语句块之前用一个局部变量保存第一次抛出的 `Exception` ，在 `finally` 语句块捕获异常的时候，判断这个局部变量是否为 `null` ，如果不为 `null` 则抛出局部变量保存的异常对象，否则直接抛出第二个异常对象。另外一种是继承 `Exception` 类并添加一个 `List` 或其它容器保存已经抛出的异常对象，然后在上一层方法中通过循环依次处理。  
 在JDK1.7以后， `Throwable` 对象提供了 `addSupperssed` 和 `getSupperssed` 方法，允许把 `finally` 语句块中产生的异常通过 `addSupperssed` 方法添加到try语句产生的异常中。

 
```
public final synchronized void addSuppressed(Throwable exception) {
	if (exception == this)
		throw new IllegalArgumentException(SELF_SUPPRESSION_MESSAGE, exception);
    if (exception == null)
		throw new NullPointerException(NULL_CAUSE_MESSAGE);
    if (suppressedExceptions == null) // Suppressed exceptions not recorded
        return;
    if (suppressedExceptions == SUPPRESSED_SENTINEL)
        suppressedExceptions = new ArrayList<>(1);
    suppressedExceptions.add(exception);
}

```
 `addSuppressed` 方法不能传入自身，也不可为空。如果没有设置过 `suppressedExceptions` （默认为 `SUPPRESSED_SENTINEL` ），则会给 `suppressedExceptions` 构造一个 `ArrayList` ，并将异常添加进去。

 
```
public final synchronized Throwable[] getSuppressed() {
    if (suppressedExceptions == SUPPRESSED_SENTINEL || suppressedExceptions == null)
        return EMPTY_THROWABLE_ARRAY;
    else
        return suppressedExceptions.toArray(EMPTY_THROWABLE_ARRAY);
}

```
 `getSuppressed` 方法会首先判断有没有添加过 `suppressed` ，如果没有，则返回一个空数组，否则将 `suppressedExceptions` 成员变量的值拷贝成一个 `Throwable[]` 数组然后返回。

 为了更好的理解这个功能的作用，我们将上述例子改写为：

 
```
private static void test() throws MyException {
	MyException ex = null;
	try {
		Integer.valueOf("oh");
	} catch(NumberFormatException e) {
		ex = new MyException("One", e);
		throw ex;
	} finally {
		try {
			Integer.valueOf("oh");
		} catch(NumberFormatException e) {
			MyException ex2 = new MyException("Two", e);
			if(ex != null) {
				ex.addSuppressed(ex2);
				throw ex;
			}
			throw ex2;
		}
	}
}	
public static void main(String[] args) {
	try {
		test();
	} catch (MyException e) {
		e.printStackTrace();
	}
}

```
 这样， `main` 方法获取到的异常对象 `e` 就包含了 `test` 方法中两个异常对象，输出结果为：

 
```
com.test.Test$MyException: One
	at com.test.Test.test(Test.java:38)
	at com.test.Test.main(Test.java:56)
	Suppressed: com.test.Test$MyException: Two
		at com.test.Test.test(Test.java:44)
		... 1 more
	Caused by: java.lang.NumberFormatException: For input string: "oh"
		at java.lang.NumberFormatException.forInputString(NumberFormatException.java:65)
		at java.lang.Integer.parseInt(Integer.java:580)
		at java.lang.Integer.valueOf(Integer.java:766)
		at com.test.Test.test(Test.java:42)
		... 1 more
Caused by: java.lang.NumberFormatException: For input string: "oh"
	at java.lang.NumberFormatException.forInputString(NumberFormatException.java:65)
	at java.lang.Integer.parseInt(Integer.java:580)
	at java.lang.Integer.valueOf(Integer.java:766)
	at com.test.Test.test(Test.java:36)
	... 1 more

```
 `addSuppressed` 方法添加的异常对象在打印堆栈信息时会加上："Suppressed:"前缀。

 **静态变量：**  
 **（1） `StackTraceElement SentinelHolder.STACK_TRACE_ELEMENT_SENTINEL` **  
 **和  `StackTraceElement[] SentinelHolder.STACK_TRACE_SENTINEL` **  
 这两个变量主要用于序列化，当序列化出的 `stackTrace` 的长度为1时且这个唯一的元素等价于 `STACK_TRACE_ELEMENT_SENTINEL` 时，将 `stackTrace` 成员变量设为 `null` ，这里不详细讨论了  
 **（2） `StackTraceElement[] UNASSIGNED_STACK` **  
 一个空的 `StackTraceElement[]` 数组，用来初始化或者作为返回值。  
 **（3） `List<Throwable> SUPPRESSED_SENTINEL` **  
 一个空的只读List，同样用于初始化  
 **（4） `String NULL_CAUSE_MESSAGE` 、 `SELF_SUPPRESSION_MESSAGE` 、 `CAUSE_CAPTION` 、 `SUPPRESSED_CAPTION` **  
 前两个用于作为错误信息，后两个作为printStackTrace方法的说明前缀使用  
 **（5） `Throwable[] EMPTY_THROWABLE_ARRAY` **  
 用作 `getSuppressed` 方法的返回值（当 `suppressedExceptions` 没有元素时）

 
#### []()构造方法分析：

 分析完这些变量后，相信大家已经对 `Throwable` 类已经有了一个初步的了解，下面我们再来分析构造方法。

 
```
public Throwable() {
	fillInStackTrace();
}
/**
 * @param message 异常描述信息，该参数直接赋值给实例变量detailMessage
 */
public Throwable(String message) {
    fillInStackTrace();
	detailMessage = message;
}
/**
 * @param message 异常描述信息，该参数直接赋值给实例变量detailMessage
 * @param cause 描述当前异常由哪个异常引发
 */
public Throwable(String message, Throwable cause) {
    fillInStackTrace();
    detailMessage = message;
	this.cause = cause;
}
/**
 * @param cause 描述当前异常由哪个异常引发
 */
public Throwable(Throwable cause) {
	fillInStackTrace();
	detailMessage = (cause==null ? null : cause.toString());
	this.cause = cause;
}
/**
 * @param message 异常描述信息，该参数直接赋值给实例变量detailMessage
 * @param cause 描述当前异常由哪个异常引发
 * @param enableSuppression 是否支持Suppress异常消息
 * @param writableStackTrace 是否调用fillInStackTrace使堆栈信息可以写入
 */
protected Throwable(String message, Throwable cause,
                        boolean enableSuppression, boolean writableStackTrace) {
    if (writableStackTrace) {
        fillInStackTrace();
    } else {
	    stackTrace = null;
    }
    detailMessage = message;
    this.cause = cause;
	if (!enableSuppression)
		suppressedExceptions = null;
}

```
 `Throwable` 提供了4个 `public` 构造器和1个 `protected` 构造器（该构造器由JDK1.7引入）。4个 `public` 构造器共同点就是都调用了 `fillInStackTrace` 方法。

 
```
public synchronized Throwable fillInStackTrace() {
    if (stackTrace != null || backtrace != null) {
        fillInStackTrace(0);
		stackTrace = UNASSIGNED_STACK;
    }
	return this;
}
private native Throwable fillInStackTrace(int dummy);

```
 `fillInStackTrace` 会首先判断 `stackTrace` 是不是为 `null` ，如果不为 `null` 则会调用 `native` 方法 `fillInStackTrace` 获取当前堆栈信息。那么什么时候为 `null` 呢，答案是上面的 `protected` 构造器可以指定 `writableStackTrace` 为 `false` ，这样 `stackTrace` 就为 `null` 了，就不会调用 `fillInStackTrace` 获取堆栈信息。如果你不需要异常的栈信息，你也可以重写这个方法，让它直接返回 `this` ，毕竟异常的爬栈是一个开销比较大的操作。

 下面给出一个 `fillInStackTrace` 使用的例子：

 
```
private static void throwException() throws MyException {
	throw new MyException();
}
	
private static void test() throws Throwable {
	try {
		throwException();
	} catch(MyException e) {
		e.printStackTrace();
		throw e.fillInStackTrace();
	}
}
	
public static void main(String[] args) {
	try {
		test();
	} catch (Throwable e) {
		e.printStackTrace();
	}
}

```
 控制台输出：

 
```
com.test.Test$MyException
	at com.test.Test.throwException(Test.java:43)
	at com.test.Test.test(Test.java:48)
	at com.test.Test.main(Test.java:57)
com.test.Test$MyException
	at com.test.Test.test(Test.java:51)
	at com.test.Test.main(Test.java:57)

```
 
#### []()printStackTrace方法

 很多Java新手在处理异常的时候就是直接调用此方法打印异常堆栈信息，这个方法在调试过程中很有用，但是在生产环境中就不太适用了。  
 下面我们来分析源码：

 
```
public void printStackTrace() {
	printStackTrace(System.err);
}

public void printStackTrace(PrintStream s) {
	printStackTrace(new WrappedPrintStream(s));
}

public void printStackTrace(PrintWriter s) {
	printStackTrace(new WrappedPrintWriter(s));
}

private void printStackTrace(PrintStreamOrWriter s) {
    Set<Throwable> dejaVu = Collections.newSetFromMap(new IdentityHashMap<Throwable, Boolean>());
    dejaVu.add(this);

    synchronized (s.lock()) {
        s.println(this);
        StackTraceElement[] trace = getOurStackTrace();
        for (StackTraceElement traceElement : trace)
                s.println("\tat " + traceElement);

        for (Throwable se : getSuppressed())
            se.printEnclosedStackTrace(s, trace, SUPPRESSED_CAPTION, "\t", dejaVu);
                
        Throwable ourCause = getCause();
		if (ourCause != null)
	        ourCause.printEnclosedStackTrace(s, trace, CAUSE_CAPTION, "", dejaVu);
	}
}
}

```
 `printStackTrace` 把传入的输入流用内部类 `WrappedPrintStream` 或 `WrappedPrintWriter` 包装，主要用来实现 `printStackTrace` 方法在打印堆栈信息时的线程安全。

   
  