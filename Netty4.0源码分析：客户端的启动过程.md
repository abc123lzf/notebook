---
title: Netty4.0源码分析：客户端的启动过程
date: 2018-10-20 23:11:50
tags: CSDN迁移
---
 版权声明：尊重原创，转载请注明出处 [ https://blog.csdn.net/abc123lzf/article/details/83152303]( https://blog.csdn.net/abc123lzf/article/details/83152303)   
  博主水平有限，如有错误欢迎纠正

 
### []()一、引言

 开始学习Netty源码可以从它的启动过程入手。Netty客户端的启动需要构造一个io.netty.bootstrap.Bootstrap类，并可以通过它设置一系列参数。  
 我们从下面这个代码入手：

 
```
public static void main(String[] args) throws InterruptedException {
	Bootstrap boot = new Bootstrap();
	boot.group(new NioEventLoopGroup())
		.channel(NioSocketChannel.class)
		.option(ChannelOption.SO_TIMEOUT, 5000)
		.handler(new ChannelInitializer<SocketChannel>() {
			@Override
			protected void initChannel(SocketChannel ch) throws Exception {
				ChannelPipeline pipeline = ch.pipeline();
				pipeline.addLast(new HttpResponseEncoder());
				pipeline.addLast(new HttpRequestDecoder());
			}
		});
	
	ChannelFuture future = boot.bind("127.0.0.1", 8888).sync();
	future.channel().closeFuture().sync();
}

```
 客户端的启动可以分为以下几个步骤：  
 1、构造Bootstrap类，这个类专门用来引导客户端  
 2、绑定一个EventLoopGroup作为处理连接的线程池，这的NioEventLoopGroup基于异步IO实现。  
 3、指定Channel通道的类型，这里指定了NioSocketChannel  
 4、指定连接参数（可选）  
 5、添加进站处理器和出站处理器用于处理客户端接收到的信息和发送的信息，这里添加了一个HTTP请求解码器和HTTP响应编码器。  
 6、连接指定的IP和端口。

 
### []()二、Channel初始化

 Netty框架实现了自己的Channel，作为Socket的抽象，每当Netty建立了一个Socket连接后都会随之建立对应的Channel。Netty自己实现的Channel实际上是对JDK中的NIO做了一层封装。  
 NioSocketChannel继承关系图如下：  
 ![在这里插入图片描述](https://img-blog.csdn.net/20181018211522648?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FiYzEyM2x6Zg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

 除了NioSocketChannel，Netty还提供了其它SocketChannel接口实现类：  
 NioServerSocketChannel：用于异步IO的服务端TCP连接  
 NioDatagramChannel：用于异步IO的UDP连接  
 NioSctpChannel：SCTP协议的异步IO连接  
 NioSctpServerChannel：SCTP协议的服务端异步IO连接  
 OioSocketChannel：阻塞IO实现的客户端连接  
 OioServerSocketChannel：阻塞IO实现的服务端连接  
 OioDatagramSocketChannel：阻塞IO实现的UDP的服务端连接  
 OioSctpChannel：阻塞IO的SCTP连接  
 OioSctpServerChannel：阻塞IO的SCTP服务端连接

 NioSocketChannel类提供了4种public构造方法：

 
     方法签名                                     | 作用                                                             
     ---------------------------------------- | --------------------------------------------------------------- 
     NioSocketChannel()                       | 构造一个默认的SocketChannel，Selector为默认的Selector                      
     NioSocketChannel(SelectorProvider)       | 构造一个SocketChannel，其Selector是这个指定的SelectorProvider的provider方法返回值
     NioSocketChannel(SocketChannel)          | 构造一个SocketChannel，将指定的java.nio.SocketChannel作为Channel通道        
     NioSocketChannel(Channel, SocketChannel) | 构造一个SocketChannel，并指定一个父Channel和java.nio.SocketChannel         

这些构造方法的源码如下：

 
```
public NioSocketChannel() {
	this(newSocket(DEFAULT_SELECTOR_PROVIDER));
}
public NioSocketChannel(SelectorProvider provider) {
    this(newSocket(provider));
}
public NioSocketChannel(SocketChannel socket) {
	this(null, socket);
}
public NioSocketChannel(Channel parent, SocketChannel socket) {
	super(parent, socket);
    config = new NioSocketChannelConfig(this, socket.socket());
}

```
 这些构造方法允许用户指定自己的SelectorProvider（SelectorProvider是生产NIO组件的工厂类）、java.nio.SocketChannel和父Channel。  
 如果没有指定SelectorProvider，那么则通过默认的SelectorProvider构造一个SocketChannel。

 
```
private static final SelectorProvider DEFAULT_SELECTOR_PROVIDER = SelectorProvider.provider();

private static SocketChannel newSocket(SelectorProvider provider) {
    try {
        return provider.openSocketChannel();
    } catch (IOException e) {
        throw new ChannelException("Failed to open a socket.", e);
    }
}

```
 我们在通过Bootstrap类引导的时候，调用channel方法时传入的是SocketChannel接口的实现类，channel方法的实现在Bootstrap的父类AbstractBootstrap，其中泛型参数B是Bootstrap，参数C是Channel：

 
```
public B channel(Class<? extends C> channelClass) {
	//省略参数检查代码
    return channelFactory(new BootstrapChannelFactory<C>(channelClass));
}

```
 channel方法会根据Channel的Class对象构造一个BootstrapChannelFactory工厂类：

 
```
private static final class BootstrapChannelFactory<T extends Channel> implements ChannelFactory<T> {
    private final Class<? extends T> clazz;
    BootstrapChannelFactory(Class<? extends T> clazz) {
        this.clazz = clazz;
    }
    @Override public T newChannel() {
        try {
            return clazz.getConstructor().newInstance();
        } catch (Throwable t) {
            throw new ChannelException("Unable to create Channel from class " + clazz, t);
        }
    }
    @Override public String toString() {
        return StringUtil.simpleClassName(clazz) + ".class";
    }
}

```
 这个工厂类通过Channel类的无参构造方法构造Channel的实现类并返回。  
 工厂对象构造完成后，Bootstrap类随即会调用channelFactory方法：

 
```
public B channelFactory(ChannelFactory<? extends C> channelFactory) {
    //省略参数检查代码
    this.channelFactory = channelFactory;
    return self();
}

```
 现在我们回到NioSocketChannel的构造方法中，NioSocketChannel构造方法首先会调用父类AbstractNioByteChannel的构造方法：

 
```
protected AbstractNioByteChannel(Channel parent, SelectableChannel ch) {
	super(parent, ch, SelectionKey.OP_READ);
}

```
 这里继续调用了父类AbstractNioChannel的构造方法，并传入了SelectionKey.OP_READ作为变量readInterestOp的值

 
```
protected AbstractNioChannel(Channel parent, SelectableChannel ch, int readInterestOp) {
    super(parent);
    this.ch = ch;
    this.readInterestOp = readInterestOp;
    try {
    	//配置为非阻塞IO
        ch.configureBlocking(false);
    } catch (IOException e) {
        try {
            ch.close();
        } catch (IOException e2) {
            if (logger.isWarnEnabled()) {
                logger.warn("Failed to close a partially initialized socket.", e2);
            }
        }
        throw new ChannelException("Failed to enter non-blocking mode.", e);
    }
}

```
 再是父类AbstractChannel的构造方法：

 
```
protected AbstractChannel(Channel parent) {
	this.parent = parent;
    unsafe = newUnsafe();
    pipeline = newChannelPipeline();
}

```
 **在初始化过程中，一共初始化了以下成员变量：**  
 AbstractChannel：  
 1、指定父Channel通道（如果在构造NioSocketChannel时不为null）  
 2、unsafe变量通过newUnsafe方法指定，newUnsafe方法在AbstractChannel类中是一个抽象方法，其子类NioSocketChannel重写了这个方法：

 
```
@Override
protected AbstractNioUnsafe newUnsafe() {
	return new NioSocketChannelUnsafe();
}

```
 NioSocketChannelUnsafe是NioSocketChannel的内部类，它的继承关系如下：  
 ![在这里插入图片描述](https://img-blog.csdn.net/20181019110537658?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FiYzEyM2x6Zg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

 Unsafe接口及其实现类主要实现了底层的Channel读写操作，是Netty与JDK NIO实现的桥梁，并且存储了一系列连接信息，这个接口属于Channel的内部接口，访问权限为包访问权限。  
 3、调用newChannelPipeline方法设置该Channel默认的管道（ChannelPipeline）实现类：

 
```
protected DefaultChannelPipeline newChannelPipeline() {
	return new DefaultChannelPipeline(this);
}

```
 每个Channel的构造都会实例化一个DefaultChannelPipeline：

 
```
protected DefaultChannelPipeline(Channel channel) {
    this.channel = ObjectUtil.checkNotNull(channel, "channel");
	//构造一个双向链表
    tail = new TailContext(this);
    head = new HeadContext(this);
    head.next = tail;
    tail.prev = head;
}

```
 DefaultChannelPipeline在内部维护了一个双向链表，其TailContext和HeadContext都是AbstractChannelHandlerContext的子类，并且TailContext实现了ChannelInboundHandler，HeadContext实现了ChannelOutboundHandler接口。Netty通过这个来实现其管道机制，这两个类很重要。

 AbstractNioChannel的属性：  
 1、将指定的SelectableChannel（java.nio.SocketChannel的父类）实现类赋给成员变量ch  
 2、将成员变量readInterestOp赋值为SelectionKey.OP_READ  
 赋值完成后，随即设为非阻塞模式。

 NioSocketChannel的属性：  
 将成员变量config赋值为new NioSocketChannelConfig(this, socket.socket())

 
### []()三、EventLoopGroup的初始化

 在Netty的线程模型中，EventLoop将由一个永远不会改变的Thread驱动，同时任务可以直接提交给EventLoop实现，已达到立刻执行或者调度执行的目的，EventLoop接口只提供了一个方法：

 
```
public interface EventLoop extends OrderedEventExecutor, EventLoopGroup {
	@Override EventLoopGroup parent();
}

```
 parent方法用于返回这个EventLoop所属的EventLoopGroup。

 EventLoop、EventLoopGroup主要实现类和接口结构图：  
 ![在这里插入图片描述](https://img-blog.csdn.net/20181019145210812?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FiYzEyM2x6Zg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

 可以看出，EventLoopGroup是一个ScheduledExecutorService线程池接口实现类，可以把它当成是一个多个线程组成线程池。EventLoop同样也可以看成是一个“线程池”，但它只有一个线程。NioEventLoopGroup提供了6个public构造方法，可以指定这个NioEventLoopGroup的线程数量、线程工厂、SelectorProvider、SelectStrategyFactory、RejectedExecutionHandler（拒绝执行处理器）。这些构造方法会调用父类MultithreadEventLoopGroup的构造方法：

 
```
protected MultithreadEventLoopGroup(int nThreads, ThreadFactory threadFactory, Object... args) {
	super(nThreads == 0? DEFAULT_EVENT_LOOP_THREADS : nThreads, threadFactory, args);
}

```
 如果没有指定线程数量或设置为0，那么线程数量会默认设置为当前机器的CPU逻辑处理器数量的2倍。  
 下面来看父类MultithreadEventExecutorGroup的构造方法，其中args参数第1个参数为指定的SelectorProvider对象，第2个参数为SelectStrategyFactory对象，第3个参数为拒绝执行处理器RejectedExecutionHandler。

 
```
protected MultithreadEventExecutorGroup(int nThreads, ThreadFactory threadFactory, Object... args) {
    if (nThreads <= 0)
        throw new IllegalArgumentException(String.format("nThreads: %d (expected: > 0)", nThreads));
	//如果没有指定线程工厂，那么由Netty设置默认的线程工厂
    if (threadFactory == null)
        threadFactory = newDefaultThreadFactory();
	//创建数组，用于保存这个EventLoopGroup拥有的EventLoop
    children = new SingleThreadEventExecutor[nThreads];
    //如果线程数是2的幂次，那么采用n&length-1计算下一个EventLoop，否则采用取模法计算
    if (isPowerOfTwo(children.length)) {
        chooser = new PowerOfTwoEventExecutorChooser();
    } else {
        chooser = new GenericEventExecutorChooser();
    }

    for (int i = 0; i < nThreads; i ++) {
        boolean success = false;
        try {
            children[i] = newChild(threadFactory, args);
            success = true;
        } catch (Exception e) {
            throw new IllegalStateException("failed to create a child event loop", e);
        } finally {
            if (!success) {
               //省略异常处理代码...
            }
        }
    }

    final FutureListener<Object> terminationListener = new FutureListener<Object>() {
        @Override
        public void operationComplete(Future<Object> future) throws Exception {
            if (terminatedChildren.incrementAndGet() == children.length) {
                terminationFuture.setSuccess(null);
            }
        }
    };

    for (EventExecutor e: children) {
        e.terminationFuture().addListener(terminationListener);
    }
}

```
 MultithreadEventExecutorGroup构造方法首先判定有没有指定线程工厂，如果没有指定则调用newDefaultThreadFactory构造一个线程工厂：

 
```
protected ThreadFactory newDefaultThreadFactory() {
	return new DefaultThreadFactory(getClass());
}

```
 DefaultThreadFactory创造的线程对象为FastThreadLocalThread，它继承线程类Thread：

 
```
public class FastThreadLocalThread extends Thread {
	private final boolean cleanupFastThreadLocals;
	private InternalThreadLocalMap threadLocalMap;
}

```
 FastThreadLocalThread实现了自己的InternalThreadLocalMap，用于存储线程私有的变量。  
 构造线程完成后，线程工厂会将这个线程设为守护线程并将优先级设为指定的优先级（如果没有指定则默认优先级为5）

 随后，会构造一个SingleThreadEventExecutor类型的数组并将长度设置为指定的线程数并赋给成员变量children，随后会根据数组长度是否为2的幂次来定义访问下一个children数组的方式，接着会通过循环调用newChild来构造EventLoop来填满children数组。  
 newChild方法如下：

 
```
@Override
protected EventExecutor newChild(ThreadFactory threadFactory, Object... args) throws Exception {
	return new NioEventLoop(this, threadFactory, (SelectorProvider) args[0],
		((SelectStrategyFactory) args[1]).newSelectStrategy(), (RejectedExecutionHandler) args[2]);
}

```
 newChild方法会根据构造方法传入的一系列参数来构造NioEventLoop实例，NioEventLoop继承自抽象类SingleThreadEventExecutor。这样，children数组就相当于EventLoopGroup线程池的工作线程。

 
### []()四、Bootstrap的引导

 看完Channel和EventLoopGroup的初始化过程后，我们现在来分析Bootstrap引导过程。

 
##### []()1、调用group方法绑定一个EventLoopGroup

 
```
public B group(EventLoopGroup group) {
    //省略参数检查代码...
    this.group = group;
    return self(); //返回this
}

```
 group方法很简单，就是将构造完成的EventLoopGroup绑定到Bootstrap中

 
##### []()2、指定Channel的实现类

 前面已经讲解了

 
##### []()3、指定连接参数

 可以调用option方法在连接前指定一系列连接参数：

 
```
public <T> B option(ChannelOption<T> option, T value) {
    //省略参数检查代码
    //如果value为null就移除这个参数
    if (value == null) {
        synchronized (options) {
            options.remove(option);
        }
    //添加参数
    } else {
        synchronized (options) {
            options.put(option, value);
        }
    }
    return self();
}

```
 Bootstrap的父类AbstractBootstrap的options变量指定了一个LinkedHashMap，用来存储参数值。  
 关于参数的设置可以参考这篇文章：[https://blog.csdn.net/zhousenshan/article/details/72859923](https://blog.csdn.net/zhousenshan/article/details/72859923)

 
##### []()4、指定ChannelHandler

 可以调用handler方法指定ChannelHandler用于指定一个ChannelHandler处理请求，一般为ChannelInitializer，ChannelInitializer在初始化过程中被临时添加到ChannelPipeline，等到引导过程即将完成时，会调用handlerAdded方法（这个方法是ChannelHandler接口要求实现的），而handlerAdded方法又会调用重写的initChannel方法，然后从DefaultChannelPipeline维护的队列中移除这个ChannelInitializer

 
##### []()5、连接

 调用Bootstrap的connect方法（有多个重载的方法）可以向指定的IP和端口发起TCP连接：

 
```
public ChannelFuture connect(SocketAddress remoteAddress) {
    //省略检查参数代码
    validate();
    return doConnect(remoteAddress, localAddress());
}

```
 connect方法首先会检查是否指定了EventLoopGroup和ChannelHandler，否则会抛出异常。检查完后，就会调用doConnect方法来执行连接

 
```
private ChannelFuture doConnect(final SocketAddress remoteAddress, final SocketAddress localAddress) {
	//构造Channel并注册Channel
    final ChannelFuture regFuture = initAndRegister();
    //获取刚刚初始化完成的Channel
    final Channel channel = regFuture.channel();
    if (regFuture.cause() != null)
        return regFuture;
    //获取它的DefaultChannelPromise
    final ChannelPromise promise = channel.newPromise();
    //如果通道注册已完成，那么直接建立连接
    if (regFuture.isDone()) {
        doConnect0(regFuture, channel, remoteAddress, localAddress, promise);
    //否则加入监听器，异步等到注册完成再建立连接
    } else {
        regFuture.addListener(new ChannelFutureListener() {
            @Override
            public void operationComplete(ChannelFuture future) throws Exception {
                doConnect0(regFuture, channel, remoteAddress, localAddress, promise);
            }
        });
    }
    return promise;
}

```
 doConnect方法可以简单概括为：  
 1、首先调用initAndRegister初始化一个Channel并注册这个Channel（异步调用）  
 2、根据这个Channel构造它的DefaultChannelPromise  
 3、如果此时Channel已经注册完成，那么调用doConnect0方法尝试发起连接。如果还尚未注册完成，那么往ChannelFuture中添加一个监听器，等到注册完成后再调用doConnect0方法

 下面我们来逐步分析doConnect方法中调用的比较重要的方法  
 **（1）initAndRegister方法**

 
```
final ChannelFuture initAndRegister() {
    Channel channel = null;
    try {
    	//构造指定的Channel（以上面的例子为例，构造的Channel为NioSocketChannel）
        channel = channelFactory().newChannel();
        init(channel);
    } catch (Throwable t) {
        //省略处理异常代码
    }
	//将Channel注册到EventLoopGroup(EventLoop)中
    ChannelFuture regFuture = group().register(channel);
    if (regFuture.cause() != null) {
        if (channel.isRegistered()) {
            channel.close();
        } else {
            channel.unsafe().closeForcibly();
        }
    }
    return regFuture;
}

```
 init方法会将Bootstrap中指定的ChannelHandler添加到这个Channel所属的ChannelPipeline的尾部，然后，将连接参数和Attribute添加到AttributeMap中。

 
```
@Override @SuppressWarnings("unchecked")
void init(Channel channel) throws Exception {
	//获取该Channel的所属的管道
    ChannelPipeline p = channel.pipeline();
    //将ChannelInitializer添加到管道尾部
    p.addLast(handler());
    //获取指定的连接参数Map并调用setChannelOptions将参数添加到Channel中
    final Map<ChannelOption<?>, Object> options = options();
    synchronized (options) {
        setChannelOptions(channel, options, logger);
    }
	//获取attr方法设置的Attribute，用于设置到AttributeMap中
    final Map<AttributeKey<?>, Object> attrs = attrs();
    synchronized (attrs) {
        for (Entry<AttributeKey<?>, Object> e: attrs.entrySet()) {
            channel.attr((AttributeKey<Object>) e.getKey()).set(e.getValue());
        }
    }
}

```
 Channel的ChannelPipeline属于DefaultChannelPipeline，现在来看DefaultChannelPipeline的addLast方法实现：

 
```
@Override
public final ChannelPipeline addLast(ChannelHandler... handlers) {
	return addLast(null, handlers);
}

@Override
public final ChannelPipeline addLast(EventExecutorGroup executor, ChannelHandler... handlers) {
    //省略参数检查代码
    for (ChannelHandler h: handlers) {
        if (h == null)
            break;
        addLast(executor, null, h);
    }
    return this;
}

@Override
public final ChannelPipeline addLast(EventExecutorGroup group, String name, ChannelHandler handler) {
    final AbstractChannelHandlerContext newCtx;
    synchronized (this) {
    	//检查是否属于ChannelHandlerAdapter，如果属于那么检查它是否已经添加过了并且不是@Sharable
        checkMultiplicity(handler);
        //构造一个DefaultChannelHandlerContext
        newCtx = newContext(group, filterName(name, handler), handler);
        //添加到管道尾部，并构造好一个双向链表
        addLast0(newCtx);
        //如果之前没有添加过ChannelHandler
        if (!registered) {
            newCtx.setAddPending();
            callHandlerCallbackLater(newCtx, true);
            return this;
        }
        EventExecutor executor = newCtx.executor();
        if (!executor.inEventLoop()) {
            newCtx.setAddPending();
            executor.execute(new Runnable() {
                @Override
                public void run() {
                    callHandlerAdded0(newCtx);
                }
            });
            return this;
        }
    }
    callHandlerAdded0(newCtx);
    return this;
}

```
 调用addLast0方法后，我们指定的ChannelInitializer被添加到了管道尾部，DefaultChannelPipeline在内部维护了一个双向链表，并用HeadContext和TailContext作为头、尾结点（HeadContext和TailContext即是ChannelHandlerContext也是ChannelHandler）：  
 ![在这里插入图片描述](https://img-blog.csdn.net/20181020162858955?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FiYzEyM2x6Zg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

 register方法调用了指定的EventLoopGroup（或者EventLoop）的register方法，如果是EventLoopGroup，会调用next方法获取children数组的下一个EventLoop，然后调用它的register方法：

 
```
@Override
public ChannelFuture register(Channel channel) {
	return register(channel, new DefaultChannelPromise(channel, this));
}

@Override
public ChannelFuture register(final Channel channel, final ChannelPromise promise) {
    //省略检查参数代码
    channel.unsafe().register(this, promise);
    return promise;
}

//Unsafe中的register方法
@Override
public final void register(EventLoop eventLoop, final ChannelPromise promise) {
    //省略异常检查代码...
    //获取EventLoop
    AbstractChannel.this.eventLoop = eventLoop;
    if (eventLoop.inEventLoop()) {
        register0(promise);
    } else {
        try {
            eventLoop.execute(new Runnable() {
                @Override public void run() {
                    register0(promise);
                }
            });
        } catch (Throwable t) {
           //省略异常检查代码
        }
    }
}

private void register0(ChannelPromise promise) {
    try {
        if (!promise.setUncancellable() || !ensureOpen(promise))
            return;
        boolean firstRegistration = neverRegistered;
        //将Channel注册到Selector上，并将返回的SelectionKey赋给变量
        doRegister();
        neverRegistered = false;
        registered = true;
        pipeline.invokeHandlerAddedIfNeeded();
        safeSetSuccess(promise);
        //调用管道的fireChannelRegistered方法，表示Channel已经注册
        pipeline.fireChannelRegistered();
        if (isActive()) {
            if (firstRegistration) {
            	//通知管道Handler，管道已经激活
                pipeline.fireChannelActive();
            } else if (config().isAutoRead()) {
                beginRead();
            }
        }
    } catch (Throwable t) {
        // 省略异常处理
    }
}

```
 **回到addLast方法中：**  
 当我们将Bootstrap指定的初始ChannelHandler添加到管道后，callHandlerAdded0方法随即会被调用  
 callHandlerAdded0方法会调用Bootstrap通过handler指定的ChannelHandler的handlerAdded方法

 
```
private void callHandlerAdded0(final AbstractChannelHandlerContext ctx) {
	 try {
		ctx.setAddComplete(); //将handlerState设为2，表示已完成Handler的添加操作
        ctx.handler().handlerAdded(ctx);
     } catch(Throwable t){
     	//省略异常处理代码
     }
}

```
 在上面的例子中，我们传入的是ChannelInitializer的实例，所以这里调用的是ChannelInitializer的handlerAdd方法：

 
```
@Override
public void handlerAdded(ChannelHandlerContext ctx) throws Exception {
    if (ctx.channel().isRegistered()) {
         initChannel(ctx);
    }
}

@SuppressWarnings("unchecked")
private boolean initChannel(ChannelHandlerContext ctx) throws Exception {
    if (initMap.putIfAbsent(ctx, Boolean.TRUE) == null) { // Guard against re-entrance.
        try {
        	//initChannel是我们在例子中重写的代码，主要是用来添加我们需要的ChannelHandler
            initChannel((C) ctx.channel());
        } catch (Throwable cause) {
            exceptionCaught(ctx, cause);
        } finally {
        	//最后将这个ChannelInitializer从管道中移除
            remove(ctx);
        }
        return true;
    }
    return false;
}

```
 handlerAdded会调用initChannel方法，而initChannel方法会调用我们重写的initChannel方法。从上面的例子中，我们重写了initChannel方法添加了一个HttpResponseEncoder和HttpRequestDecoder。

 图解：  
 调用initChannel方法后：  
 ![在这里插入图片描述](https://img-blog.csdn.net/20181020163315204?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FiYzEyM2x6Zg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)  
 调用remove方法后：  
 ![在这里插入图片描述](https://img-blog.csdn.net/20181020163510415?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FiYzEyM2x6Zg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

 initAndRegister主要分为两步：  
 1、构造一个Channel并将参数值、属性、对应的SelectorProvider添加到这个Channel中  
 2、将这个Channel注册到EventLoop中，并将包含的java.nio.SocketChannel注册到Selector中  
 **（2）doConnect0方法**  
 doConnect0会尝试建立连接：

 
```
private static void doConnect0(final ChannelFuture regFuture, final Channel channel,
    final SocketAddress remoteAddress, final SocketAddress localAddress, final ChannelPromise promise) {

    channel.eventLoop().execute(new Runnable() {
	    @Override public void run() {
	        if (regFuture.isSuccess()) { //正常情况下返回true
	        	//建立连接(以上面例子为例，这里调用的是AbstractChannel的connect方法)
	            if (localAddress == null) {
	                channel.connect(remoteAddress, promise);
	            } else {
	                channel.connect(remoteAddress, localAddress, promise);
	            }
	            //添加一个监听器，如果连接失败那么关闭Channel
	            promise.addListener(ChannelFutureListener.CLOSE_ON_FAILURE);
	        } else {
	            promise.setFailure(regFuture.cause());
	        }
	    }
	});
}

```
 这里会调用Channel的connect方法进行连接，以上面的例子，这里调用的是NioSocketChannel的connect方法：

 
```
@Override
public ChannelFuture connect(SocketAddress remoteAddress) {
	return pipeline.connect(remoteAddress);
}

```
 Channel的connect方法默认是调用它所属的管道（DefaultChannelPipeline）的connect方法：

 
```
@Override
public final ChannelFuture connect(SocketAddress remoteAddress) {
	return tail.connect(remoteAddress);
}

```
 这里调用了尾结点（类型TailContext）的connect方法，刚才在initAndRegister调用的init方法中我们已经把ChannelInitializer实例添加到了管道尾部，通过TailContext封装，所以这里会调用TailContext的connect方法，但是TailContext没有重写connect方法，所以这里调用的是父类AbstractChannelHandlerContext的的connect方法：

 
```
@Override
public ChannelFuture connect(final SocketAddress remoteAddress, final SocketAddress localAddress, 
		final ChannelPromise promise) {
    //省略参数、状态检测代码
    //找到最近的一个持有出站处理器的ChannelHandlerContext
    final AbstractChannelHandlerContext next = findContextOutbound();
    //获取EventExecutor
    EventExecutor executor = next.executor();
    //如果当前线程就是这个EventLoop持有的线程，那么直接调用invokeConnect方法执行
    if (executor.inEventLoop()) {
        next.invokeConnect(remoteAddress, localAddress, promise);
    } else {
    	//否则向这个EventLoop提交任务，由这个线程负责连接
        safeExecute(executor, new Runnable() {
            @Override
            public void run() {
                next.invokeConnect(remoteAddress, localAddress, promise);
            }
        }, promise, null);
    }
    return promise;
}

```
 connect方法首先会尝试找到最近的一个持有出站处理器的ChannelHandlerContext，以上面的例子为例，这个出站处理器就是HttpResponseEncoder。找到出站处理器后，就会获取这个ChannelHandlerContext所属的Channel的EventLoop。如果这个EventLoop持有的线程属于当前线程，那么直接调用invokeConnect方法，否则由EventLoop持有的线程异步执行。

 
```
private void invokeConnect(SocketAddress remoteAddress, SocketAddress localAddress, ChannelPromise promise) {
	//如果ChannelHandler已经初始化完成
    if (invokeHandler()) {
        try {
            ((ChannelOutboundHandler) handler()).connect(this, remoteAddress, localAddress, promise);
        } catch (Throwable t) {
            notifyOutboundHandlerException(t, promise);
        }
    } else {
        connect(remoteAddress, localAddress, promise);
    }
}

```
 这里继续调用其connect方法， 直到调用到HeadContext的connect方法：

 
```
@Override
public void connect(ChannelHandlerContext ctx, SocketAddress remoteAddress, 
		SocketAddress localAddress, ChannelPromise promise) throws Exception {
	unsafe.connect(remoteAddress, localAddress, promise);
}

```
 这里调用了AbstractNioUnsafe的connect方法：

 
```
@Override
public final void connect(final SocketAddress remoteAddress, final SocketAddress localAddress, 
		final ChannelPromise promise) {
    if (!promise.setUncancellable() || !ensureOpen(promise)) {
        return;
    }

    try {
        if (connectPromise != null)
            throw new ConnectionPendingException();
        boolean wasActive = isActive();
        //尝试建立连接(通过NIO的SocketChannel)
        if (doConnect(remoteAddress, localAddress)) {
            fulfillConnectPromise(promise, wasActive);
        } else {
            //省略连接失败处理代码
        }
    } catch (Throwable t) {
        //省略异常代码...
    }
}

@Override
protected boolean doConnect(SocketAddress remoteAddress, SocketAddress localAddress) throws Exception {
    if (localAddress != null) {
        doBind0(localAddress);
    }

    boolean success = false;
    try {
    	//通过JDK NIO的SocketChannel建立连接
        boolean connected = SocketUtils.connect(javaChannel(), remoteAddress);
        if (!connected) {
        	//关注连接事件
            selectionKey().interestOps(SelectionKey.OP_CONNECT);
        }
        success = true;
        return connected;
    } finally {
        if (!success) {
            doClose();
        }
    }
}

```
   
  