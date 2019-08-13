---
title: 初步了解Netty核心组件：Channel、ChannelHandler和ChannelPipeline
date: 2018-06-02 16:34:22
tags: CSDN迁移
---
 版权声明：尊重原创，转载请注明出处 [ https://blog.csdn.net/abc123lzf/article/details/80547310]( https://blog.csdn.net/abc123lzf/article/details/80547310)   
  #### 1、Channel

 这里的Channel与JDK中的Channel有所区别，Netty的Channel在JDK的基础上进行了封装并赋予了更多的功能，用户可以使用Channel进行一下操作：

  
  * 查询Channel状态和配置Channel的参数 
  * 进行IO操作(read\write\connect\bind)  Channel接口大大降低了直接使用Socket类的复杂性，Channel的实现类有：   
 EmbeddedChannel、LocalServerChannel、NioDatagramChannel、NioSctpChannel、NioSocketChannel   
 在使用Channel时需要注意：   
 **（1）Channel中所有的IO操作都是异步的**   
 当我们调用一个IO操作时，会返回一个ChannelFuture对象，这意味着方法会立刻返回但是不代表操作已经完成。如果需要在操作完成后执行一系列的操作，那么我们需要通过ChannelFuture对象来判断。   
 例如：

 
```
ServerBootstrap boot = new ServerBootstrap();
//一系列引导操作...
ChannelFuture future = boot.bind(port).sync(); //绑定一个端口，并返回一个ChannelFuture对象
if (future.isSuccess()) { //如果绑定成功
    serverSocketChannel = (ServerSocketChannel) future.channel();
    System.out.println("服务器启动成功");
} else {
    System.out.println("服务器启动失败");
}
future.channel().closeFuture().sync();
```
 **（2）Channel之间是有关联的**   
 如果一个Channel由另一个Channel创建，那么它们之间存在着父子关系。例如调用ServerSocketChannel.accpet()方法接收到了SocketChannel对象，那么它的父Channel是ServerSocketChannel。   
 **（3）释放资源**   
 当一个Channel不再使用时，需要调用close方法来关闭Channel

 
#### 2、ChannelHandler

 ChannelHandler在Netty中本身是一个接口，它的实现类负责接收并响应事件通知（在Netty中，ChannelHandler中的方法是由网络事件触发的），所有的数据处理逻辑应包含在ChannelHander中。可以这么认为，ChannelHandler类似于Filter，它负责对入站或出站数据进行拦截、处理（例如将Socket传入二进制数据转换成字符串、文件、对象…）。   
 **下面是ChannelHandler的核心类类图：**   
 ![这里写图片描述](http://upload-images.jianshu.io/upload_images/3288959-54828cf366eae132.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)   
 图片转自：[https://blog.csdn.net/yexin94822739/article/details/73338646](https://blog.csdn.net/yexin94822739/article/details/73338646)   
 其实Netty不止包含了这些ChannelHandler，Netty也提供了大量的编码器、解码器，例如基本的二进制数据转换：ByteToMessageDecoder、MessageToMessageDecoder，对象转换处理器ObjectDecoder、ObjectEncoder，甚至是HTTP协议转换处理器：HttpRequestDecoder、HttpResponseDecoder等，它们都继承了上述核心类，并重写了它们的方法或额外添加了其它方法，负责将传入的数据转换成期望的类型，而不必再去重新写一系列麻烦的转换过程。这里不再赘述，只介绍最基本的进站、出站处理器。

 ChannelHandler是最顶层的接口，它包含最基本的方法：

 
```
public interface ChannelHandler {
    //ChannelHandler在被添加到ChannelPipeline时调用
    void handlerAdded(ChannelHandlerContext ctx) throws Exception;
    //ChannelHandler从ChannelPipeline删除时调用
    void handlerRemoved(ChannelHandlerContext ctx) throws Exception;
    //ChannelHandler处理过程中发生异常时调用（已过时）
    @Deprecated
    void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception;
    @Inherited
    @Documented
    @Target(ElementType.TYPE)
    @Retention(RetentionPolicy.RUNTIME)
    @interface Sharable {
    }
}
```
 可能大家注意到了这里有个@Sharable注解，Sharable注解用在ChannelHandler的实现类上，被注解的ChannelHandler可以被加入到一个或多个ChannelPipeline一次或多次，意味着如果你可以保证此ChannelHandler是线程安全的，那么可以使用此注解，否则不要使用该注解。

 ChannelInboundHandler负责接收入站事件、数据，这些数据进行一系列处理后即可被业务逻辑调用，你的业务逻辑应放在ChannelInboundHandler中。   
 ChannelOutboundHanlder负责处理出站事件。   
 下面是ChannelInboundHandler接口的源代码及其解析：

 
```
public interface ChannelInboundHandler extends ChannelHandler {
    //Channel注册到EventLoop完成之后调用（意味着只调用一次）
    void channelRegistered(ChannelHandlerContext ctx) throws Exception;
    //Channel从EventLoop取消注册时调用
    void channelUnregistered(ChannelHandlerContext ctx) throws Exception;
    //Channel被激活时调用，比如客户端接入服务器（对于一次客户端连接调用一次）
    void channelActive(ChannelHandlerContext ctx) throws Exception;
    //已注册的Channel到生命周期结束时调用
    void channelInactive(ChannelHandlerContext ctx) throws Exception;
    //Channel有可读事件时调用，msg为读到的对象（如果前面的进站处理器没有转换为其它类型，一般将msg视为ByteBuf类型）
    void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception;
    //可读事件处理完成后调用
    void channelReadComplete(ChannelHandlerContext ctx) throws Exception;
    //用户事件触发时调用（一般用的较少）
    void userEventTriggered(ChannelHandlerContext ctx, Object evt) throws Exception;
    //Channel可写入状态改变时调用（一般用的较少）
    void channelWritabilityChanged(ChannelHandlerContext ctx) throws Exception;
    //Channel发生异常时调用，cause为抛出的异常
    @Override
    @SuppressWarnings("deprecated")
    void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception;
}
```
 ChannelInboundHandlerAdapter和ChannelOutboundHanlderAdapter是Netty提供的基础进站、出站实现类。   
 下面是ChannelInboundHandlerAdapter的源代码

 
```
public class ChannelInboundHandlerAdapter extends ChannelHandlerAdapter implements ChannelInboundHandler {
    @Override
    public void channelRegistered(ChannelHandlerContext ctx) throws Exception {
        ctx.fireChannelRegistered();
    }
    @Override
    public void channelUnregistered(ChannelHandlerContext ctx) throws Exception {
        ctx.fireChannelUnregistered();
    }
    @Override
    public void channelActive(ChannelHandlerContext ctx) throws Exception {
        ctx.fireChannelActive();
    }
    @Override
    public void channelInactive(ChannelHandlerContext ctx) throws Exception {
        ctx.fireChannelInactive();
    }
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        ctx.fireChannelRead(msg);
    }
    @Override
    public void channelReadComplete(ChannelHandlerContext ctx) throws Exception {
        ctx.fireChannelReadComplete();
    }
    public void userEventTriggered(ChannelHandlerContext ctx, Object evt) throws Exception {
        ctx.fireUserEventTriggered(evt);
    }
    @Override
    public void channelWritabilityChanged(ChannelHandlerContext ctx) throws Exception {
        ctx.fireChannelWritabilityChanged();
    }
    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause)
            throws Exception {
        ctx.fireExceptionCaught(cause);
    }
}
```
 可以看出它们本身默认实现全部是将事件传递给下一个ChannelHandler（所以不要直接将它添加到ChannelPipeline中，没有任何意义），所以我们应当自定义一个处理类继承它并根据实际的业务需求重写它们的方法，例如：

 
```
public class MyInboundHandler extends ChannelInboundHandlerAdapter {
    @Override
    public void channelActive(ChannelHandlerContext ctx) {
        System.out.println("已收到客户端连接");
        //一系列业务逻辑
    }
    // 客户端与服务器断开时调用
    @Override
    public void channelInactive(ChannelHandlerContext ctx) {
        System.out.println("客户端会话关闭");
        //一系列关闭客户端会话的业务逻辑
    }
    @Override
    public void channelReadComplete(ChannelHandlerContext ctx) {
        ctx.flush();
    }
    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable t) {
        if (!(t instanceof IOException)) {
            t.printStackTrace();
        } else {
            System.out.println("客户端断开连接");
        }
        ctx.close();
    }
    // 用于读取客户端发送的请求
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
        System.out.println("已收到客户端请求");
        //执行一系列业务逻辑
    }
}
```
 **SimpleChannelInboundHandler抽象类**   
 SimpleChannelInboundHandler<I>一般用于客户端，I是你需要处理的Java类型   
 下面是源码：

 
```
public abstract class SimpleChannelInboundHandler<I> extends ChannelInboundHandlerAdapter {

    private final TypeParameterMatcher matcher;
    private final boolean autoRelease;

    protected SimpleChannelInboundHandler() {
        this(true);
    }

    protected SimpleChannelInboundHandler(boolean autoRelease) {
        matcher = TypeParameterMatcher.find(this, SimpleChannelInboundHandler.class, "I");
        this.autoRelease = autoRelease;
    }

    protected SimpleChannelInboundHandler(Class<? extends I> inboundMessageType) {
        this(inboundMessageType, true);
    }

    protected SimpleChannelInboundHandler(Class<? extends I> inboundMessageType, boolean autoRelease) {
        matcher = TypeParameterMatcher.get(inboundMessageType);
        this.autoRelease = autoRelease;
    }

    public boolean acceptInboundMessage(Object msg) throws Exception {
        return matcher.match(msg);
    }

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        boolean release = true;
        try {
            //判断是否是期望的类型(即指定的泛型参数I)
            if (acceptInboundMessage(msg)) {
                @SuppressWarnings("unchecked")
                I imsg = (I) msg;
                channelRead0(ctx, imsg);
            } else {
                release = false;
                //传入下一个处理器
                ctx.fireChannelRead(msg);
            }
        } finally {
            if (autoRelease && release) {
                ReferenceCountUtil.release(msg);
            }
        }
    }

    protected abstract void channelRead0(ChannelHandlerContext ctx, I msg) throws Exception;
}
```
 继承这个类后，我们只需要实现channelRead0方法即可，当channelRead真正被调用的时候channelRead0中的代码才会被执行。   
 从源代码可以看出，这里的channelRead方法主要是识别传入的msg是否是期望的类型，如果是，则执行channelRead0方法，如果不是，则传入下一个进站处理器，并且如果msg失去了引用，那么会自动释放资源。

 
#### 3、ChannelPipeline

 ChannelHandler的容器是ChannelPipeline，它负责将ChannelHandler按照添加顺序保存在一个双向链表中，当有进站或出站事件时，它会保证按照ChannelHandler添加顺序依次拦截、处理或者包装。   
 **下面是ChannelPipeline结构图**   
 ![这里写图片描述](https://timgsa.baidu.com/timg?image&amp;quality=80&amp;size=b9999_10000&amp;sec=1527928915916&amp;di=50d0c9af9d3c1783a58937a062f0edd8&amp;imgtype=jpg&amp;src=http://img2.imgtn.bdimg.com/it/u=4118467692,419380513&amp;fm=214&amp;gp=0.jpg)   
 使事件流经ChannelPipeline是ChannelHandler的工作，ChannelPipeline维护了一个双向链表，使得它的执行顺序是由ChannelHandler被添加的顺序决定的。   
 每一个Channel控制一个ChannelPipeline，每个ChannelHandler对应一个ChannelHandlerContext，ChannelHandlerContext接口代表了ChannelHandler和ChannelPipeline之间的绑定，它保存了ChannelHandler所属的Channel、ChannelPipeline、下一个ChannelHandler等一系列配置，也可以执行关闭Channel、断开远程节点等一系列操作，并且最重要的一点是它可以用于写出站数据，当你调用writeAndFlush()方法写入数据时，它会将数据放到最后一个出站处理器，并经过一系列你定义的出站处理器发送给远程主机。

 **将ChannelHandler安装到ChannelPipeline的步骤如下：**   
 （1）一个ChannelInitializer的实现被注册到了ServerBootstrap或者Bootstrap中   
 （2）当ChannelInitializer.initChannel()方法被调用时，ChannelInitializer将在ChannelPipeline中安装自定义的ChannelHandler   
 （3）ChannelInitializer将自己从ChannelPipeline中移除   
 例如：

 
```
ServerBootstrap boot = new ServerBootstrap();

boot.group(connGroup, workGroup).channel(NioServerSocketChannel.class).option(ChannelOption.SO_BACKLOG, 128)
    .option(ChannelOption.SO_KEEPALIVE, true).childOption(ChannelOption.TCP_NODELAY, true)
    .childHandler(new ChannelInitializer<SocketChannel>() {
        @Override
        protected void initChannel(SocketChannel ch) throws Exception {
            //添加一组ChannelHandler
            ch.pipeline().addLast(new ObjectDecoder(1024 * 1024,
                    ClassResolvers.cacheDisabled(this.getClass().getClassLoader())));
            ch.pipeline().addLast(new ObjectEncoder());
            ch.pipeline().addLast(new MyInboundHandler());
        }
    });

try {
    ChannelFuture future = boot.bind(port).sync();
    if (future.isSuccess()) {
        serverSocketChannel = (ServerSocketChannel) future.channel();
        System.out.println("服务器已启动");
    } else {
        System.out.println("服务器启动失败");
    }
    future.channel().closeFuture().sync();
} catch (InterruptedException e) {
    e.printStackTrace();
} finally {
    connGroup.shutdownGracefully();
    workGroup.shutdownGracefully();
}
```
 **关于ChannelHandler添加顺序**   
 （1）进站处理器的执行顺序是按添加顺序(addLast)顺序执行的，出站处理器是逆序执行的。   
 （2）出站处理器不能添加到最后，一定要添加到最后一个进站处理器之前，否则这个出站处理器是无效的。

   
  