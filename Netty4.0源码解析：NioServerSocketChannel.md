---
title: Netty4.0源码解析：NioServerSocketChannel
date: 2018-10-28 17:07:04
tags: CSDN迁移
---
 版权声明：尊重原创，转载请注明出处 [ https://blog.csdn.net/abc123lzf/article/details/83447883]( https://blog.csdn.net/abc123lzf/article/details/83447883)   
  ### []()一、引言

 Netty的Channel在JDK NIO的Channel基础上做了一层封装，提供了更多的功能。Netty的中的Channel实现类主要有：NioServerSocketChannel（用于服务端非阻塞地接收TCP连接）、NioSocketChannel（用于维持非阻塞的TCP连接）、NioDatagramChannel（用于非阻塞地处理UDP连接）、OioServerSocketChannel（用于服务端阻塞地接收TCP连接）、OioSocketChannel（用于阻塞地接收TCP连接）、OioDatagramChannel（用于阻塞地处理UDP连接）：  
 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20181027170502978.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FiYzEyM2x6Zg==,size_27,color_FFFFFF,t_70)

 一个EventLoop一般持有多个Channel，每个EventLoop持有一个对应的线程，有这个线程负责处理这些Channel发出的事件。

 本篇文章我们先从服务端的NioServerSocketChannel开始分析：

 
--------
 
### []()二、初始化过程

 NioServerSocketChannel在JDK的ServerSocketChannel的基础上做了一层封装。在NioServerSocketChannel的初始化过程中，可以简要地分为三步：  
 1、实例化NioServerSocketChannel  
 2、完成NioServerSocketChannel的管道初始化的第一步骤（向管道添加初始的ChannelHandler）  
 3、将NioServerSocketChannel注册到NioEventLoopGroup，在此期间完成管道初始化的第二步骤（比如执行ChannelInitializer的handlerAdd方法）  
 4、绑定端口，开始接受客户端连接。

 我们在ServerBootstrap进行引导的过程中，需要调用channel方法指定一个ServerChannel实现类的Class对象，channel方法随即会生成一个ChannelFactory工厂于ServerBootstrap实例中。NioServerSocketChannel的实例化在ServerBootstrap的bind方法中完成。bind方法调用的doBind方法中的initAndRegister方法中，会通过ChannelFactory实例构造出一个NioServerSocketChannel：

 
```
final ChannelFuture initAndRegister() {
    Channel channel = null;
    try {
    	//通过ChannelFactory构造一个Channel实例
        channel = channelFactory().newChannel();
        init(channel);
    } catch (Throwable t) {
        if (channel != null) {
            channel.unsafe().closeForcibly();
            return new DefaultChannelPromise(channel, GlobalEventExecutor.INSTANCE).setFailure(t);
        }
        return new DefaultChannelPromise(new FailedChannel(), GlobalEventExecutor.INSTANCE).setFailure(t);
    }
	//获取bossGroup实例并调用register方法绑定
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
 通过ChannelFactory构造的Channel都是通过无参构造方法构造的，我们来分析NioServerSocketChannel的构造方法：

 
```
public NioServerSocketChannel() {
    this(newSocket(DEFAULT_SELECTOR_PROVIDER));
}

```
 这里通过静态newSocket方法构造了一个JDK的ServerSocketChannel：

 
```
private static final SelectorProvider DEFAULT_SELECTOR_PROVIDER = SelectorProvider.provider();
private static ServerSocketChannel newSocket(SelectorProvider provider) {
    try {
        return provider.openServerSocketChannel();
    } catch (IOException e) {
        throw new ChannelException("Failed to open a server socket.", e);
    }
}

```
 这里通过默认的SelectorProvider实例通过调用它的openServerSocketChannel构造了一个JDK的ServerSocketChannel实例，接着调用另外的重载的构造方法

 
```
public NioServerSocketChannel(ServerSocketChannel channel) {
    super(null, channel, SelectionKey.OP_ACCEPT);
    config = new NioServerSocketChannelConfig(this, javaChannel().socket());
}

```
 NioServerSocketChannelConfig是NioServerSocketChannel的内部类，保存了以下内容：NioServerSocketChannel（this）、和刚刚构造的ServerSocketChannel通过socket方法获取的ServerSocket实例、引导过程通过options方法设置的参数。

 NioServerSocketChannel父类AbstractNioMessageChannel的构造方法：

 
```
protected AbstractNioMessageChannel(Channel parent, SelectableChannel ch, int readInterestOp) {
    super(parent, ch, readInterestOp);
}

```
 AbstractNioMessageChannel父类AbstractNioChannel构造方法：

 
```
protected AbstractNioChannel(Channel parent, SelectableChannel ch, int readInterestOp) {
    super(parent);
    this.ch = ch; //传入JDK ServerSocketChannel
    this.readInterestOp = readInterestOp; //感兴趣通道事件为OP_ACCEPT
    try { //将ServerSocketChannel设为非阻塞
        ch.configureBlocking(false);
    } catch (IOException e) { //如果抛出异常那么关闭ServerSocketChannel
        try {
            ch.close();
        } catch (IOException e2) {
            if (logger.isWarnEnabled())
                logger.warn("Failed to close a partially initialized socket.", e2);
        }
        throw new ChannelException("Failed to enter non-blocking mode.", e);
    }
}

```
 AbstractNioChannel父类AbstractChannel构造方法：

 
```
protected AbstractChannel(Channel parent) {
    this.parent = parent; //ServerSocketChannel不存在父Channel
    unsafe = newUnsafe();
    pipeline = newChannelPipeline();
}

```
 ServerSocketChannel不存在父Channel，所以parent为null，只有SocketChannel存在（其parent为ServerSocketChannel实例）。  
 接着通过newUnsafe方法构造一个Channel.Unsafe接口实现类。在客户端引导过程已经提到过，Unsafe是Netty到JDK NIO的桥梁，Unsafe接口定义了以下方法：

 
     方法签名                                                       | 作用                                                                          
     ---------------------------------------------------------- | ---------------------------------------------------------------------------- 
     SocketAddress localAddress()                               | 获取该Channel的本地地址                                                             
     SocketAddress remoteAddress()                              | 获取该Channel连接地址                                                              
     void register(EventLoop, ChannelPromise)                   | 向EventLoop注册这个Channel，并通知指定的ChannelPromise                                  
     void bind(SocketAddress, ChannelPromise)                   | 指定一个本地地址，将Channel绑定在这个地址，并通知指定的ChannelPromise                               
     void connect(SocketAddress, SocketAddress, ChannelPromise) | 通过本地地址向指定的地址发起连接，并通知ChannelPromise                                          
     void disconnect(ChannelPromise)                            | 取消连接，并通知指定的ChannelPromise                                                   
     void close(ChannelPromise)                                 | 关闭Channel通道，并通知指定的ChannelPromise                                            
     void closeForcibly()                                       | 立刻关闭Channel通道，不通知ChannelPromise                                             
     void deregister(ChannelPromise)                            | 将这个Channel从EventLoop取消注册，并通知指定的ChannelPromise                               
     void beginRead()                                           | 写入缓冲区，以便进站处理器能够读到进站数据                                                       
     void write(Object, ChannelPromise)                         | 向缓冲区中写入数据，并通知指定的ChannelPromise                                              
     void flush()                                               | 输出调用write方法写入的缓冲区数据                                                         
     ChannelPromise voidPromise()                               | 返回这个Channel的特殊ChannelPromise(Void Promise)，用于将ChannelPromise作为参数但不希望得到通知的操作。
     ChannelOutboundBuffer outboundBuffer()                     | 获取出站缓冲区                                                                     

对于NioServerSocketChannel，其newUnsafe方法实现在父类AbstractNioMessageChannel中：

 
```
@Override
protected AbstractNioUnsafe newUnsafe() {
    return new NioMessageUnsafe();
}

```
 NioMessageUnsafe继承了AbstractNioUnsafe，重写了read方法，用于处理OP_ACCPET事件。关于事件处理细节可以参考我的博客：[https://blog.csdn.net/abc123lzf/article/details/83313530](https://blog.csdn.net/abc123lzf/article/details/83313530)

 newChannelPipeline方法则是比较简单，直接构造一个DefaultChannelPipeline就完了。

 回到ServerBootstrap：当initAndRegister方法执行结束后，init方法随即会被调用并传入刚才构造的ServerSocketChannel实例作为参数：

 
```
@Override
void init(Channel channel) throws Exception {
	//将通过option方法指定参数加入到Map中
    final Map<ChannelOption<?>, Object> options = options();
    synchronized (options) {
        setChannelOptions(channel, options, logger);
    }
    //将初始化属性传入到Map中
    final Map<AttributeKey<?>, Object> attrs = attrs();
    synchronized (attrs) {
        for (Entry<AttributeKey<?>, Object> e: attrs.entrySet()) {
            @SuppressWarnings("unchecked")
            AttributeKey<Object> key = (AttributeKey<Object>) e.getKey();
            channel.attr(key).set(e.getValue());
        }
    }
    ChannelPipeline p = channel.pipeline();
    //获取workGroup、childGroup指定的ChannelHandler
    final EventLoopGroup currentChildGroup = childGroup;
    final ChannelHandler currentChildHandler = childHandler;
    final Entry<ChannelOption<?>, Object>[] currentChildOptions;
    final Entry<AttributeKey<?>, Object>[] currentChildAttrs;
    synchronized (childOptions) {
        currentChildOptions = childOptions.entrySet().toArray(newOptionArray(childOptions.size()));
    }
    synchronized (childAttrs) {
        currentChildAttrs = childAttrs.entrySet().toArray(newAttrArray(childAttrs.size()));
    }
	//向管道尾部添加一个ChannelInitializer实例
    p.addLast(new ChannelInitializer<Channel>() {
        @Override
        public void initChannel(final Channel ch) throws Exception {
            final ChannelPipeline pipeline = ch.pipeline();
            ChannelHandler handler = handler();
            //添加用户自定义的bossGroup的ChannelHandler
            if (handler != null)
                pipeline.addLast(handler);
			//向管道尾部添加一个ServerBootstrapAcceptor实例
            ch.eventLoop().execute(new Runnable() {
                @Override
                public void run() {
                    pipeline.addLast(new ServerBootstrapAcceptor(ch, currentChildGroup, 
                    		currentChildHandler, currentChildOptions, currentChildAttrs));
                }
            });
        }
    });
}

```
 init方法将引导过程通过options方法设置的参数导入到NioServerSocketChannel的ChannelConfig中，然后再将初始属性导入到NioServerSocketChannel间接父类DefaultAttributeMap中。

 完成上述步骤后，向ServerSocketChannel所属的管道尾部添加一个ChannelInitializer实例，它重写了initChannel方法，并指定向管道添加用户自定义的ChannelHandler（通过handler方法指定的），然后再异步添加一个ServerBootstrapAcceptor实例，用于将接收到的客户端连接对应的NioSocketChannel转交给workGroup进行管理。

 前几篇博客我们已经提到过，ChannelInitializer在完成自己的工作（执行完initChannel方法）后会将自己从管道中移除。但是就目前而言还仅仅只是完成了管道初始化的第一步骤，因为ChannelInitializer的handlerAdd方法还尚未调用。

 init方法执行完毕后，就会将这个ServerSocketChannel注册到bossGroup中，最终是通过AbstractUnsafe的register方法进行注册：

 
```
@Override
public final void register(EventLoop eventLoop, final ChannelPromise promise) {
    if (eventLoop == null)
        throw new NullPointerException("eventLoop");
    if (isRegistered()) { //不允许重复注册或者同一个Channel注册到不同的EventLoop中
        promise.setFailure(new IllegalStateException("registered to an event loop already"));
        return;
    }
    if (!isCompatible(eventLoop)) { //如果不能注册到这种类型的EventLoop中
        promise.setFailure(new IllegalStateException("incompatible event loop type: " +
        		eventLoop.getClass().getName()));
        return;
    }
    AbstractChannel.this.eventLoop = eventLoop;
    //如果当前线程就是eventLoop所属的线程，那么直接执行register0方法，否则异步执行
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
            logger.warn("Force-closing a channel whose registration task was not accepted by an event loop: {}",
                    AbstractChannel.this, t);
            closeForcibly();
            closeFuture.setClosed();
            safeSetFailure(promise, t);
        }
    }
}

```
 register方法在进行一系列参数检查和状态检查后，继而会执行register0方法：

 
```
private void register0(ChannelPromise promise) {
    try {
        if (!promise.setUncancellable() || !ensureOpen(promise))
            return;
        boolean firstRegistration = neverRegistered; //默认为true
        doRegister(); //注册到Selector中
        neverRegistered = false;
        registered = true;
        pipeline.invokeHandlerAddedIfNeeded();
        safeSetSuccess(promise);
        //向管道发出Channel注册事件
        pipeline.fireChannelRegistered();
        if (isActive()) { //如果Channel是可用的
            if (firstRegistration) {
            	//向管道发出Channel可用事件
                pipeline.fireChannelActive();
            } else if (config().isAutoRead()) {
                beginRead();
            }
        }
    } catch (Throwable t) {
        closeForcibly();
        closeFuture.setClosed();
        safeSetFailure(promise, t);
    }
}

```
 register0方法首先调用doRegister进行注册：

 
```
@Override
protected void doRegister() throws Exception {
    boolean selected = false;
    for (;;) {
        try {
            selectionKey = javaChannel().register(eventLoop().unwrappedSelector(), 0, this);
            return;
        } catch (CancelledKeyException e) {
            if (!selected) {
                eventLoop().selectNow();
                selected = true;
            } else {
                throw e;
            }
        }
    }
}

```
 doRegister将ServerSocketChannel注册到了EventLoop的Selector上，并将附加对象设为this。doRegister方法执行结束后，就可以认为Channel已经注册到了EventLoop中，因为现在这个Channel持有的JDK Channel已经被EventLoop持有的Selector管理。  
 随后，就会调用管道对象的invokeHandlerAddedIfNeeded方法：

 
```
final void invokeHandlerAddedIfNeeded() {
    assert channel.eventLoop().inEventLoop();
    if (firstRegistration) {
        firstRegistration = false;
        callHandlerAddedForAllHandlers();
    }
}

```
 invokeHandlerAddedIfNeeded此时就会调用callHandlerAddedForAllHandlers执行回调任务。

 让我们回顾下服务端的引导过程，当调用addLast方法将ChannelInitializer添加到管道中后，如果管道检测到这是它第一次调用addLast方法，并不会马上执行ChannelInitializer的handlerAdd方法去执行我们重写的方法，而是注册了一个回调任务，并把这个回调任务添加到了DefaultChannelPipeline的成员变量pendingHandlerCallbackHead中，如果有多个回调任务，那么它可以通过它的成员变量next去维护一个单向链表。

 了解这些后，我们再来看callHandlerAddedForAllHandlers这个方法的实现：

 
```
private void callHandlerAddedForAllHandlers() {
    final PendingHandlerCallback pendingHandlerCallbackHead;
    synchronized (this) {
        assert !registered;
        registered = true; 
        pendingHandlerCallbackHead = this.pendingHandlerCallbackHead;
        this.pendingHandlerCallbackHead = null;
    }
    //遍历该链表，依次执行回调任务
    PendingHandlerCallback task = pendingHandlerCallbackHead;
    while (task != null) {
        task.execute();
        task = task.next;
    }
}

```
 这样，我们指定的ChannelInitializer的handlerAdd方法随即会被调用，管道初始化的第二步骤也就随之完成。

 回到register0方法：  
 接下来会依次向管道传递ChannelRegister、ChannelActive事件。传递完成后，NioServerSocketChannel开始进入端口绑定过程。  
 回到ServerBootstrap的doBind方法，doBind方法会开始进行端口的绑定过程，最终会调用到NioServerSocketChannel的doBind方法：

 
```
@Override
protected void doBind(SocketAddress localAddress) throws Exception {
    if (PlatformDependent.javaVersion() >= 7) { //如果JDK版本在1.7以上
        javaChannel().bind(localAddress, config.getBacklog());
    } else {
        javaChannel().socket().bind(localAddress, config.getBacklog());
    }
}

```
 这里直接调用到了JDK ServerSocketChannel的bind方法直接绑定指定的端口，并指定了最大连接数（默认为128），可以通过调用options并传入ChannelOption.SO_BACKLOG，然后指定一个最大连接数。当连接数超出后，客户端的连接请求会被阻塞。

 至此，NioServerSocketChannel初始化过程完成。

 
--------
 
### []()三、事件处理

 初始化过程完成后，此时的NioServerSocketChannel已经被EventLoop的Selector所管理，Selector则由EventLoop所属的线程进行轮询。这个线程运行NioEventLoop的run方法，通过一个无限循环不停地处理IO事件和一般任务。

 当有客户端发起连接时，ServerSocketChannel会发出OP_ACCEPT事件，就会通过Unsafe的read方法（实现在AbstractNioMessageUnsafe）处理这个事件：

 
```
@Override
public void read() {
    assert eventLoop().inEventLoop();
    final ChannelConfig config = config();
    
    if (!config.isAutoRead() && !isReadPending()) {
        // ChannelConfig.setAutoRead(false) was called in the meantime
    	//从事件集中移除这个事件
        removeReadOp();
        return;
    }
    //获取每个循环读取消息的最大字节数
    final int maxMessagesPerRead = config.getMaxMessagesPerRead();
    final ChannelPipeline pipeline = pipeline();
    boolean closed = false;
    Throwable exception = null;
    try {
        try {
            for (;;) {
            	//将数据读到readBuf中，返回读取到的字节数
                int localRead = doReadMessages(readBuf);
                if (localRead == 0)
                    break;
                if (localRead < 0) {
                    closed = true;
                    break;
                }
                if (!config.isAutoRead())
                    break;
                //如果readBuf的长度大于maxMessagesPerRead，退出循环
                if (readBuf.size() >= maxMessagesPerRead)
                    break;
            }
        } catch (Throwable t) {
            exception = t;
        }
        setReadPending(false);
        //获取读取到的ByteBuf数量，并通过循环将这些ByteBuf传递给ChannelInboundHandler
        int size = readBuf.size();
        for (int i = 0; i < size; i ++) {
            pipeline.fireChannelRead(readBuf.get(i));
        }
        //清除这些读取到的ByteBuf，并调用ChannelInboundHandler的fireChannelReadComplete方法
        readBuf.clear();
        pipeline.fireChannelReadComplete();

        //如果发生异常，调用ChannelInboundHandler的fireExceptionCaught方法
        if (exception != null) {
            closed = closeOnReadError(exception);
            pipeline.fireExceptionCaught(exception);
        }

        if (closed)
            if (isOpen())
                close(voidPromise());
    } finally {
    	//再次检查这个事件有没有从事件集中去除
        if (!config.isAutoRead() && !isReadPending()) {
            removeReadOp();
        }
    }
}

```
 关于NioEventLoop事件处理细节可以参考我的博客：[https://blog.csdn.net/abc123lzf/article/details/83313530](https://blog.csdn.net/abc123lzf/article/details/83313530) ，这里就不再赘述了。

   
  