---
title: java.io包中的输入流、输出流相关类概览
date: 2018-09-02 17:20:21
tags: CSDN迁移
---
 版权声明：尊重原创，转载请注明出处 [ https://blog.csdn.net/abc123lzf/article/details/82290335]( https://blog.csdn.net/abc123lzf/article/details/82290335)   
  ### 一、引言

 java.io里面提供了许多I/O类，可以方便地实现数据的输入和输出   
 java.io包下主要包含以下几个类的类型：   
 1、字节输入流：如InputStream及其子类   
 2、字节输出流：如OutputStream及其子类   
 3、字符输入流：如Reader及其子类   
 4、字符输出流：如Writer及其子类   
 5、文件本地地址描述：File类   
 6、文件描述符：FileDescriptor类   
 7、IO异常类：IOException、FileNotFoundException等   
 8、文件系统类：FileSystem、WinNTFileSystem类（非public类）   
 9、Console类，可以读入控制台打印的信息也可输出到控制台。可由System.console获得

 这篇文章会从整体分析java.io包下的输入输出流，并会分析抽象类InputStream、OutputStream、Reader、Writer的源码。

 
--------
 
### 二、字节输入流：InputStream

 InputStream是所有字节输入流的抽象基类，它是字节输入流的模板，提供了一系列基本API来读入数据。   
 我们先来看java.io.InputStream的类定义及其方法：

 
```
public abstract class InputStream implements Closeable {
    private static final int MAX_SKIP_BUFFER_SIZE = 2048;
    //读入一个字节并返回这个字节
    public abstract int read() throws IOException;

    //将所有的字节读入到byte数组b中
    public int read(byte b[]) throws IOException {
        return read(b, 0, b.length);
    }

    //从数组b的off位置开始，读入len个字节到数组b，返回读到的字节数量(小于等于len)
    public int read(byte b[], int off, int len) throws IOException {
        if (b == null)
            throw new NullPointerException();
        else if (off < 0 || len < 0 || len > b.length - off)
            throw new IndexOutOfBoundsException();
        else if (len == 0)
            return 0;

        int c = read();
        //读到-1(EOF)说明已经读完
        if (c == -1)
            return -1;
        //从数组b的off开始读
        b[off] = (byte)c;
        int i = 1;
        try {
            for (; i < len ; i++) {
                c = read(); //调用抽象方法read()
                if (c == -1) //读到-1说明已经读完
                    break;
                b[off + i] = (byte)c;
            }
        } catch (IOException ee) { }
        return i;
    }
    //跳过n个字节，返回实际上跳过的字节数(小于等于n)
    public long skip(long n) throws IOException {
        long remaining = n;
        int nr;
        if (n <= 0)
            return 0;
        //默认最大不能跳过2048个字节
        int size = (int)Math.min(MAX_SKIP_BUFFER_SIZE, remaining);
        byte[] skipBuffer = new byte[size];
        while (remaining > 0) {
            nr = read(skipBuffer, 0, (int)Math.min(size, remaining));
            if (nr < 0)
                break;
            remaining -= nr;
        }
        return n - remaining;
    }

    //返回一共有多少个字节可读
    public int available() throws IOException {
        return 0;
    }

    //关闭输入流
    public void close() throws IOException {}

    //当读取不超过readlimit个字节时，调用reset方法必须能将read指针还原至标记处
    public synchronized void mark(int readlimit) {}

    //重置输入流包含的字节数据，默认实现为不支持
    public synchronized void reset() throws IOException {
        throw new IOException("mark/reset not supported");
    }
    //返回这个InputStream支不支持mark方法
    public boolean markSupported() {
        return false;
    }
}
```
 InputStream类提供了一系列基本的输入流操作，并提供了其默认的实现方法。

 mark方法的作用是在流当中设置一个标记，readlimit参数表示在标记失效前，最多可以读多少个字节。在设置标记后，你读取的字节数没有超过readlimit，那么你就可以通过调用reset方法将read指针重置到标记处，如果超过了readlimit，那么你调用reset方法后会抛出一个异常（视实现类的具体实现而定，有些类不会抛出异常）。

 InputStream实现了java.io.Closeable接口，代表是可关闭的，java.io.Closeable在JDK1.7之后实现了java.lang.AutoCloseable接口。实现了AutoCloseable接口的类可以再try的括号中声明，并且无须在finally块中显式调用close方法关闭。

 java.io包下的所有InputStream及其子类关系图：   
 ![这里写图片描述](https://img-blog.csdn.net/20180901205823986?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FiYzEyM2x6Zg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)   
 **InputStream子类使用场景简介：**   
 1、StringBufferInputStream（已过时，建议使用StringReader）：构造这个类必须传入一个String字符串，这个类的实例可以将这个字符串中的内容输出。   
 2、FilterInputStream：   
 这个类及其子类的主要作用是封装其它的InputStream实现类，并为它们提供额外的功能。其内部只有一个InputStream类型的成员变量in，并提供了protected权限的构造方法（构造时需传入InputStream），相当于其它InputStream的Facade（参考装饰器设计模式）。下面介绍其子类:   
 （1）DataInputStream：字面上为数据输入流，内部包装了一个InputStream，用于从字节流中读出基本数据类型：如int、double等，也可用于读出UTF编码的字符   
 （2）BufferedInputStream：其内部有一个缓冲区，提供缓冲功能提高IO速度，第二个作用是给其它InputStream类实现mark和reset功能。   
 （3）LineNumberInputStream（已过时，建议使用LineNumberReader）   
 （4）PushbackInputStream：提供了unread方法，可以把读到的数据回退到输入流中，供第二次读取。   
 3、ByteArrayInputStream：构造时可以传入一个byte数组，并以这个byte数组为数据源。   
 4、FileInputStream：构造时可以传入一个文件地址或文件描述符，并通过本地方法读取文件的数据。   
 5、SequenceInputStream: 在构造时可传入多个InputStream，用于将这些输入流合并   
 6、ObjectInputStream: 对象输入流，用于将二进制流反序列化成对象   
 7、PipedInputStream：管道输入流，它的作用是让多线程可以通过这个管道输入流进行线程间的通讯，在使用时必须和PipedOutputStream配套使用。

 
--------
 
### 三、字节输出流：OutputStream

 OutputStream是所有字节输出流的基类，提供了一系列方法来输出字节。

 
```
public abstract class OutputStream implements Closeable, Flushable {
    //将单个字节b写入到输出流
    public abstract void write(int b) throws IOException;
    //将b.length个字节从指定的byte数组b写入此输出流
    public void write(byte b[]) throws IOException {
        write(b, 0, b.length);
    }
    //将数组b从off位置开始写入len个字节到输出流
    public void write(byte b[], int off, int len) throws IOException {
        if (b == null) {
            throw new NullPointerException();
        } else if ((off < 0) || (off > b.length) || (len < 0) ||
                   ((off + len) > b.length) || ((off + len) < 0)) {
            throw new IndexOutOfBoundsException();
        } else if (len == 0) {
            return;
        }
        for (int i = 0 ; i < len ; i++) {
            write(b[off + i]);
        }
    }
    //执行输出
    public void flush() throws IOException {
    }
    //关闭输出流
    public void close() throws IOException {
    }
}
```
 相对InputStream来说，OutputStream实现要简单些。   
 OutputStream及其子类结构图：   
 ![这里写图片描述](https://img-blog.csdn.net/20180901212301920?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FiYzEyM2x6Zg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)   
 **OutputStream子类使用场景简介：**   
 1、PipedOutputStream：管道输出流，它的作用是让多线程可以通过这个管道输出流进行线程间的通讯，在使用时必须和PipedInputStream配套使用   
 2、FilterOutputStream：类似FilterInputStream，将OutputStream进行包装，并为它们提供额外的功能。   
 （1）BufferedOutputStream：提供一个缓冲的功能，可以将其中的字节数据一次性全部输出，防止频繁的输出数据带来的性能下降。   
 （2）DataOutputStream：类似DataInputStream，可以将二进制数据按照一定格式输出，比如输出int、long、或者UTF编码的字符等。   
 （3）PrintStream：将二进制数据按指定编码转换成字符串输出   
 3、ByteArrayOutputStream：以字节数组为数据输出   
 4、FileOutputStream：可以指定一个磁盘路径，并将二进制数据生成文件写到指定的路径中   
 5、ObjectOutputStream：将对象序列化为二进制数据并输出。

 
--------
 
### 四、字符输入流：Reader

 Reader类java.io包中所有字符输入流的基类，定义了基本的方法实现并提供了部分方法的默认实现。   
 子类大多会重写里面的大部分方法

 
```
public abstract class Reader implements Readable, Closeable {
    //私有锁，用于保证线程安全
    protected Object lock;

    //默认私有锁为this
    protected Reader() {
        this.lock = this;
    }

    //可以指定一个对象作为私有锁
    protected Reader(Object lock) {
        if (lock == null)
            throw new NullPointerException();
        this.lock = lock;
    }

    //将数据读到CharBuffer(java.nio包下的类)中
    public int read(CharBuffer target) throws IOException {
        int len = target.remaining();
        char[] cbuf = new char[len];
        int n = read(cbuf, 0, len);
        if (n > 0)
            target.put(cbuf, 0, n);
        return n;
    }

    //从输入流中读出一个字符
    public int read() throws IOException {
        char cb[] = new char[1];
        if (read(cb, 0, 1) == -1)
            return -1;
        else
            return cb[0];
    }

    //将输入流的数据读到字符数组cbuf中
    public int read(char cbuf[]) throws IOException {
        return read(cbuf, 0, cbuf.length);
    }

    //从cbuf[off]位置开始，读入len个char字符到cbuf数组中
    abstract public int read(char cbuf[], int off, int len) throws IOException;
    //skip方法最大跳过值
    private static final int maxSkipBufferSize = 8192;
    //将跳过的字符存放在这个数组中
    private char skipBuffer[] = null;

    //跳过n个字符，返回实际跳过的字符数量(只会小于等于n)
    public long skip(long n) throws IOException {
        if (n < 0L)
            throw new IllegalArgumentException("skip value is negative");
        //n只能介于0~8192之间
        int nn = (int) Math.min(n, maxSkipBufferSize);
        synchronized (lock) {
            if ((skipBuffer == null) || (skipBuffer.length < nn))
                skipBuffer = new char[nn];
            long r = n;
            while (r > 0) {
                int nc = read(skipBuffer, 0, (int)Math.min(r, nn));
                //读到-1说明读到末尾
                if (nc == -1)
                    break;
                r -= nc;
            }
            return n - r;
        }
    }

    //返回Reader是否就绪
    public boolean ready() throws IOException {
        return false;
    }

    //是否支持mark，mark、reset作用前面已经说过
    public boolean markSupported() {
        return false;
    }

    public void mark(int readAheadLimit) throws IOException {
        throw new IOException("mark() not supported");
    }

    public void reset() throws IOException {
        throw new IOException("reset() not supported");
    }
    //关闭输入流
    abstract public void close() throws IOException;
}
```
 Reader实现了java.nio.Readable和java.io.Closeable接口。Readable接口要求实现类可以将字符读取到指定的字符缓冲中(java.nio.CharBuffer)。

 Reader的方法构成大致和InputStream差不多，只不过Reader提供了两个成员变量：lock和skipBuffer，前者用来实现线程安全（Reader要求子类要实现线程安全），后者用来存储跳过的字符。

 Reader及其子类结构图：   
 ![这里写图片描述](https://img-blog.csdn.net/20180902145747525?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FiYzEyM2x6Zg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)   
 **Reader子类使用场景简介：**   
 1、BufferedReader：和BufferedInputStream类似，将一个Reader实例包装后实现其额外的功能，内部实现了一个缓冲区可将数据分块读入，避免一个字符一个字符读入影响性能。   
 LineNumberReader：BufferedReader的子类，同样将一个Reader包装，这个类提供了readLine等方法，可以将字符一行一行（以换行符为界）地读入。   
 2、CharArrayReader：类似ByteArrayInputStream，构造时需要传入一个char数组并将这个char数组作为作为输入源。   
 3、InputStreamReader：InputStream转换成Reader的桥梁，构造时必须传入一个InputStream对象，可以传入一个编码方式。这个类会将InputStream中的二进制流以指定的编码方式编码成字符串。   
 FileReader：InputStreamReader的子类，用于读取文件并转换成字符形式，内部实现很简单，可以自己去看看源码。   
 4、FilterReader：功能类似FilterInputStream，只不过这是个抽象类。   
 PushbackReader：类似PushbackInputStream，可以将读出的字符回退到输入流，供第二次读取。   
 5、PipedReader：管道字符输入流，类似PipedInputStream，用于线程间的通信，需要配合PipedWriter使用。   
 6、StringReader：将一个String字符串作为输入源。

 
--------
 
### 五、字符输入流：Writer

 Writer是java.io包中所有字符输出流的基类，提供基本的方法定义和默认的方法实现用来输出数据。

 
```
public abstract class Writer implements Appendable, Closeable, Flushable {
    //存放输出数据的缓冲区
    private char[] writeBuffer;

    //writeBuffer最大长度
    private static final int WRITE_BUFFER_SIZE = 1024;

    //私有锁，用于实现线程安全
    protected Object lock;

    //无参构造器，默认私有锁的实现为this
    protected Writer() {
        this.lock = this;
    }

    //可以指定一个锁
    protected Writer(Object lock) {
        if (lock == null)
            throw new NullPointerException();
        this.lock = lock;

    //写入一个字符c
    public void write(int c) throws IOException {
        synchronized (lock) {
            if (writeBuffer == null)
                writeBuffer = new char[WRITE_BUFFER_SIZE];
            writeBuffer[0] = (char) c;
            write(writeBuffer, 0, 1);
        }
    }

    //将char数组中的字符写入到Writer末尾
    public void write(char cbuf[]) throws IOException {
        write(cbuf, 0, cbuf.length);
    }

    //从char数组cbuf[off]开始写入len个字符到Writer末尾
    abstract public void write(char cbuf[], int off, int len) throws IOException;

    //将字符串str中的数据写入
    public void write(String str) throws IOException {
        write(str, 0, str.length());
    }

    //从字符串str的第off个字符开始写入len个字符到Writer末尾
    public void write(String str, int off, int len) throws IOException {
        synchronized (lock) {
            char cbuf[];
            if (len <= WRITE_BUFFER_SIZE) {
                if (writeBuffer == null) {
                    writeBuffer = new char[WRITE_BUFFER_SIZE];
                }
                cbuf = writeBuffer;
            } else {    // Don't permanently allocate very large buffers.
                cbuf = new char[len];
            }
            str.getChars(off, (off + len), cbuf, 0);
            write(cbuf, 0, len);
        }
    }

    //将CharSequence(实现类主要有String、StringBuilder、StringBuffer)写入到Writer末尾，返回this
    public Writer append(CharSequence csq) throws IOException {
        if (csq == null)
            write("null");
        else
            write(csq.toString());
        return this;
    }
    //将CharSequence的start位置到end位置之间的字符写入到Writer末尾
    public Writer append(CharSequence csq, int start, int end) throws IOException {
        CharSequence cs = (csq == null ? "null" : csq);
        write(cs.subSequence(start, end).toString());
        return this;
    }
    //将单个字符c写入到Writer末尾
    public Writer append(char c) throws IOException {
        write(c);
        return this;
    }
    //刷新缓冲区，执行输出
    abstract public void flush() throws IOException;
    //关闭输出流
    abstract public void close() throws IOException;
}
```
 虽然看上去有很多重载的write方法，但实际上都间接调用了抽象方法write(char cbuf[], int off, int len)这个方法。   
 私有成员变量writerBuffer主要用在write(String, int, int)上，用于存储截取出来的字符串并一次性调用上述抽象write方法。

 Writer及其子类结构图：   
 ![这里写图片描述](https://img-blog.csdn.net/20180901220404563?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FiYzEyM2x6Zg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)   
 **Writer子类使用场景简介：**   
 1、BufferedWriter：类似BufferedOutputStream，将一个Writer实例包装，可以将这个被包装的Writer中的数据存入内部的缓冲区，将其一次性输出，减少反复输出带来的性能损耗。   
 2、PrintWriter：也是利用了装饰者设计模式，可以包装一个Writer或OutputStream，可以将里面的数据按照格式输出成字符串形式。   
 3、CharArrayWriter：类似ByteArrayOutputStream，以一个char数组为输出源。   
 4、OutputStreamWriter：类似于InputStreamWriter，是OutputStream和Writer的一个桥梁。可以将OutputStream的二进制数据转换成字符形式（可以自定义编码方式）输出。   
 FileWriter：类似FileReader，是OutputStreamWriter的子类。可以用来读入一个文件中的数据并转换为字符形式。   
 5、StringWriter：内部封装了一个StringBuffer，并以这个StringBuffer作为输出源。   
 6、PipedWrtier：参照PipedReader，用于线程通信。   
 7、FilterWriter：抽象类，包装一个Writer实现类。这个类在java.io包中没有子类

   
  