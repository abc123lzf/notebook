---
title: Netty4.0源码解析：管道模型ChannelPipeline
date: 2018-10-27 14:48:34
tags: CSDN迁移
---
 版权声明：尊重原创，转载请注明出处 [ https://blog.csdn.net/abc123lzf/article/details/83412636]( https://blog.csdn.net/abc123lzf/article/details/83412636)   
  ### []()一、引言

 在Netty中每个 `Channel` 都持有一个 `ChannelPipeline` ， `ChannelPipeline` 持有多个 `ChannelHandler` 用于处理IO事件，这些 `ChannelHandler` 的顺序由一个双向链表维护，双向链表的头尾结点为 `HeadContext` 的实例和 `TailContext` 实例。

 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20181026120049209.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FiYzEyM2x6Zg==,size_27,color_FFFFFF,t_70)  
 双向链表由 `ChannelHandler` 持有的 `ChannelHandlerContext` 绑定。

 `EventLoopGroup` 、 `EventLoop` 、 `Channel` 、 `ChannelPipeline` 、 `ChannelHandler` 、 `ChannelHandlerContext` 的关系图：  
 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20181026120750919.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FiYzEyM2x6Zg==,size_27,color_FFFFFF,t_70)

 `ChannelPipeline` 、 `ChannelHandlerContext` 继承关系图：  
 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20181026123203503.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FiYzEyM2x6Zg==,size_27,color_FFFFFF,t_70)

 `ChannelPipeline` 的唯一实现类是 `DefaultChannelPipeline` ，所以我们从 `DefaultChannelPipeline` 入手 `ChannelPipeline` 的处理流程

 
### []()二、源码解析

 
##### []()1、管道的构造

 `DefaultChannelPipeline` 的构造在 `Channel` （ `AbstractChannel` ）的构造阶段就已经完成：

 
```
protected DefaultChannelPipeline(Channel channel) {
	this.channel = ObjectUtil.checkNotNull(channel, "channel");
    //实例化头结点和尾结点
    tail = new TailContext(this);
    head = new HeadContext(this);
    //将头、尾结点组成一个双向链表
    head.next = tail;
    tail.prev = head;
}

```
 `DefaultChannelPipeline` 构造了两个对象： `HeadContext` 对象和 `TailContext` 对象，从上图我们可以看到， `HeadContext` 实现了 `ChannelHandlerContext` 、 `ChannelInboundHandler` 、 `ChannelOutboundHandler` 接口，也就是说， `HeadContext` 实现了三个角色：作为**链表的结点**、作为**进站处理器**和**出站处理器**。 `TailContext` 实现了两个接口： `ChannelHandlerContext` 和 `ChannelInboundHandler` ，也就是作为链表结点和进站处理器。

 `HeadContext` 构造方法如下：

 
```
HeadContext(DefaultChannelPipeline pipeline) {
	super(pipeline, null, HEAD_NAME, false, true);
    unsafe = pipeline.channel().unsafe();
    setAddComplete();
}

```
 `HeadContext` 构造阶段首先调用了父类 `AbstractChannelHandlerContext` 的构造方法：

 
```
AbstractChannelHandlerContext(DefaultChannelPipeline pipeline, EventExecutor executor, 
							String name, boolean inbound, boolean outbound) {
	this.name = ObjectUtil.checkNotNull(name, "name");
    this.pipeline = pipeline; //指定为这个HandContext的ChannelPipeline
    this.executor = executor; //指定为null
    this.inbound = inbound; //指定为false
    this.outbound = outbound; //指定为true
    ordered = executor == null || executor instanceof OrderedEventExecutor; //为true
}

```
 然后将这个管道所属的 `Channel` 的 `Unsafe` 实例引用添加到当前的 `HeadContext` 的 `unsafe` 变量中，然后调用 `setAddComplete` 方法将 `handlerState` （代表 `ChannelHandler` 的状态）改为 `ADD_COMPLETE` 状态。  
  `HeadContext` 持有 `Channel` 的 `Unsafe` 实例是因为需要在服务端引导过程即将完成时，调用 `Unsafe` 的 `bind` 方法绑定端口。

 再来看 `TailContext` 的构造方法：

 
```
TailContext(DefaultChannelPipeline pipeline) {
	super(pipeline, null, TAIL_NAME, true, false);
    setAddComplete();
}

```
 同样调用了父类 `AbstractChannelHandlerContext` 的构造方法，并指定其 `inbound` 为 `true` ， `outbound` 为 `false` 。  
 然后同样调用了 `setAddComplete` 方法。

 
##### []()2、向管道添加ChannelHandler

 向管道添加 `ChannelHandle` r可以通过调用 `addFirst` 、 `addLast` 、 `addBefore` 、 `addAfter` 方法添加，但最常使用的还是 `addLast` 方法，所以我们从 `addLast` 方法入手：

 `addLast` 方法可以向管道添加 `ChannelHandler` ，它有4个重载的方法：

 
```
public final ChannelPipeline addLast(ChannelHandler... handlers);
public final ChannelPipeline addLast(EventExecutorGroup executor, ChannelHandler... handlers);
public final ChannelPipeline addLast(String name, ChannelHandler handler);
public final ChannelPipeline addLast(EventExecutorGroup group, String name, ChannelHandler handler);

```
 我们先从第一个重载的方法开始分析：

 
```
@Override
public final ChannelPipeline addLast(ChannelHandler... handlers) {
	return addLast(null, handlers);
}

@Override
public final ChannelPipeline addLast(EventExecutorGroup executor, ChannelHandler... handlers) {
    if (handlers == null)
        throw new NullPointerException("handlers");
    for (ChannelHandler h: handlers) {
        if (h == null)
            break;
        addLast(executor, null, h);
    }
    return this;
}

```
 首先遍历 `handlers` 数组，对于非 `null` 元素，调用第4个重载的 `addLast` 方法添加到管道：

 
```
@Override
public final ChannelPipeline addLast(EventExecutorGroup group, String name, ChannelHandler handler) {
    final AbstractChannelHandlerContext newCtx;
    synchronized (this) { //保证添加管道的线程安全性
        checkMultiplicity(handler); //检查这个Handler实例能否被重复添加到管道中
        //构造一个DefaultChannelHandlerContext实例并返回
        newCtx = newContext(group, filterName(name, handler), handler);
        addLast0(newCtx); //添加到管道TailContext的前一个结点
        //如果本次添加ChannelHandler的操作是第一次进行的(说明在引导阶段)，那么registered为false
        if (!registered) {
            newCtx.setAddPending(); //将handlerState改为ADD_PENDING
            //注册一个回调任务
            callHandlerCallbackLater(newCtx, true);
            return this;
        }
		//获取这个管道所属的Channel对应的EventExecutor(EventLoop)
        EventExecutor executor = newCtx.executor();
        //如果当前线程不是这个executor所属的线程，那么向这个EventLoop提交任务异步执行callHandlerAdded0
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
    //到这里则说明当前线程就是executor所属的线程，直接调用callHandlerAdded0方法
    callHandlerAdded0(newCtx);
    return this;
}

```
 这个方法的步骤可以分为以下几点：  
 1、首先检查这个管道能否被重复添加，可以被重复添加的前提是这个 `ChannelHandler` 类加上了 `@Sharable` 注解。如果不能重复添加则抛出异常，如果可以添加则将这个 `ChannelHandler` 实例标记为添加过。

 
```
private static void checkMultiplicity(ChannelHandler handler) {
    if (handler instanceof ChannelHandlerAdapter) {
        ChannelHandlerAdapter h = (ChannelHandlerAdapter) handler;
        if (!h.isSharable() && h.added) {
            throw new ChannelPipelineException(h.getClass().getName() +
                    " is not a @Sharable handler, so can't be added or removed multiple times.");
        }
        h.added = true;
    }
}

```
 2、调用 `newContext` 方法构造一个 `DefaultChannelHandlerContext` ，并将需要添加的 `ChannelHandler` 添加到里面：

 
```
private AbstractChannelHandlerContext newContext(EventExecutorGroup group, String name, ChannelHandler handler) {
	return new DefaultChannelHandlerContext(this, childExecutor(group), name, handler);
}

```
 3、调用 `addLast0` 方法将这个 `ChannelHandlerContext` 添加到管道持有的双向链表的 `TailContext` 实例的前面。  
 4、如果这是第一次调用这个管道对象的 `addLast` 方法，那么会向这个管道注册一个回调接口，然后方法返回结束。

 
```
private void callHandlerCallbackLater(AbstractChannelHandlerContext ctx, boolean added) {
    assert !registered;
    PendingHandlerCallback task = added ? new PendingHandlerAddedTask(ctx) : new PendingHandlerRemovedTask(ctx);
    PendingHandlerCallback pending = pendingHandlerCallbackHead;
    if (pending == null) {
        pendingHandlerCallbackHead = task;
    } else {
        // Find the tail of the linked-list.
        while (pending.next != null) {
            pending = pending.next;
        }
        pending.next = task;
    }
}

```
 这里会向 `ChannelPipeline` 的回调对象链表（变量 `pendingHandlerCallbackHead` ）添加一个 `PendingHandlerAddedTask` 实例：

 
```
private final class PendingHandlerAddedTask extends PendingHandlerCallback {
	PendingHandlerAddedTask(AbstractChannelHandlerContext ctx) {
        super(ctx);
    }
    @Override public void run() {
        callHandlerAdded0(ctx);
    }
    //省略execute方法
}

```
 这个回调接口会在 `Channel` 注册到 `NioEventLoopGroup` 的时候执行。具体在 `AbstractUnsafe` 类的 `register0` 方法， `register0` 方法调用了管道对象的 `invokeHandlerAddedIfNeeded` 方法，在方法执行过程中就会执行所有注册的回调任务。

 5、如果当前线程不是这个 `ChannelPipeline` 对应的 `Channel` 所属的线程，那么会向这个 `Channel` 所属的线程对应的 `EventLoop` 提交一个任务，来达到异步执行 `callHandlerAdded0` 方法的目的。如果正好就是当前线程，那么直接调用 `callHandlerAdded0` 方法。

 
```
private void callHandlerAdded0(final AbstractChannelHandlerContext ctx) {
    try {
        ctx.setAddComplete(); //状态设为ADD_COMPLETE
        ctx.handler().handlerAdded(ctx); //向ChannelHandler发出Handler添加事件
    } catch (Throwable t) { //如果发生任何异常
        boolean removed = false;
        try {
            remove0(ctx); //从链表中移除这个ChannelHandlerContext
            try {
                ctx.handler().handlerRemoved(ctx); //移除这个ChannelHandler
            } finally {
                ctx.setRemoved(); //handlerState改为REMOVE_COMPLETE
            }
            removed = true;
        } catch (Throwable t2) {
            if (logger.isWarnEnabled())
                logger.warn("Failed to remove a handler: " + ctx.name(), t2);
        }
        //省略日志记录代码...
    }
}

```
 回顾一下服务端的引导过程，当我们调用bind方法后，服务端的启动过程随即展开。 `callHandlerAdded0` 方法随即会被调用，我们指定的初始 `ChannelHandler` 随即会被添加到 `ChannelPipeline` 中。但是我们一般都是调用 `handler/childHandler` 方法传入 `ChannelInitializer` 的子类，并重写它的 `initChannel` 方法，在这个方法中添加我们实际需要的管道对象。那么， `ChannelInitializer` 是如何实现添加我们自定义的管道呢？答案就在 `callHandlerAdded0` 方法的 `ctx.handler().handlerAdded(ctx)` 这段代码中：

 
```
@Override
public void handlerAdded(ChannelHandlerContext ctx) throws Exception {
	if (ctx.channel().isRegistered()) { //如果Channel已经注册到EventLoop
		initChannel(ctx);
	}
}
@SuppressWarnings("unchecked")
private boolean initChannel(ChannelHandlerContext ctx) throws Exception {
    if (initMap.putIfAbsent(ctx, Boolean.TRUE) == null) { // Guard against re-entrance.
        try {
            initChannel((C) ctx.channel()); //调用我们重写的方法
        } catch (Throwable cause) {
            exceptionCaught(ctx, cause);
        } finally { //最终将this从ChannelPipeline链表中移除
            remove(ctx);
        }
        return true;
    }
    return false;
}

```
 该方法首先执行我们重写的 `initChannel` 方法，执行完成后将这个 `ChannelInitializer` 实例从当前的 `ChannelPipeline` 中移除。如果在调用我们重写的方法过程中发生未捕获异常，那么会调用 `exceptionCaught` 方法将Channel关闭。

 
##### []()3、管道事件的传输

 管道事件的传输离不开 `ChannelHandlerContext` 。从字面意思上可以看出， `ChannelHandlerContext` 是 `ChannelHandler` 的上下文对象，它将 `ChannelPipeline` 和 `ChannelHandler` 关联起来，并提供一系列接口操作 `Channel` 和 `ChannelHandler` ，传递管道事件等一系列操作。

 在上篇文章中我们已经提到过（ [https://blog.csdn.net/abc123lzf/article/details/83313530](https://blog.csdn.net/abc123lzf/article/details/83313530) ）：当 `EventLoop` 持有的 `Channel` 通道接收到OP_ACCEPT事件（仅限 `bossGroup` 中的 `ServerSocketChannel` ）或者 `OP_READ` 事件时，就会调用 `ChannelPipeline` 的 `fireChannelRead` 方法向管道传递读入事件，并将 `SocketChannel` ( `bossGroup` )或者 `ByteBuf` ( `workGroup` )传递到管道中:

 
```
@Override
public final ChannelPipeline fireChannelRead(Object msg) {
	AbstractChannelHandlerContext.invokeChannelRead(head, msg);
    return this;
}

```
 这里调用了 `AbstractChannelHandlerContext` 的静态方法 `invokeChannelRead` ，并将这个 `ChannelPipeline` 持有的双向链表的头结点 `HeadContext` 实例作为参数传入：

 
```
static void invokeChannelRead(final AbstractChannelHandlerContext next, final Object msg) {
    ObjectUtil.checkNotNull(msg, "msg");
    EventExecutor executor = next.executor();
    //如果当前线程和这个ChannelPipeline所属的EventLoop持有的线程相同，那么直接调用invokeChannelRead
    //否则提交任务异步执行
    if (executor.inEventLoop()) {
        next.invokeChannelRead(msg);
    } else {
        executor.execute(new Runnable() {
            @Override
            public void run() {
                next.invokeChannelRead(msg);
            }
        });
    }
}

private void invokeChannelRead(Object msg) {
    if (invokeHandler()) { //返回这个ChannelHandler是否为ADD_COMPLETE状态（即所有的Handler已经初始化完成）
        try {
            ((ChannelInboundHandler) handler()).channelRead(this, msg);
        } catch (Throwable t) {
            notifyHandlerException(t);
        }
    } else {
        fireChannelRead(msg);
    }
}

```
 `invokeChannelRead(Object)` 方法首先通过 `invokeHandler` 方法判断它持有的 `ChannelHandler` 是否为 `ADD_COMPLETE` 状态，如果是，就调用这个 `ChannelHandler(HeadContext)` 的 `channelRead` 方法传播读入事件：（下面是 `HeadContext` 的实现）

 
```
@Override
public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
	ctx.fireChannelRead(msg);
}

```
 每个 `ChannelHandler` 实例都可以重写这个 `channelRead` 方法，以实现自己的业务逻辑。  
  `HeadContext` 默认的 `channelRead` 实现则是直接将这个事件传入下一个 `ChannelHandler` ：

 
```
@Override
public ChannelHandlerContext fireChannelRead(final Object msg) {
    invokeChannelRead(findContextInbound(), msg);
    return this;
}

```
 首先是调用 `findContextInbound` 寻找链表的下一个进站处理器结点（ `ChannelInboundHandler` ）：

 
```
private AbstractChannelHandlerContext findContextInbound() {
    AbstractChannelHandlerContext ctx = this;
    do {
        ctx = ctx.next;
    } while (!ctx.inbound);
    return ctx;
}

```
 该方法很简单，就是遍历链表直到找到 `ChannelInboundHandler` 为止。

 
##### []()4、bossGroup的管道向workGroup管道的事件传递

 `bossGroup` 在处理 `OP_ACCPET` 事件时，调用 `channelRead` 在管道中传输的对象是 `ServerSocketChannel` 接收连接得到的 `SocketChannel` 而不是 `ByteBuf` 。 `bossGroup` 的职责就是将 `SocketChannel` 转发给 `workGroup` 处理。那么 `bossGroup` 是如何做到的呢？答案就在 `ServerBootstrap` 的内部类 `ServerBootstrapAcceptor` 。

 在 `bossGroup` 的 `ChannelPipeline` 中， `ServerBootstrapAcceptor` 位于 `ChannelPipeline` 的最后（除了 `TailContext` ），它的 `channelRead` 方法实现了将 `SocketChannel` 转发给 `workGroup` 管理：

 
```
@Override
@SuppressWarnings("unchecked")
public void channelRead(ChannelHandlerContext ctx, Object msg) {
    final Channel child = (Channel) msg; 
    //将我们通过ServerBootstrap的childHandler指定的ChannelHandler添加到child持有的管道
    child.pipeline().addLast(childHandler);
    //初始化连接参数
    setChannelOptions(child, childOptions, logger);
	//初始化属性
    for (Entry<AttributeKey<?>, Object> e: childAttrs) {
        child.attr((AttributeKey<Object>) e.getKey()).set(e.getValue());
    }
	//向childGroup注册这个SocketChannel，并添加一个监听器监听是否添加成功
    try {
        childGroup.register(child).addListener(new ChannelFutureListener() {
            @Override
            public void operationComplete(ChannelFuture future) throws Exception {
                if (!future.isSuccess()) { //如果添加失败则关闭这个SocketChannel
                    forceClose(child, future.cause());
                }
            }
        });
    } catch (Throwable t) {
        forceClose(child, t);
    }
}

```
 `ServerBootstrapAcceptor` 的 `channelRead` 方法主要功能是初始化这个 `SocketChannel` 的连接参数和属性，然后将其注册到 `childGroup` 中，由 `childGroup` 进行管理。然后向返回的 `ChannelFuture` 注册一个监听器，如果最终添加失败，那么会关闭这个 `SocketChannel` 。

   
  