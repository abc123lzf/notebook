---
title: 深入理解Java序列化机制：ObjectOutputStream源码简要分析
date: 2018-09-03 22:00:01
tags: CSDN迁移
---
 版权声明：尊重原创，转载请注明出处 [ https://blog.csdn.net/abc123lzf/article/details/82318148]( https://blog.csdn.net/abc123lzf/article/details/82318148)   
  这篇文章是博主阅读源码之后根据自己的理解写出来的，由于网上ObjectOutputStream源码分析文章很少且大多并不详细，所以只分析了一小部分，可能会有错误或描述的不到位的地方，欢迎指出。

 
### 一、引言

 java.io.ObjectOutputStream是实现序列化的关键类，它可以将一个对象转换成二进制流，然后可以通过ObjectInputStream将二进制流还原成对象。

 在阅读ObjectOutputStream源码之前，我们先来回顾一下序列化相关的基础知识：   
 1、需要序列化的类必须实现java.io.Serializable接口，否则会抛出NotSerializableException异常   
 2、如果检测到反序列化后的类的serialVersionUID和对象二进制流的serialVersionUID不同，则会抛出   
 异常。   
 3、Java的序列化会将一个类包含的引用中所有的成员变量保存下来（深度复制），所以里面的引用类型必须也要实现java.io.Serializable接口。   
 4、对于不采用默认序列化或无须序列化的成员变量，可以添加transient关键字，并不是说添加了transient关键字就一定不能序列化。   
 5、每个类可以实现readObject、writeObject方法实现自己的序列化策略，即使是transient修饰的成员变量也可以手动调用ObjectOutputStream的writeInt等方法将这个成员变量序列化。

 
### 二、使用方法

 我们先来回顾一下ObjectOutputStream的使用方法：

 
```
class Person implements Serializable {
    private static final long serialVersionUID = 1386583756403881124L;
    String name;
    int age;
}

public class Test {
    public static void main(String[] args) throws IOException {
        FileOutputStream fos = new FileOutputStream("D:\\out.txt");
        ObjectOutputStream oos = new ObjectOutputStream(fos);
        Person p = new Person();
        p.name = "LZF";
        p.age = 19;
        oos.writeObject(p);
        oos.close();
    }
}
```
 ObjectOutputStream只有一个public权限的构造方法：该构造方法需要传入一个OutputStream表示将对象二进制流写入到指定的OutputStream。   
 查看out.txt：输出数据如下：   
 ![这里写图片描述](https://img-blog.csdn.net/20180903105709160?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FiYzEyM2x6Zg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

 
### 三、源码简要分析

 ObjectOutputStream的实现很复杂，建议读者们先对ObjectOutputStream源码的主要方法先过一遍再往下看。   
 ObjectOutputStream类定义：

 
```
public class ObjectOutputStream extends OutputStream 
        implements ObjectOutput, ObjectStreamConstants {
    //...
}
```
 ObjectOutputStream继承了OutputStream类，实现了ObjectOutput接口和ObjectStreamConstants接口   
 ObjectStreamConstants接口并没有定义方法，其内部定义了很多byte类型常量，表示序列化后的单个字节数据的含义。

 了解完这些成员变量后，我们从几个最常用的序列化操作为切入点分析：ObjectOutputStream的构造方法和writeObject方法。   
 **ObjectOutputStream的构造方法：**

 
```
public ObjectOutputStream(OutputStream out) throws IOException {
    //检查继承权限
    verifySubclass();
    //构造一个BlockDataOutputStream用于向out写入序列化数据
    bout = new BlockDataOutputStream(out);
    //构造一个大小为10，负载因子为3的HandleTable和ReplaceTable
    handles = new HandleTable(10, (float) 3.00);
    subs = new ReplaceTable(10, (float) 3.00);
    //恒为false，除非子类调用protected构造方法
    enableOverride = false;
    writeStreamHeader();
    //将缓存模式打开，写入数据时先写入缓冲区
    bout.setBlockDataMode(true);
    if (extendedDebugInfo) {
        debugInfoStack = new DebugTraceInfoStack();
    } else {
        debugInfoStack = null;
    }
}
```
 BlockDataOutputStream是ObjectOutputStream的内部类，它将构造ObjectOutputStream传入的OutputStream实例包装起来，当外部类ObjectOutputStream需要向这个OutputStream写入序列化数据时，就由这个类来完成实际的写入操作。

 构造方法首先调用verifySubclass方法分析现在构造的是不是ObjectOutputStream的子类，即：

 
```
private void verifySubclass() {
    Class<?> cl = getClass();
    //如果构造的不是ObjectOutputStream的子类则直接返回
    if (cl == ObjectOutputStream.class)
        return;
    //否则获取安全管理器检查是否有继承ObjectOutputStream的权限
    SecurityManager sm = System.getSecurityManager();
    if (sm == null)
        return;
    //移除Caches中已经失去引用的Class对象
    processQueue(Caches.subclassAuditsQueue, Caches.subclassAudits);
    //将ObjectOutputStream的子类存入Caches
    WeakClassKey key = new WeakClassKey(cl, Caches.subclassAuditsQueue);

    Boolean result = Caches.subclassAudits.get(key);
    if (result == null) {
        result = Boolean.valueOf(auditSubclass(cl));
        Caches.subclassAudits.putIfAbsent(key, result);
    }
    if (result.booleanValue())
        return;
    //如果没有权限则抛出SecurityException异常
    sm.checkPermission(SUBCLASS_IMPLEMENTATION_PERMISSION);
}
```
 该方法如果识别到构造的是ObjectOutputStream的子类，则会检查是否拥有SUBCLASS_IMPLEMENTATION_PERMISSION权限，否则抛出SecurityException异常。   
 另外，ObjectOutputStream通过一个Cache静态内部类中的ConcurrentHashMap来缓存ObjectOutputStream子类信的息。Class类通过内部类WeakClassKey（继承WeakReference，将一个弱引用指向一个Class对象）存储。

 
```
private static class Caches {
    static final ConcurrentMap<WeakClassKey,Boolean> subclassAudits = new ConcurrentHashMap<>();
    static final ReferenceQueue<Class<?>> subclassAuditsQueue = new ReferenceQueue<>();
}
```
 在进行完ObjectOutputStream的类型检查后，构造方法会随之构建一个BlockDataOutputStream用于向传入的OutputStream写入对象信息，并构建长度为10，负载因子为3的HandleTable和ReplaceTable。随后，将魔数(0xACED)和版本标识符(0x0005)写入文件头，用来检测是不是一个序列化对象。

 
```
protected void writeStreamHeader() throws IOException {
    bout.writeShort(STREAM_MAGIC); //写入两个字节：0xAC和0xED
    bout.writeShort(STREAM_VERSION); //写入两个字节:0x00和0x05
}
```
 最后根据sun.io.serialization.extendedDebugInfo配置信息决定是否启用调式信息栈。

 
```
private static final boolean extendedDebugInfo =
        java.security.AccessController.doPrivileged(
            new sun.security.action.GetBooleanAction(
                "sun.io.serialization.extendedDebugInfo")).booleanValue();
```
 如果extendedDebugInfo为true，则构造方法会构造一个DebugTraceInfoStack，否则置为null。

 构造完ObjectOutputStream对象后，我们一般会随之调用writeObject(Object)方法将对象写入

 
```
public final void writeObject(Object obj) throws IOException {
    //在ObjectOutputStream中这个变量恒为false，只有子类为true
    if (enableOverride) {
        //实现为空，供子类重写用
        writeObjectOverride(obj);
        return;
    }
    try {
        writeObject0(obj, false);
    } catch (IOException ex) {
        if (depth == 0)
            writeFatalException(ex);
        throw ex;
    }
}
```
 writeObject方法首先会检查是否是ObjectOutputStream的子类，如果是则调用writeObjectOverride方法，这个方法默认实现为空，需要子类根据实际业务需求定制序列化方法。   
 随后调用writeObject0方法

 
```
private void writeObject0(Object obj, boolean unshared) throws IOException {
    //关闭缓冲模式，直接向目标OutputStream写入数据
    boolean oldMode = bout.setBlockDataMode(false);
    depth++;
    try {
        int h;
        //处理以前写过的和不可替换的对象
        //如果obj为null(只有当obj为null时才会返回null)
        if ((obj = subs.lookup(obj)) == null) {
            writeNull();
            return;
        } else if (!unshared && (h = handles.lookup(obj)) != -1) {
            writeHandle(h);
            return;
        } else if (obj instanceof Class) {
            writeClass((Class) obj, unshared);
            return;
        } else if (obj instanceof ObjectStreamClass) {
            writeClassDesc((ObjectStreamClass) obj, unshared);
            return;
        }

        Object orig = obj;
        Class<?> cl = obj.getClass();
        //序列化对象对应的Class对象的详细信息
        ObjectStreamClass desc;
        for (;;) {
            Class<?> repCl;
            //获取序列化对象对应的Class对象详细信息，待会会讨论ObjectStreamClass
            desc = ObjectStreamClass.lookup(cl, true);
            //直接break，因为最后(repCl=obj.getClass())==null恒等于true(我也不知道为什么这里要有for循环)
            if (!desc.hasWriteReplaceMethod() ||
                    (obj = desc.invokeWriteReplace(obj)) == null ||
                    (repCl = obj.getClass()) == cl)
                    break;
                cl = repCl;
        }
        if (enableReplace) {
            //replaceObject用来替换这个对象进行序列化，默认实现为空，一般用于子类重写实现序列化的定制
            Object rep = replaceObject(obj);
            //如果对象被替换了
            if (rep != obj && rep != null) {
                cl = rep.getClass();
                //重新查找对应的ObjectStreamClass
                desc = ObjectStreamClass.lookup(cl, true);
            }
            obj = rep;
        }

        //如果对象被替换了(非ObjectOutputStream子类不会发生)
        if (obj != orig) {
            subs.assign(orig, obj);
            if (obj == null) {
                writeNull();
                return;
            } else if (!unshared && (h = handles.lookup(obj)) != -1) {
                writeHandle(h);
                return;
            } else if (obj instanceof Class) {
                writeClass((Class) obj, unshared);
                return;
            } else if (obj instanceof ObjectStreamClass) {
                writeClassDesc((ObjectStreamClass) obj, unshared);
                return;
            }
        }

        //序列化对象类型为String、数组、枚举时，调用定制的写入方法
        if (obj instanceof String) {
            writeString((String) obj, unshared);
        } else if (cl.isArray()) {
            writeArray(obj, desc, unshared);
        } else if (obj instanceof Enum) {
            writeEnum((Enum<?>) obj, desc, unshared);
        //一般对象类型的写入(当然需要实现序列化接口)
        } else if (obj instanceof Serializable) {
            writeOrdinaryObject(obj, desc, unshared);
        //如果没有实现序列化接口会抛出异常
        } else {
            if (extendedDebugInfo)
                throw new NotSerializableException(cl.getName() + "\n" + debugInfoStack.toString());
            else
                throw new NotSerializableException(cl.getName());
        }
    } finally {
        //结束方法前将方法栈深减去1
        depth--;
        bout.setBlockDataMode(oldMode);
    }
}
```
 ObjectStreamClass存储了一个Class对象的信息，其实例变量包括：Class对象，Class名称，serialVersionUID，实现了Serializable接口还是 Externalizable接口，非transient修饰的变量，自定义的writeObject和readObject的Method对象。

 下面来看ObjectStreamClass的lookup方法：

 
```
static ObjectStreamClass lookup(Class<?> cl, boolean all) {
    //如果all为false且cl并没有实现Serializable接口则直接返回null
    if (!(all || Serializable.class.isAssignableFrom(cl))) {
        return null;
    }
    //清除失去Class引用的ObjectStreamClass缓存
    //(缓存的用途是避免反复对同一个Class创建ObjectStreamClass对象)
    processQueue(Caches.localDescsQueue, Caches.localDescs);
    //创建一个临时的WeakClassKey用于从缓存中查找对应的ObjectStreamClass或EntryFuture
    WeakClassKey key = new WeakClassKey(cl, Caches.localDescsQueue);
    //获取保存有ObjectStreamClass或EntryFuture的引用
    Reference<?> ref = Caches.localDescs.get(key);
    Object entry = null;
    //如果引用不为null则直接获取其中的对象给entry
    if (ref != null) {
        entry = ref.get();
    }
    EntryFuture future = null;
    //如果引用的对象被GC
    if (entry == null) {
        //创建一个EntryFuture对象并将软引用newRef指向它
        EntryFuture newEntry = new EntryFuture();
        Reference<?> newRef = new SoftReference<>(newEntry);
        do {
            //从缓存中删除这个失去引用的键值对
            if (ref != null)
                Caches.localDescs.remove(key, ref);
            //将被newRef引用的EntryFuture添加到缓存（这里使用putIfAbsent而不是put可能是为了防止有其它线程已经添加了）
            ref = Caches.localDescs.putIfAbsent(key, newRef);
            if (ref != null)
                entry = ref.get();
        //循环直到ref为null或entry不为null
        } while (ref != null && entry == null);
        //如果entry为null
        if (entry == null)
            future = newEntry;
    }
    //如果从缓存中拿到了ObjectStreamClass
    if (entry instanceof ObjectStreamClass) {
        return (ObjectStreamClass) entry;
    }
    //如果从缓存中得到了EntryFuture
    if (entry instanceof EntryFuture) {
        future = (EntryFuture) entry;
        //如果创建这个EntryFuture的线程就是当前线程，即这个EntryFuture
        //是在前面的代码ref = Caches.localDescs.putIfAbsent(key, newRef);语句中设置的
        if (future.getOwner() == Thread.currentThread()) {
            entry = null;
        } else {
            entry = future.get();
        }
    }
    //如果entry为null那么就创建一个新的ObjectStreamClass对象并加入缓存
    if (entry == null) {
        try {
            entry = new ObjectStreamClass(cl);
        } catch (Throwable th) {
            entry = th;
        }
        //设置这个ObjectStreamClass实例
        if (future.set(entry)) {
            Caches.localDescs.put(key, new SoftReference<Object>(entry));
        } else {
            entry = future.get();
        }
    }
    //最后如果entry为ObjectOutputStream则直接返回，否则抛出异常
    if (entry instanceof ObjectStreamClass) {
        return (ObjectStreamClass) entry;
    } else if (entry instanceof RuntimeException) {
        throw (RuntimeException) entry;
    } else if (entry instanceof Error) {
        throw (Error) entry;
    } else {
        throw new InternalError("unexpected entry: " + entry);
    }
}
```
 在ObjectStreamClass类的内部类Caches中，存在一个类型为ConcurrentMap的静态成员变量localDescs：

 
```
static final ConcurrentMap<WeakClassKey,Reference<?>> localDescs = new ConcurrentHashMap<>();
private static final ReferenceQueue<Class<?>> localDescsQueue = new ReferenceQueue<>();
```
 ObjectStreamClass引入这个缓存主要是为了提高获取类信息的速度，如果反复对一个类的实例们进行序列化操作，那么只需要实例化一个ObjectStreamClass实例并导入这个缓存。   
 WeakClassKey继承WeakReference，将一个弱引用指向这个Class对象，当对应的ClassLoader失去引用时，不至于导致垃圾回收器无法回收这个Class对象。   
 引用队列localDescsQueue主要用于processQueue方法清除localDescs中无用的缓存。

 至于ObjectStreamClass的内部类EntryFuture的作用，我个人认为是为了实现多线程调用lookup方法而设立的。

 
```
private static class EntryFuture {
    private static final Object unset = new Object();
    private final Thread owner = Thread.currentThread();
    private Object entry = unset;

    //entry是ObjectStreamClass实例
    synchronized boolean set(Object entry) {
        if (this.entry != unset)
                return false;
        this.entry = entry;
        //entry已被设置，唤醒正在调用get方法的线程
        notifyAll();
        return true;
    }

    synchronized Object get() {
        boolean interrupted = false;
        while (entry == unset) {
            try {
                //等待到entry被set为止
                wait();
            } catch (InterruptedException ex) {
                interrupted = true;
            }
        }
        //如果被强制打断则返回null
        if (interrupted) {
            AccessController.doPrivileged(
                new PrivilegedAction<Void>() {
                    public Void run() {
                        Thread.currentThread().interrupt();
                        return null;
                    }
                });
        }
        //如果是正常被set方法唤醒的则直接返回设置好的ObjectStreamClass
        return entry;
    }
    //返回创建这个EntryFuture的线程
    Thread getOwner() {
        return owner;
    }
}
```
 现在回到writeObject0方法中   
 在获取到ObjectStreamClass对象后，会判断需要序列化的类是哪种类型。   
 下面我们就只分析writeOrdinaryObject方法：

 
```
private void writeOrdinaryObject(Object obj, ObjectStreamClass desc,
         boolean unshared) throws IOException {
    if (extendedDebugInfo)
        debugInfoStack.push(
                (depth == 1 ? "root " : "") + "object (class \"" +
                obj.getClass().getName() + "\", " + obj.toString() + ")");
    try {
        //检查ObjectStreamClass对象
        desc.checkSerialize();
        //写入字节0x73
        bout.writeByte(TC_OBJECT);
        //写入对应的Class对象的信息
        writeClassDesc(desc, false);
        handles.assign(unshared ? null : obj);
        if (desc.isExternalizable() && !desc.isProxy()) {
            writeExternalData((Externalizable) obj);
        } else {
            //写入这个对象变量信息及其父类的成员变量
            writeSerialData(obj, desc);
        }
    } finally {
        if (extendedDebugInfo) {
            debugInfoStack.pop();
        }
    }
}
```
 writeOrdinaryObject最终会以一种递归的形式写入对象信息。   
 writeSerialData方法会将这个实例及其父类基本数据类型写入文件，如果检测到有引用类型，那么会继续调用writeObject0方法写入，直到将这个对象包含的所有信息全部序列化为止。

 暂时就只能分析到这里了。

   
  