---
title: java.lang.System类源码解析（JDK 1.8）
date: 2018-08-26 18:32:27
tags: CSDN迁移
---
 版权声明：尊重原创，转载请注明出处 [ https://blog.csdn.net/abc123lzf/article/details/82080572]( https://blog.csdn.net/abc123lzf/article/details/82080572)   
  ### 一、概述

 java.lang.System类是JDK提供的一个工具类，包含了很多常用的成员变量和方法，它的无参构造器被private修饰，所以不能被实例化。其主要功能有：   
 1、提供三种标准输入输出流：err、out、in，并且可以重定向输入输出流

 
```
public final static InputStream in = null;
public final static PrintStream out = null;
public final static PrintStream err = null;
public static void setIn(InputStream in);
public static void setOut(PrintStream out);
public static void setErr(PrintStream err);
```
 2、提供访问系统相关的参数、环境变量的方法（只列出主要方法）

 
```
public static Properties getProperties();
public static Properties setProperties(Properties props);
public static java.util.Map<String,String> getenv();
public static String lineSeparator();
```
 3、保存安全管理器：SecurityManager

 
```
public static void setSecurityManager(final SecurityManager s);
public static SecurityManager getSecurityManager();
```
 4、加载文件、类库的方法

 
```
public static void load(String filename);
public static void loadLibrary(String libname);
public static native String mapLibraryName(String libname);
```
 5、一些操作JVM的方法

 
```
public static void gc();
public static void exit(int status); 
public static void runFinalization();
```
 6、一些工具方法

 
```
public static native void arraycopy(Object src, int srcPos,
        Object dest, int destPos, int length);
public static native int identityHashCode(Object x);
public static native long currentTimeMillis();
public static native long nanoTime();
```
 下面我们分点来讲解上述功能

 
### 二、标准输入输出流

 
```
public final static InputStream in = null;
public final static PrintStream out = null;
public final static PrintStream err = null;
```
 System类提供了标准输入流（成员变量in）、标准输出流（成员变量out）和标准错误流（成员变量err），它们的访问权限是public，可以直接通过System类名访问。   
 虽然看上去这些成员变量被final修饰且初始化过程中全部为空引用，但是在实际运行期间这些输入流会自动被JVM定向到键盘输入流，输出流和错误流定向到控制台输出。这是为什么呢？

 
```
/* register the natives via the static initializer.
     *
     * VM will invoke the initializeSystemClass method to complete
     * the initialization for this class separated from clinit.
     * Note that to use properties set by the VM, see the constraints
     * described in the initializeSystemClass method.
     */
private static native void registerNatives();
static {
    registerNatives();
}
```
 原来，在System类的初始化块中，调用了registerNatives本地方法来完成System类的初始化，registerNatives会调用System类的私有方法initializeSystemClass来完成类的初始化，下面来看initializeSystemClass方法的代码：

 
```
private static void initializeSystemClass() {
    props = new Properties();
    initProperties(props);  // initialized by the VM

    sun.misc.VM.saveAndRemoveProperties(props);

    lineSeparator = props.getProperty("line.separator");
    sun.misc.Version.init();

    FileInputStream fdIn = new FileInputStream(FileDescriptor.in);
    FileOutputStream fdOut = new FileOutputStream(FileDescriptor.out);
    FileOutputStream fdErr = new FileOutputStream(FileDescriptor.err);
    setIn0(new BufferedInputStream(fdIn));
    //sun.stdout.encoding为字符编码方式
    setOut0(newPrintStream(fdOut, props.getProperty("sun.stdout.encoding")));
    setErr0(newPrintStream(fdErr, props.getProperty("sun.stderr.encoding")));

    loadLibrary("zip");
    Terminator.setup();
    sun.misc.VM.initializeOSEnvironment();

    Thread current = Thread.currentThread();
    current.getThreadGroup().add(current);

    setJavaLangAccess();
    sun.misc.VM.booted();
}
```
 可以看出，System.in/out/err实际上是FileDescriptor.in，FileDescriptor.out和FileDescriptor.err。查看FileDescriptor类的部分源码：

 
```
 public final class FileDescriptor {
    public static final FileDescriptor in = standardStream(0);
    public static final FileDescriptor out = standardStream(1);
    public static final FileDescriptor err = standardStream(2);

    private static FileDescriptor standardStream(int fd) {
        FileDescriptor desc = new FileDescriptor();
        desc.handle = set(fd);
        return desc;
    }
    private static native long set(int d);
    //省略其它代码
 }
```
 FileDescriptor类表示的是文件描述符。   
 下面文件描述符的定义转载自百度百科：   
 [https://baike.baidu.com/item/%E6%96%87%E4%BB%B6%E6%8F%8F%E8%BF%B0%E7%AC%A6/9809582](https://baike.baidu.com/item/%E6%96%87%E4%BB%B6%E6%8F%8F%E8%BF%B0%E7%AC%A6/9809582)   
 文件描述符在形式上是一个非负整数。实际上，它是一个索引值，指向内核为每一个进程所维护的该进程打开文件的记录表。当程序打开一个现有文件或者创建一个新文件时，内核向进程返回一个文件描述符。在程序设计中，一些涉及底层的程序编写往往会围绕着文件描述符展开。但是文件描述符这一概念往往只适用于UNIX、Linux这样的操作系统。   
 习惯上，标准输入（standard input）的文件描述符是 0，标准输出（standard output）是 1，标准错误（standard error）是 2。尽管这种习惯并非Unix内核的特性，但是因为一些 shell 和很多应用程序都使用这种习惯，因此，如果内核不遵循这种习惯的话，很多应用程序将不能使用。   
 POSIX 定义了 STDIN_FILENO、STDOUT_FILENO 和 STDERR_FILENO 来代替 0、1、2。这三个符号常量的定义位于头文件 unistd.h。   
 文件描述符的有效范围是 0 到 OPEN_MAX。一般来说，每个进程最多可以打开 64 个文件（0 — 63）。对于 FreeBSD 5.2.1、Mac OS X 10.3 和 Solaris 9 来说，每个进程最多可以打开文件的多少取决于系统内存的大小，int 的大小，以及系统管理员设定的限制。Linux 2.4.22 强制规定最多不能超过 1,048,576 。   
 文件描述符是由无符号整数表示的句柄，进程使用它来标识打开的文件。文件描述符与包括相关信息（如文件的打开模式、文件的位置类型、文件的初始类型等）的文件对象相关联，这些信息被称作文件的上下文。

 System类提供了重定向这些标准流的方法：

 
```
public static void setIn(InputStream in) {
    checkIO();
    setIn0(in);
}
public static void setOut(PrintStream out) {
    checkIO();
    setOut0(out);
}
public static void setErr(PrintStream err) {
    checkIO();
    setErr0(err);
}
private static void checkIO() {
    SecurityManager sm = getSecurityManager();
    if (sm != null) {
        sm.checkPermission(new RuntimePermission("setIO"));
    }
}
private static native void setIn0(InputStream in);
private static native void setOut0(PrintStream out);
private static native void setErr0(PrintStream err);
```
 即使in、out、err这些成员变量是final修饰的，也可以通过本地方法进行修改。在重定向输入输出流之前，会调用checkIO方法，该方法获取安全管理器查看用户是否有权限重定向这些流，如果没有权限则会抛出SecurityException异常。

 
### 三、系统参数、环境变量、用户信息的访问

 System类提供了多个方法来访问这些参数，这些参数一般通过参数名、参数值方式存储

 
##### (1)获取JVM相关参数

 System提供一下方法访问JVM相关参数：

 
```
private static Properties props;
//获取成员变量props
public static Properties getProperties() {
    SecurityManager sm = getSecurityManager();
    if (sm != null) {
        sm.checkPropertiesAccess();
    }
    return props;
}
//设置自己的props
public static void setProperties(Properties props) {
    SecurityManager sm = getSecurityManager();
    if (sm != null) {
        sm.checkPropertiesAccess();
    }
    if (props == null) {
        props = new Properties();
        initProperties(props);
    }
        System.props = props;
}
//通过参数名访问参数值
public static String getProperty(String key) { /*...*/ }
//通过参数名访问参数值，如果没有找到该参数则返回def
public static String getProperty(String key, String def) { /*...*/ }
//添加参数
public static String setProperty(String key, String value){ /*...*/ }
//通过参数名清空该参数
public static String clearProperty(String key){ /*...*/ }
```
 在System类加载过程中，props会在先前提到的initializeSystemClass方法中由本地方法构造。props在构造完成后会持有和JVM和Java运行环境相关的参数：   
 java.version Java 运行时环境版本   
 java.vendor Java运行时环境供应商   
 java.vendor.url Java 供应商的 URL   
 java.home Java 安装目录   
 java.vm.specification.version Java 虚拟机规范版本   
 java.vm.specification.vendor Java 虚拟机规范供应商   
 java.vm.specification.name Java 虚拟机规范名称   
 java.vm.version Java 虚拟机实现版本   
 java.vm.vendor Java 虚拟机实现供应商   
 java.vm.name Java 虚拟机实现名称   
 java.specification.version Java 运行时环境规范版本   
 java.specification.vendor Java 运行时环境规范供应商   
 java.specification.name Java 运行时环境规范名称   
 java.class.version Java 类格式版本号   
 java.class.path Java 类路径   
 java.library.path 加载库时搜索的路径列表   
 java.io.tmpdir 默认的临时文件路径   
 java.compiler 要使用的 JIT 编译器的名称   
 java.ext.dirs 一个或多个扩展目录的路径   
 os.name 操作系统的名称   
 os.arch 操作系统的架构   
 os.version 操作系统的版本   
 file.separator 文件分隔符   
 path.separator 路径分隔符   
 line.separator 行分隔符   
 user.name 用户的账户名称   
 user.home 用户的主目录   
 user.dir 用户的当前工作目录

 在获取或修改这些参数前，同样会通过安全管理器检查用户是否有权限访问或修改这些参数。

 
##### （2）获取环境变量

 System类提供了获取环境变量的方法：

 
```
public static String getenv(String name) {
    SecurityManager sm = getSecurityManager();
    if (sm != null) {
        sm.checkPermission(new RuntimePermission("getenv."+name));
    }
    return ProcessEnvironment.getenv(name);
}

public static java.util.Map<String,String> getenv() {
    SecurityManager sm = getSecurityManager();
    if (sm != null) {
        sm.checkPermission(new RuntimePermission("getenv.*"));
    }
    return ProcessEnvironment.getenv();
}
```
 这些参数保存在ProcessEnvironment类中，ProcessEnvironment类继承了HashMap类，提供静态方法来访问该类的实例

 
```
private static final NameComparator nameComparator;
private static final EntryComparator entryComparator;
private static final ProcessEnvironment theEnvironment;
private static final Map<String,String> theUnmodifiableEnvironment;
private static final Map<String,String> theCaseInsensitiveEnvironment;

static {
    nameComparator  = new NameComparator();
    entryComparator = new EntryComparator();
    theEnvironment  = new ProcessEnvironment();
    theUnmodifiableEnvironment = Collections.unmodifiableMap(theEnvironment);
    //省略其它方法...
    theCaseInsensitiveEnvironment = new TreeMap<>(nameComparator);
    theCaseInsensitiveEnvironment.putAll(theEnvironment);
}

static Map<String,String> getenv() {
    return theUnmodifiableEnvironment;
}
```
 可以看出，调用getenv方法后返回的是theUnmodifiableEnvironment，该类是Collections.UnmodifiableMap的实例，这种Map是只读的，不可被修改，相当于给theEnvironment加了层保护，只允许JVM进行修改。

 
#### 四、安全管理器（SecurityManager）相关方法

 System类负责保存安全管理器，在构造安全管理器的时候会检查System类是否持有安全管理器，如果持有一个安全管理器实例再构造的话则会抛出异常。

 下面是System类安全管理器相关的方法和变量：

 
```
private static volatile SecurityManager security = null;

public static void setSecurityManager(final SecurityManager s) {
    try {
        s.checkPackageAccess("java.lang");
    } catch (Exception e) {
    }
    setSecurityManager0(s);
}

private static synchronized void setSecurityManager0(final SecurityManager s) {
    SecurityManager sm = getSecurityManager();
    if (sm != null) {
        sm.checkPermission(new RuntimePermission("setSecurityManager"));
    }

    if ((s != null) && (s.getClass().getClassLoader() != null)) {
        AccessController.doPrivileged(new PrivilegedAction<Object>() {
            public Object run() {
                s.getClass().getProtectionDomain().implies(SecurityConstants.ALL_PERMISSION);
                return null;
            }
        });
    }
    security = s;
}

public static SecurityManager getSecurityManager() {
    return security;
}
```
 在设置安全管理器前，会检查是否有java.lang包的访问权限，然后检查System类是否已经持有一个安全管理器实例，如果存在，则会检查用户是否有设置安全管理器的权限。下一步代码的作用为（谷歌翻译的注释）：新安全管理器的类不在引导类路径上。因为更新安全管理器之前要调用初始化策略，以便在尝试调用初始化策略时防止无限循环（通常涉及访问某些安全性和/或系统属性，这些属性会回调已安装的安全管理器的checkPermission方法，如果堆栈上存在非系统类（在本代码中为：新的安全管理器类），该方法将无限循环。   
 最后将成员变量security指向新的安全管理器，设置安全管理器步骤完成。

 
#### 五、加载文件、类库

 System类提供了加载库文件的方法，可以用来加载JNI库文件（用来实现本地方法）

 
```
@CallerSensitive
public static void load(String filename) {
    Runtime.getRuntime().load0(Reflection.getCallerClass(), filename);
}

@CallerSensitive
public static void loadLibrary(String libname) {
    Runtime.getRuntime().loadLibrary0(Reflection.getCallerClass(), libname);
}

public static native String mapLibraryName(String libname);
```
 load方法需要传入一个库文件的绝对路径，不能传入相对路径。比如：   
 System.load(“C:\demojni.dll”);   
 loadLibrary方法需要传入库文件名，不包括文件扩展名，比如：   
 System.load(“demojni”);   
 这些路径必须是JVM属性java.library.path所指向的路径，否则无法加载。   
 java.library.path路径可以通过System.getProperty(“java.library.path”)获得

 Windows环境下一般可以为   
 C:\Windows\system32   
 C:\Windows\Sun\Java\bin   
 %JAVA_HOME%\bin

 mapLibraryName将一个库名称映射到特定于平台的、表示本机库的字符串中。

 
### 六、操作JVM相关

 
##### （1）gc

 
```
public static void gc() {
    Runtime.getRuntime().gc();
}
```
 调用该方法会**通知** JVM进行垃圾回收(Full GC)，JVM并不保证调用该方法后马上进行垃圾回收。   
 gc方法为native方法，实现原理可以参考这篇博客：[https://www.cnblogs.com/xll1025/p/6517871.html](https://www.cnblogs.com/xll1025/p/6517871.html)

 
##### （2）exit

 
```
public static void exit(int status) {
    Runtime.getRuntime().exit(status);
}
```
 status为0时代表程序正常结束，非0代表程序异常退出。   
 该方法会强制结束当前程序。

 
##### （3）runFinalization

 
```
public static void runFinalization() {
    Runtime.getRuntime().runFinalization();
}
```
 该方法会尝试运行当前程序中即将被垃圾收集的对象的finalize方法。同样类似于gc方法，只是通知JVM执行，但并不保证该方法能够执行到位。

 
### 七、工具方法

 
##### （1）arraycopy

 
```
public static native void arraycopy(Object src, int srcPos, 
        Object dest, int destPos, int length);
```
 该方法能够通过本地方法以一种更快方式复制数组。   
 src为原数组，srcPos表示从原数组复制的起点(下标号)，dest为目标数组，destPos表示目标数组的写入起点，length表示需要复制的数组总长度。   
 关于源码实现可以参考这篇博客[https://blog.csdn.net/u011642663/article/details/49512643](https://blog.csdn.net/u011642663/article/details/49512643)

 
##### （2）identityHashCode

 
```
public static native int identityHashCode(Object x);
```
 该方法相当于调用Object对象的hashCode方法，hashCode方法默认实现为返回对象的内存地址。

 
##### （3）时间相关

 
```
public static native long currentTimeMillis();
public static native long nanoTime();
```
 currentTimeMillis()返回以毫秒为单位的当前时间，返回的是当前时间与协调世界时 1970 年 1 月 1 日午夜之间的时间差（以毫秒为单位测量）。值的粒度取决于基础操作系统，并且粒度可能更大。   
 nanoTime()返回以纳秒为单位的当前时间，返回的是当前时间与协调世界时 1970 年 1 月 1 日午夜之间的时间差（以纳秒为单位测量）。值的粒度取决于基础操作系统，并且粒度可能更大。

   
  