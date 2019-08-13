---
title: Netty4.0源码分析：服务端启动过程
date: 2018-10-21 18:38:31
tags: CSDN迁移
---
 版权声明：尊重原创，转载请注明出处 [ https://blog.csdn.net/abc123lzf/article/details/83217332]( https://blog.csdn.net/abc123lzf/article/details/83217332)   
  ### []()一、引言

 服务端启动和客户端启动大致上是相同的，不过服务端一般需要绑定两个EventLoopGroup，一个EventLoopGroup专门负责接收连接，另外一个EventLoopGroup用来处理连接后的客户端的收发消息任务。我们称前者为bossGroup，后者为workGroup/childGroup。  
 这篇文章会省略和客户端启动过程相似的地方，在阅读前，需要先了解客户端的启动过程：  
 [https://blog.csdn.net/abc123lzf/article/details/83152303](https://blog.csdn.net/abc123lzf/article/details/83152303)

 我们以下面代码为例进行讲解：

 
```
public static void main(String[] args) throws InterruptedException {
	ServerBootstrap boot = new ServerBootstrap();
	boot.group(new NioEventLoopGroup(), new NioEventLoopGroup())
		.channel(NioServerSocketChannel.class)
		.option(ChannelOption.SO_TIMEOUT, 5000)
		.handler(new LoggingHandler(LogLevel.INFO))
		.childHandler(new ChannelInitializer<SocketChannel>() {
			@Override
			protected void initChannel(SocketChannel ch) throws Exception {
				ChannelPipeline pipeline = ch.pipeline();
				pipeline.addLast(new HttpResponseEncoder());
				pipeline.addLast(new HttpRequestDecoder());
			}
		});	
	ChannelFuture future = boot.bind(8848).sync();
	future.channel().closeFuture().sync();
}

```
 
### []()二、NioServerSocketChannel的构造过程

 NioServerSocketChannel和客户端的NioSocketChanel关系图：  
 ![在这里插入图片描述](https://img-blog.csdn.net/20181021114100912?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FiYzEyM2x6Zg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)  
 可以看出NioServerSocketChannel和NioSocketChannel都继承了AbstractNioChannel、实现了Channel接口。

 我们来看NioServerSocketChannel的构造方法：

 
```
private static final SelectorProvider DEFAULT_SELECTOR_PROVIDER = SelectorProvider.provider();
public NioServerSocketChannel() {
	this(newSocket(DEFAULT_SELECTOR_PROVIDER));
}

public NioServerSocketChannel(SelectorProvider provider) {
	this(newSocket(provider));
}

public NioServerSocketChannel(ServerSocketChannel channel) {
	super(null, channel, SelectionKey.OP_ACCEPT);
    config = new NioServerSocketChannelConfig(this, javaChannel().socket());
}

private static ServerSocketChannel newSocket(SelectorProvider provider) {
    try {
        return provider.openServerSocketChannel();
    } catch (IOException e) {
        throw new ChannelException("Failed to open a server socket.", e);
    }
}

```
 其构造过程和NioSocketChannel很相似，都会通过默认或指定的SelectorProvider构造一个Channel，只不过NioSocketChannel构造的是SocketChannel，NioServerSocketChannel构造的是ServerSocketChannel，然后调用父类AbstractNioMessageChannel的构造方法，并将SelectionKey.OP_ACCEPT作为参数readInterestOp，表示关注连接事件。最后将this对象和Channel对象封装为一个SocketChannelConfig对象。

 父类AbstractNioMessageChannel的构造方法：

 
```
protected AbstractNioMessageChannel(Channel parent, SelectableChannel ch, int readInterestOp) {
	super(parent, ch, readInterestOp);
}

```
 这里也只是调用了父类AbstractNioChannel的构造方法，和客户端的NioSocketChannel类似。区别在于在AbstractChannel的构造方法调用newUnsafe方法时，AbstractNioMessageChannel重写了自己的newUnsafe方法并构造了自己的Unsafe实现类：AbstractNioMessageChannel.NioMessageUnsafe。NioMessageUnsafe实现服务端底层的NIO读写策略，其实现原理我们后续再讨论。

 
### []()三、ServerBootstrap的端口绑定过程

 bind方法有多个重载的方法，将其IP、端口等信息封装为一个SocketAddress然后调用bind(SocketAddress)方法

 
```
public ChannelFuture bind(SocketAddress localAddress) {
	validate();
    if (localAddress == null) {
        throw new NullPointerException("localAddress");
    }
    return doBind(localAddress);
}

```
 validate方法同样会检查是否绑定了至少一个NioEventLoopGroup并且指定了ServerSocketChannel的类型。

 这里的doBind方法实现在AbstractBootstrap类中：

 
```
private ChannelFuture doBind(final SocketAddress localAddress) {
	//构造一个NioServerSocketChannel实例并注册
    final ChannelFuture regFuture = initAndRegister();
    final Channel channel = regFuture.channel();
    if (regFuture.cause() != null) {
        return regFuture;
    }

    if (regFuture.isDone()) {
        ChannelPromise promise = channel.newPromise();
        doBind0(regFuture, channel, localAddress, promise);
        return promise;
    } else {
        final PendingRegistrationPromise promise = new PendingRegistrationPromise(channel);
        regFuture.addListener(new ChannelFutureListener() {
            @Override
            public void operationComplete(ChannelFuture future) throws Exception {
                Throwable cause = future.cause();
                if (cause != null) {
                    promise.setFailure(cause);
                } else {
                    promise.executor = channel.eventLoop();
                    doBind0(regFuture, channel, localAddress, promise);
                }
            }
        });
        return promise;
    }
}

```
 ServerBootstrap重写了initAndRegister中的init方法：

 
```
@Override
void init(Channel channel) throws Exception {
	//将bossGroup的连接参数设置到Channel中
    final Map<ChannelOption<?>, Object> options = options();
    synchronized (options) {
        setChannelOptions(channel, options, logger);
    }
    //将bossGroup的属性设置到Channel中
    final Map<AttributeKey<?>, Object> attrs = attrs();
    synchronized (attrs) {
        for (Entry<AttributeKey<?>, Object> e: attrs.entrySet()) {
            @SuppressWarnings("unchecked")
            AttributeKey<Object> key = (AttributeKey<Object>) e.getKey();
            channel.attr(key).set(e.getValue());
        }
    }
    ChannelPipeline p = channel.pipeline();
	//设置workGroup的连接参数和属性
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
	//添加一个ChannelInitializer子类到NioServerSocketChannel的管道尾部
    p.addLast(new ChannelInitializer<Channel>() {
        @Override
        public void initChannel(final Channel ch) throws Exception {
            final ChannelPipeline pipeline = ch.pipeline();
            //首先添加用户指定的ChannelHandler（以上面的例子为例，这里添加的是LoggingHandler）
            ChannelHandler handler = handler();
            if (handler != null) {
                pipeline.addLast(handler);
            }
			//异步添加一个ServerBootstrapAccptor
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
 ServerBootstrapAcceptor是ServerBootstrap的内部类， 它是bossGroup中Channel所属的管道中的最后一个进站处理器（除去TailContext）。当bossGroup处理完客户端的连接事件后，bossGroup会通过ServerBootstrapAcceptor将后续的客户端读写事件转交给workGroup处理。

 ServerBootstrapAcceptor构造方法：

 
```
ServerBootstrapAcceptor(final Channel channel, EventLoopGroup childGroup, ChannelHandler childHandler,
        Entry<ChannelOption<?>, Object>[] childOptions, Entry<AttributeKey<?>, Object>[] childAttrs) {
    this.childGroup = childGroup; //传入workGroup
    this.childHandler = childHandler;
    this.childOptions = childOptions;
    this.childAttrs = childAttrs;
    enableAutoReadTask = new Runnable() {
        @Override public void run() {
            channel.config().setAutoRead(true);
        }
    };
}

```
 ServerBootstrapAcceptor的channelRead方法：

 
```
@Override @SuppressWarnings("unchecked")
public void channelRead(ChannelHandlerContext ctx, Object msg) {
    final Channel child = (Channel) msg; 
    //将childHandler添加到Channel的管道尾部
    child.pipeline().addLast(childHandler);
    //初始化连接参数
    setChannelOptions(child, childOptions, logger);
    for (Entry<AttributeKey<?>, Object> e: childAttrs) {
        child.attr((AttributeKey<Object>) e.getKey()).set(e.getValue());
    }
	//向workGroup注册这个NioSocketChannel实例，并添加监听器监听后续连接
    try {
        childGroup.register(child).addListener(new ChannelFutureListener() {
            @Override
            public void operationComplete(ChannelFuture future) throws Exception {
                if (!future.isSuccess()) {
                    forceClose(child, future.cause());
                }
            }
        });
    } catch (Throwable t) {
        forceClose(child, t);
    }
}

```
 当一个客户端连接到服务器时，NIO底层会产生一个的OP_ACCEPT事件，然后NioEventLoop会通过processSelectedKey方法调用到NioMessageUnsafe类的read方法，read方法随即会调用NioServerSocketChannel的doReadMessage方法：

 
```
@Override
protected int doReadMessages(List<Object> buf) throws Exception {
    SocketChannel ch = SocketUtils.accept(javaChannel());
    try {
        if (ch != null) {
            buf.add(new NioSocketChannel(this, ch));
            return 1;
        }
    } catch (Throwable t) {
        //省略异常处理
    }
    return 0;
}

```
 doReadMessage方法会构造一个NioSocketChannel，并添加到NioMessageUnsafe实例的成员变量readBuf集合中。随后，read方法会将这个集合中所有的Channel发送到管道中，由管道中的进站处理器（ChannelInboundHandler）进行处理。

 init方法执行过程中，NioServerSocketChannel实例的管道变化情况：  
 addLast方法的addLast0方法时：  
 ![在这里插入图片描述](https://img-blog.csdn.net/20181021163730224?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FiYzEyM2x6Zg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)  
 addLast方法的callHandlerAdded0方法被调用时：  
 ![在这里插入图片描述](https://img-blog.csdn.net/20181021163946422?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FiYzEyM2x6Zg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)  
 随后，ChannelInitializer内部的方法会将this从管道中移除：  
 ![在这里插入图片描述](https://img-blog.csdn.net/20181021164153579?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FiYzEyM2x6Zg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

 对于workGroup的Channel的管道添加操作，也是一样的。不过在服务端中，workGroup的Channel的管道添加操作只会在建立连接的时候创建了NioSocketChannel的时候才会将我们通过ServerBootstrap的childHandler指定的ChannelInitializer添加到它的管道中。

 回到bind方法：  
 当initAndRegister方法执行结束后，就会调用doBind0方法：

 
```
private static void doBind0(final ChannelFuture regFuture, final Channel channel,
        final SocketAddress localAddress, final ChannelPromise promise) {
    channel.eventLoop().execute(new Runnable() {
        @Override
        public void run() {
            if (regFuture.isSuccess()) {
                channel.bind(localAddress, promise).addListener(ChannelFutureListener.CLOSE_ON_FAILURE);
            } else {
                promise.setFailure(regFuture.cause());
            }
        }
    });
}

```
 doBind0方法异步调用NioServerSocketChannel的bind方法进行绑定端口操作：

 
```
@Override
public ChannelFuture bind(SocketAddress localAddress, ChannelPromise promise) {
	return pipeline.bind(localAddress, promise);
}

```
 这里和客户端一样将绑定操作委托给它持有的管道：

 
```
@Override
public final ChannelFuture bind(SocketAddress localAddress, ChannelPromise promise) {
	return tail.bind(localAddress, promise);
}

```
 TailContext没有重写bind的方法，所以这里调用的是AbstractChannelHandlerContext的bind方法：

 
```
@Override
public ChannelFuture bind(final SocketAddress localAddress, final ChannelPromise promise) {
	//省略参数、状态检查代码
	//找到下一个出站处理器，直到HeadContext
    final AbstractChannelHandlerContext next = findContextOutbound();
    EventExecutor executor = next.executor();
    if (executor.inEventLoop()) {
        next.invokeBind(localAddress, promise);
    } else {
        safeExecute(executor, new Runnable() {
            @Override
            public void run() {
                next.invokeBind(localAddress, promise);
            }
        }, promise, null);
    }
    return promise;
}

```
 这里会遍历所有实现ChannelOutboundHandler接口的ChannelHandler并调用它们的bind方法，一直调用到HeadContext（也实现了这个接口）的bind方法：

 
```
@Override
public void bind(ChannelHandlerContext ctx, SocketAddress localAddress, 
		ChannelPromise promise)throws Exception {
	unsafe.bind(localAddress, promise);
}

```
 AbstractUnsafe的bind方法实现：

 
```
@Override
public final void bind(final SocketAddress localAddress, final ChannelPromise promise) {
    //省略参数、状态检查代码
    boolean wasActive = isActive();
    try {
        doBind(localAddress);
    } catch (Throwable t) {
        //省略异常处理
        return;
    }
	//通知管道中的ChannelHandler
    if (!wasActive && isActive()) {
        invokeLater(new Runnable() {
            @Override
            public void run() {
                pipeline.fireChannelActive();
            }
        });
    }
    safeSetSuccess(promise);
}

```
 最终调用的是NioServerSocketChannel中的doBind方法：

 
```
@Override
protected void doBind(SocketAddress localAddress) throws Exception {
	if (PlatformDependent.javaVersion() >= 7) {
		javaChannel().bind(localAddress, config.getBacklog());
        } else {
            javaChannel().socket().bind(localAddress, config.getBacklog());
        }
    }

```
 doBind方法通过调用JDK NIO包中的SocketChannel的bind方法来实现端口的绑定。

 **如有错误或建议，欢迎评论**

   
  