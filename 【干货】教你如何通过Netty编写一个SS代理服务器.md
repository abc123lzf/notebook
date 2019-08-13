---
title: 【干货】教你如何通过Netty编写一个SS代理服务器
date: 2019-05-31 23:39:12
tags: CSDN迁移
---
 版权声明：尊重原创，转载请注明出处 [ https://blog.csdn.net/abc123lzf/article/details/90724591]( https://blog.csdn.net/abc123lzf/article/details/90724591)   
  ### []()准备

 本文假设读者具备以下知识：

  
  * 熟悉Java网络编程（了解BIO/NIO）与多线程编程（了解JUC中的常用工具） 
  * 熟悉Netty网络编程框架 
  * 熟悉Socks5代理协议、SSL加密通信  开发环境：

  
  * JDK 1.8 
  * Intellij IDEA  
### []()功能需求

  
  * 通过客户端，接收其他应用程序的Socks5协议的代理请求（仅限TCP代理） 
  * 客户端能够自行选择是全局模式、PAC模式还是直连模式（完全不走服务端） 
  * 服务器负责接受客户端请求，进行消息解密后与目标服务器取得连接，并转发给目标服务器。收到目标服务器响应后再将其转发给客户端。 
  * 服务端具备客户端的认证功能 
  * 客户端和服务端的通信内容和特征不能够被轻易的识别  
### []()性能需求

  
  * 服务端能够支持100个客户端的连接，同一时间段能够流畅处理25个客户端的请求 
  * 客户端不能够占用过多系统资源 
  * 客户端可以流畅地观看视频  
### []()源码

 [https://github.com/abc123lzf/flyingsocks](https://github.com/abc123lzf/flyingsocks)  
 选择分支v1.0，欢迎Fork/Star  
 为了表述方便，在文章中我们称该项目为 `flyingsocks` 

 
### []()客户端功能实现

 客户端程序基本结构：  
 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190726222044851.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FiYzEyM2x6Zg==,size_16,color_FFFFFF,t_70)

 
#### []()1、本地Socks5代理请求的接收

 Socks5代理请求包含了3个阶段（均使用同一个TCP连接）：

  
  2. 首先应用程序发送一个**Socks5初始化报文**。初始化报文一般包含了协议的版本及一些基本的信息，客户端仅需要发送一个响应报文（其内容一般为是否需要进行认证） 
  4. 如果需要认证，那么应用程序会发送一个**Socks5认证报文**。客户端收到应用程序的认证报文后需要进行用户名、密码的核对，如果通过则发送一个 `SUCCESS` 报文，否则发送一个 `FAILURE` 报文并关闭连接。 
  6. 接下来，应用程序会发送一个**Socks5命令报文**，该报文包含了代理类型（TCP还是UDP代理）、目的主机名或IP地址、端口号。客户端收到该报文后，根据其报文内容向应用程序返回一条消息（内容主要是是否能够对该目标主机进行代理）。如果是UDP代理，还需要向客户端返回一个端口号，表示接下来的代理内容发送到客户端的这个端口（这里暂时不考虑UDP代理，所以但凡是UDP代理统一返回 `COMMAND_NOT_SUPPORTED` 报文） 
  8. 完成上述所有步骤后，应用程序便可以通过该连接发送代理内容了。  Netty作为一个成熟的网络编程框架，自然配备了可以处理Socks5代理请求的工具。  
 比如：

  
  * `SocksInitRequestDecoder` ，用于解析Socks5初始化请求报文 
  * `SocksCmdRequestDecoder` ，用于解析Socks5命令请求报文 
  * `SocksMessageEncoder` ，用于编码Socks响应，即将模型对象转换为 `ByteBuf`  
  * `SocksInitRequest` 、 `SocksInitResponse` ，Socks5初始化报文请求、响应模型 
  * `SocksAuthRequest` 、 `SocksAuthResponse` ，Socks5认证报文请求、响应模型 
  * `SocksCmdRequest` 、 `SocksCmdResponse` ，Socks5命令报文请求、响应模型  在 `flyingsocks` 中，我们使用 `SocksReceiverComponent` 组件来完成本地Socks5代理请求的接收。

 
##### []()（1）引导

 
```
ServerBootstrap boot = new ServerBootstrap();
boot.group(new NioEventLoopGroup(1), new NioEventLoopGroup(4))
    .channel(NioServerSocketChannel.class)
    .childHandler(new ChannelInitializer<SocketChannel>() {
    	@Override
        protected void initChannel(SocketChannel socketChannel) {
        	ChannelPipeline cp = socketChannel.pipeline();
            cp.addLast(new SocksInitRequestDecoder());
            cp.addLast(new SocksMessageEncoder());
            cp.addLast(new SocksRequestHandler()); //这里负责调用我们自定的业务逻辑
        }
    });
//绑定1080端口，只允许本地应用程序连接
boot.bind("127.0.0.1", 1080).sync();

```
 此时管道中的 `Handler` 有：  
 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190727025023747.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FiYzEyM2x6Zg==,size_16,color_FFFFFF,t_70)

 
##### []()（2）Socks5基本消息处理Handler：SocksRequestHandler

 
```
private class SocksRequestHandler extends SimpleChannelInboundHandler<SocksRequest> {
	@Override
    protected void channelRead0(ChannelHandlerContext ctx, SocksRequest request) {
    	switch (request.requestType()) {
        	case INIT: {  //如果是初始化报文
                if(!auth) { //如果无需认证，则返回NO_AUTH响应
                	ctx.pipeline().addFirst(new SocksCmdRequestDecoder());  //添加命令报文解码器到管道首部
                	ctx.writeAndFlush(new SocksInitResponse(SocksAuthScheme.NO_AUTH));
                } else {  //如果需要认证，则返回AUTH_PASSWORD报文
                	ctx.pipeline().addFirst(new SocksAuthRequestDecoder());
                    ctx.writeAndFlush(new SocksInitResponse(SocksAuthScheme.AUTH_PASSWORD));
                }
                break;
            }
            case AUTH: {  //如果是认证报文
            	if(!(ctx.pipeline().first() instanceof SocksCmdRequestDecoder))
            		ctx.pipeline().addFirst(new SocksCmdRequestDecoder());  //同样添加命令报文解码器到管道首部
                if(!auth) { //如果不需要认证，则直接返回SUCCESS
                	ctx.writeAndFlush(new SocksAuthResponse(SocksAuthStatus.SUCCESS));
                } else {  //如果需要认证则核对username和password
                    SocksAuthRequest req = (SocksAuthRequest) request;
                    if(req.username().equals(username) && req.password().equals(password)) {
                    	ctx.pipeline().addFirst(new SocksCmdRequestDecoder()).remove(SocksAuthRequestDecoder.class);
                        ctx.writeAndFlush(new SocksAuthResponse(SocksAuthStatus.SUCCESS));
                    } else {
                        ctx.writeAndFlush(new SocksAuthResponse(SocksAuthStatus.FAILURE));
                    }
                }
                break;
            }
            case CMD: {
                SocksCmdRequest req = (SocksCmdRequest) request;
                SocksCmdType type = req.cmdType();
                if(type == SocksCmdType.CONNECT) {  //如果是TCP代理
                	//添加SocksCommandRequestHandler，并移除当前Handler
                    ctx.pipeline().addLast(new SocksCommandRequestHandler()).remove(this);
                    //传递给SocksCommandRequestHandler处理
                    ctx.fireChannelRead(req);
                } else {  //如果是UDP或者其他类型的代理，则返回COMMAND_NOT_SUPPORTED并关闭连接
                    ctx.writeAndFlush(new SocksCmdResponse(SocksCmdStatus.COMMAND_NOT_SUPPORTED, SocksAddressType.IPv4));
                    ctx.close();
                    return;
                }
                break;
            }
            case UNKNOWN: {  //如果是未知类型的报文则关闭连接
                ctx.close();
        	}
 		}
	}
}

```
 `SocksRequestHandler` 可以处理任何Socks5请求，包括初始化请求、认证请求和命令请求。在完成初始化请求和认证请求后，如果无需认证，会在管道首部添加一个 `SocksCmdRequestDecoder` ：  
 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190727025333205.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FiYzEyM2x6Zg==,size_16,color_FFFFFF,t_70)  
 如果需要认证，则首先是添加一个 `SocksAuthRequestDecoder` ：  
 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190727030036133.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FiYzEyM2x6Zg==,size_16,color_FFFFFF,t_70)  
 完成认证步骤后，才会在首部添加 `SocksCmdRequestDecoder` ，并删除 `SocksAuthRequestDecoder` 。  
 当客户端成功指定目标服务器及其端口后，会移除 `SocksRequestHandler` 并添加 `SocksCommandRequestHandler` ：  
 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190727030441793.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FiYzEyM2x6Zg==,size_16,color_FFFFFF,t_70)

 
##### []()（3）Socks5命令处理器：SocksCommandRequestHandler

 
```
private class SocksCommandRequestHandler extends SimpleChannelInboundHandler<SocksCmdRequest> {
	@Override
    protected void channelRead0(ChannelHandlerContext ctx, SocksCmdRequest request) {
    	String host = request.host();
        int port = request.port();
        //将目标主机名、端口号、应用程序与客户端连接的Channel对象封装到SocksProxyRequest
        SocksProxyRequest spq = new SocksProxyRequest(host, port, ctx.channel());
        //返回SUCCESS响应
        ctx.writeAndFlush(new SocksCmdResponse(SocksCmdStatus.SUCCESS, SocksAddressType.IPv4));
        //添加TCPProxyMessageHandler，移除该Handler
        ctx.pipeline().addLast(new TCPProxyMessageHandler(spq)).remove(this);
    }
}

```
 `SocksProxyRequest` 是我们自定义的对象，用来封装本次 `Socks5` 代理请求目标主机名、端口号、应用程序与 `flyingsocks` 客户端连接的Channel对象、消息队列等。  
 完成上述操作后，会将 `TCPProxyMessageHandler` 添加到管道尾部，并移除 `SocksCommandRequestHandler` ：  
 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190727030628530.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FiYzEyM2x6Zg==,size_16,color_FFFFFF,t_70)

 
##### []()（4）TCP代理消息转发处理器：TCPProxyMessageHandler

 在完成上述操作后，应用程序便会将二进制流发送给客户端，要求客户端将其转发给目标服务器。

 
```
private class TCPProxyMessageHandler extends SimpleChannelInboundHandler<ByteBuf> {

	private final SocksProxyRequest proxyRequest;
    private TCPProxyMessageHandler(SocksProxyRequest request) {
        super(false);
        this.proxyRequest = request;
        //将请求通知给其他组件
        getParentComponent().publish(request);
    }
    
    @Override
    protected void channelRead0(ChannelHandlerContext ctx, ByteBuf msg) {
    	//一旦收到消息就放到SocksProxyRequest的消息队列中
        proxyRequest.getMessageQueue().offer(msg);
    }

    @Override
    public void channelInactive(ChannelHandlerContext ctx) {
        ctx.pipeline().remove(this);
        ctx.fireChannelInactive();
    }
}

```
 
#### []()2、客户端直连的实现

 在以下情况我们会通过客户端直接建立与目标服务器的连接：

  
  * 客户端设置为直连模式 
  * 客户端设置为PAC模式，且目标服务器不在PAC列表中  客户端直连处理逻辑我们统一通过 `SocksProxyComponent` 和 `SocksSenderComponent` 来实现。  
 其中， `SocksProxyComponent` 维护了一组线程，负责从 `SocksProxyRequest` 中的消息队列拉取消息，并转发给 `SocksSenderComponent` 。

 
##### []()(1) 代理请求对象：SocksProxyRequest

 `SocksProxyRequest` 继承了 `ProxyRequest` ，其中 `ProxyRequest` 维护了以下变量：

 
```
//目标服务器主机名
protected String host;
//目标服务器端口
protected int port;
//客户端和应用程序的SocketChannel通道
protected Channel clientChannel;

```
 `SocksProxyRequest` 还维护了以下变量：

 
```
//当无需进行代理直接与目标服务器(例如www.baidu.com)连接时的Channel对象
private Channel serverChannel;
//消息队列
private final BlockingQueue<ByteBuf> messageQueue;

```
 可能有人会问，为什么需要维护一个消息队列。因为客户端负责转发的是TCP流量，并不会关注这个流量是采用了哪个应用层协议，该流量是如何“分包”的对于客户端来说是不透明的，所以为了保证所有的流量都能够按序抵达目标服务器，我们才需要维护一个 `ByteBuf` 消息队列，只要 `ChannelInboundHandler` 收到一个 `ByteBuf` 对象，就将这个对象推送到该消息队列中。

 
##### []()(2) SocksProxyRequest的发布

 为了能够让组件 `SocksProxyComponent` 和 `SocksSenderComponent` 处理一个新的 `SocksProxyRequest` ， `SocksProxyComponent` 必须提供一个 `publish` 接口能够让 `SocksReceiverComponent` 来发布 `SocksProxyRequest` ，这里我们先介绍 `SocksProxyComponent` 的父类 `ProxyComponent` 的 `publish` 实现

 
```
	@Override
    public void publish(ProxyRequest request) {
        if(requestSubscribers.size() == 0)  //如果订阅者为空
            log.warn("No RequestSubscriber found in manager");
        //判断是需要直连还是需要转发给服务器处理
        boolean np = needProxy(request.getHost());
        boolean consume = false; //该Request是否消费
        int count = 0;
        //遍历所有的订阅者
        for(ProxyRequestSubscriber sub : requestSubscribers) {
        	//判断该订阅者是否能够处理该ProxyRequest
            if(sub.requestType().isAssignableFrom(request.getClass()) &&
                    (sub.receiveNeedProxy() && np || sub.receiveNeedlessProxy() && !np)) {
                if(count == 0) {
                    sub.receive(request);
                } else {  //如果有第二个订阅者能够处理本请求，那么复制一份Request给该订阅者
                    try {
                        sub.receive((ProxyRequest) request.clone());
                    } catch (CloneNotSupportedException e) {
                        throw new Error(e);
                    }
                }

                consume = true;
                count++;
            }
        }

        if(!consume) {  //如果没有消费就回收该ByteBuf
            ReferenceCountUtil.release(request.getClientMessage());
            if(log.isWarnEnabled())
                log.warn("ProxyRequest was not consume");
        }
    }

```
 `ProxyComponent` 的 `publish` 实现较为简单，即遍历所有的 `ProxyRequest` 订阅者（实现 `ProxyRequestSubscriber` 接口的对象），然后根据 `ProxyRequest` 类型、是否需要转发到服务器来判定该订阅者能否处理这个 `ProxyRequest` ，如果可以处理，就调用 `ProxyRequestSubscriber` 接口的 `receive` 方法。

 这里我们再来看 `ProxyComponent` 的子类 `SocksProxyComponent` 的实现：

 
```
@Override
public void publish(ProxyRequest request) {
	super.publish(request);
    if(!super.needProxy(request.getHost())) {
		ClientMessageTransferTask task = transferTaskList.get(
					Math.abs(request.hashCode() % transferTaskList.size()));
        task.pushProxyRequest((SocksProxyRequest) request);
    }
}

```
 这里首先会调用父类的 `publish` 方法，如果该 `ProxyRequest` 无需转发给 `flyingsocks` 服务器，那么根据 `SocksProxyRequest` 的 `hashCode` ，从 `transferTaskList` 中选取一个 `ClientMessageTransferTask` ，然后将 `SocksProxyRequest` 推送给该 `ClientMessageTransferTask` 。

 需要注意的是，每个 `ClientMessageTransferTask` 都由一个线程管理， `ClientMessageTransferTask` 和 `Thread` 是一一对应的关系。 `ClientMessageTransferTask` 的职责是负责管理一个 `SocksProxyRequest` 列表，通过不停地尝试从每个 `SocksProxyRequest` 的消息队列拉取 `ByteBuf` 发送给客户端与目标服务器的 `SocketChannel` （在该 `SocketChannel` 活跃的情况下，也就是成功与目标服务器构建了一个TCP连接）。

 为什么要设计一个 `ProxyComponent` 基类？这是考虑到后期可能需要添加HTTP代理功能（虽然 `Socks5` 代理协议也可以代理 `HTTP` 协议），HTTP代理协议是一个应用层的代理协议，不像 `Socks5` 代理协议是基于传输层的，我们可以根据HTTP代理协议的特征划分出每个包，这样就不需要维护一个 `ByteBuf` 消息队列了（只需要保存一个 `ByteBuf` 就可以了）。

 `ClientMessageTransferTask` 实现如下：

 
```
	private final class ClientMessageTransferTask implements Runnable {
		//SocksProxyRequest列表，由于每个ClientMessageTransferTask是单个线程独占的，所以无需确保线程安全
        private final List<SocksProxyRequest> requests = new LinkedList<>();
        //新的SocksProxyRequest统一放置于该队列
        private final BlockingQueue<SocksProxyRequest> newRequestQueue = new LinkedBlockingQueue<>();

        @Override
        public void run() {
            Thread t = Thread.currentThread();
            while(!t.isInterrupted()) {  //只要线程尚未中断，就一直执行
                ListIterator<SocksProxyRequest> it = requests.listIterator();
                while(it.hasNext()) {
                    SocksProxyRequest req = it.next();
                    Channel sc, cc;  //分别是客户端与目标服务器的Channel、应用程序和客户端的Channel
                    //如果客户端与应用程序的连接已经失效，那么从列表移除这个SocksProxyRequest，并且清空消息队列防止内存泄漏
                    if((cc = req.getClientChannel()) != null && !cc.isActive()) {
                        it.remove();
                        clearProxyRequest(req);
                        continue;
                    }
					//如果客户端与目标服务器的Channel为null，说明连接尚未建立成功
                    if((sc = req.getServerChannel()) == null) {
                        continue;
                    }
					//如果与目标服务器的连接失效，那么同样从列表移除这个SocksProxyRequest并清空消息队列
                    if(!sc.isActive()) {
                        it.remove();
                        clearProxyRequest(req);  ///删除无用的ProxyRequest
                        continue;
                    }
					//获取ProxyRequest中的消息队列
                    BlockingQueue<ByteBuf> queue = req.getMessageQueue();
                    ByteBuf buf;
                    try { //通过循环依次发送到目标服务器
                        while ((buf = queue.poll(1, TimeUnit.MILLISECONDS)) != null) {
                            sc.writeAndFlush(buf);
                        }
                    } catch (InterruptedException e) {
                        break;
                    }
                }

                //接收新的代理请求，并添加到list，等待下一个循环的处理
                SocksProxyRequest newRequest;
                try {
                    while ((newRequest = newRequestQueue.poll(2, TimeUnit.MILLISECONDS)) != null) {
                        requests.add(newRequest);
                    }
                } catch (InterruptedException e) {
                    break;
                }
            }
        }

        private void clearProxyRequest(SocksProxyRequest request) {
            BlockingQueue<ByteBuf> queue = request.getMessageQueue();
            ByteBuf buf;
            try {  //通过Netty的ReferenceCountUtil释放ByteBuf
                while ((buf = queue.poll(1, TimeUnit.MILLISECONDS)) != null) {
                    ReferenceCountUtil.release(buf);
                }
            } catch (InterruptedException ignore) {
                // IGNORE
            }
        }
		//用于添加新的SocksProxyRequest
        private void pushProxyRequest(SocksProxyRequest request) {
            newRequestQueue.offer(request);
        }

```
 
##### []()3、SocksProxyRequest的订阅者：SocksSenderComponent

 `SocksSenderComponent` 的非静态内部类 `SocksProxyRequestSubscriber` 实现了 `ProxyRequestSubscriber` 接口，它的任务比较简单：建立与目标服务器的连接。

 
```
	private final class SocksProxyRequestSubscriber implements ProxyRequestSubscriber {
        @Override
        public void receive(ProxyRequest req) {
            SocksProxyRequest request = (SocksProxyRequest) req;
            String host = request.getHost();
            int port = request.getPort();

            Bootstrap b = connectBootstrap.clone();
            b.handler(new ChannelInitializer<SocketChannel>() {
                @Override
                protected void initChannel(SocketChannel ch) {
             		//添加一个DirectConnectHandler即可
                    ch.pipeline().addFirst(new DirectConnectHandler(request));
                }
            });
			//向目标服务器发起连接，并添加监听器监听连接是否成功
            b.connect(host, port).addListener((ChannelFuture f) -> {
                if(!f.isSuccess()){
                    if(f.cause() instanceof ConnectTimeoutException) {
                        if(log.isInfoEnabled())
                            log.info("connect to " + request.getHost() + ":" + request.getPort() + " failure, cause connect timeout");
                    } else if(log.isWarnEnabled()) {
                    	log.warn("connect establish failure, from " + request.getHost() + ":" + request.getPort(), f.cause());
                    }
                    request.getClientChannel().close();  //连接建立失败关闭与应用程序的连接
                    f.channel().close();
                } else {
                    if(log.isTraceEnabled())
                        log.trace("connect establish success, from {}:{}", request.getHost(), request.getPort());
                    //连接建立成功，设置ServerChannel
                    request.setServerChannel(f.channel());
                }
            });
        }

        @Override
        public boolean receiveNeedProxy() {
            return false;
        }

        @Override
        public boolean receiveNeedlessProxy() {
            return true;
        }

        @Override
        public Class<? extends ProxyRequest> requestType() {
            return SocksProxyRequest.class;
        }
    }

```
 可以看到， `SocksProxyRequestSubscriber` 仅仅就是建立与目标服务器的连接，并监听连接建立状态。在连接建立后，会向管道中添加我们自定义的 `DirectConnectHandler` ，负责处理目标服务器发来的消息。这个Handler逻辑很简单，收到目标服务器发来的消息后就立刻通过应用程序与客户端的 `Channel` 发送该消息给应用程序，没有中间商赚差价，具体代码就不贴了。

 
#### []()3、客户端与服务器的通信

 当目标网站（例如谷歌、YouTube、Facebook等）需要通过代理才能连接时，就需要将客户端收到的二进制流转发给服务器来处理了，对于客户端而言，这部分的逻辑都实现了在了 `ProxyServerComponent` 组件。每个 `ProxyServerComponent` 对象代表一个服务器的连接。

 客户端和服务器构建TCP连接后可以分为三个阶段：

  
  2. 客户端和服务器协商一个16字节长度的分隔符 
  4. 客户端向服务器发送认证信息 
  6. 客户端向服务器发送代理请求，服务端向客户端发送响应  我们先从引导阶段开始分析：

 
##### []()（1）引导

 通常情况下，客户端与服务器的连接是一个SSL加密的连接，所以客户端需要持有服务器的SSL证书。所以在引导之前，我们首先需要根据证书来生成一个 `SslHandler` （这里使用OpenSSL创建的证书）：

 
```
InputStream is = ....  //证书文件的输入流
SslContext ctx = SslContextBuilder.forClient().trustManager(is).build();
SslHandler handler = ctx.newHandler(ByteBufAllocator.DEFAULT);

```
 这样就生成了一个 `SslHandler` 实例。需要注意的是， `SslHandler` 实例是不能够在不同的连接之间共享的，所以每产生一个连接就需要根据 `SslContext` 实例（ `SslContext` 实例可以复用）来生成一个 `SslHandler` 。

 下面我们来看看客户端与服务器建立连接的引导阶段：

 
```
EncryptProvider provider = EncryptSupport.lookupProvider("OpenSSL", OpenSSLEncryptProvider.class);
//EncryptProvider初始化流程...
loopGroup = new NioEventLoopGroup(2);
bootstrap = new Bootstrap()
			.channel(NioSocketChannel.class)
            .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, cfg.getConnectionTimeout())
            .option(ChannelOption.SO_KEEPALIVE, true)
            .handler(new ChannelInitializer<SocketChannel>() {
                @Override
                protected void initChannel(SocketChannel ch) throws Exception {
                    ChannelPipeline cp = ch.pipeline();
                    if(provider != null) {
                    	//如果加密/解密消息的InboundHanlder和OutboundHandler是同一个Handle
                        if(!provider.isInboundHandlerSameAsOutboundHandler())
                            cp.addLast(provider.encodeHandler(params));
                        cp.addLast(provider.decodeHandler(params));
                    }
                    //用于业务逻辑
                    cp.addLast(new InitialHandler());
                }
            });

```
 如果是通过SSL进行加密连接，那么会在连接建立的时候添加两个Handler： `SslHandler` 和 `InitialHandler` ，其中 `SslHandler` 即是进站处理器也是出站处理器， `InitialHandler` 是进站处理器。

 完成 `Bootstrap` 对象的创建后，我们会调用 `doConnect` 方法来发起连接：

 
```
private void doConnect(boolean sync) {
	if(active)
        throw new IllegalStateException("This component has been connect.");

    String host = config.getHost();  //Flyingsocks服务器地址
    int port = config.getPort();     //Flyingsocks服务器端口

    if(log.isInfoEnabled())
        log.info("Connect to flyingsocks server {}:{}...", host, port);

    Bootstrap b = bootstrap.clone().group(this.loopGroup);
    ChannelFuture f = b.connect(host, port);

    final CountDownLatch waitLatch = new CountDownLatch(1);
    f.addListener(new GenericFutureListener<Future<? super Void>>() {
        @Override
        public void operationComplete(Future<? super Void> future) {
            if (future.isSuccess()) {  //如果连接成功
                if(log.isInfoEnabled())
                    log.info("Connect success to flyingsocks server {}:{}", host, port);
				//标记为活跃状态
                active = true;
                //提交TransferTask，用于处理ProxyRequest
                for(int i = 0; i < DEFAULT_PROCESSOR_THREAD; i++) { 
                    clientMessageProcessor.submit(new ClientMessageTransferTask());
                }
                //连接成功后注册到父组件
                getParentComponent().registerSubscriber(ProxyServerComponent.this);
                f.removeListener(this);
            } else {
                log.warn("Can not connect to flyingsocks server, cause:", future.cause());
                f.removeListener(this);
                afterChannelInactive(); //重新尝试连接
            }
            if(sync) //释放等待的线程
            	waitLatch.countDown();
            }
        });

    try {
        if (sync)  
            waitLatch.await(10000, TimeUnit.MILLISECONDS);
    } catch (InterruptedException e) {
        if(log.isWarnEnabled())
            log.warn("ProxyServerComponent interrupted when synchronize doConnect");
    }
}

```
 
##### []()（2）初始化Handler：InitialHandler

 成功构建与Flyingsocks服务器的连接后， `InitialHandler` 会被添加到管道中，之后 `InitialHandler` 会生成一个随机的16字节长的byte数组，作为服务端与客户端通信协议的分隔符：

 
```
@Override
public void channelActive(ChannelHandlerContext ctx) {
	if(log.isTraceEnabled())
		log.trace("Start flyingsocks server connection initialize");
    ProxyServerSession session = new ProxyServerSession((SocketChannel) ctx.channel());

    Random random = new Random();
    byte[] delimiter = new byte[DelimiterMessage.DEFAULT_SIZE];
    random.nextBytes(delimiter);
    DelimiterMessage msg = new DelimiterMessage(delimiter);
    session.setDelimiter(delimiter);
    ProxyServerComponent.this.proxyServerSession = session;

    try {
        ctx.writeAndFlush(msg.serialize());
        ctx.fireChannelActive();
    } catch (SerializationException e) {
    	log.error("Serialize DelimiterMessage occur a exception", e);
        ctx.close();
    }
}

```
 这个分隔符会立刻发送到服务器，服务器收到该分隔符后会发送一个确认消息，消息内容就是分隔符的内容，达到一个“握手”目的。 `InitialHandler` 收到确认消息后，会在 `SslHandler` 后面添加一个 `DelimiterBasedFrameDecoder` （Netty自带），这样，后面的Handler收到的都是被分隔符分隔消息，就不会出现所谓的“粘包”、“半包”问题了。除了 `DelimiterBasedFrameDecoder` ，我们还需要添加一个 `AuthHandler` ，这个 `Handler` 是用于实现认证的（类似于SS那样需要填写服务器密码）：

 
```
@Override
public void handlerAdded(ChannelHandlerContext ctx) {
	AuthMessage msg;
    switch (config.getAuthType()) {  //根据配置信息获取认证方式
        case SIMPLE: msg = new AuthMessage(AuthMessage.AuthMethod.SIMPLE); break;
        case USER: msg = new AuthMessage(AuthMessage.AuthMethod.USER); break;
        default:
        	throw new IllegalArgumentException("Auth method: " + config.getAuthType() + " Not support.");
    }

    List<String> keys = msg.getAuthMethod().getContainsKey();
    for(String key : keys) {  //将认证需要的参数(例如用户名、密码)放入请求
    	msg.putContent(key, config.getAuthArgument(key));
    }
    try {  //执行请求序列化，并发送至服务器
    	ctx.writeAndFlush(msg.serialize());
    	//向管道添加ProxyHandler
        ctx.pipeline().remove(this).addLast(new ProxyHandler());
     } catch (SerializationException e) {
         log.error("Serialize AuthMessage occur a exception:", e);
     }
}

```
 这里会根据配置文件构造好认证请求 `AuthMessage` ，将其序列化为 `ByteBuf` 通过 `Channel` 管道发送给服务器，移除当前 `Handler` 并添加 `ProxyHandler` 准备接收服务器的代理消息（如果认证尚未通过，服务器会主动关闭连接）。

 认证通过后，客户端就可以正式向服务器发送代理消息了，客户端和服务器的通信始终是通过一条连接通信。那么，客户端是如何向服务器发送应用程序的消息，并能够正确地接收服务器发回的响应信息发回给服务器的呢？为了保证这个需求，我们将客户端与服务器的代理请求报文设置成了以下格式：  
 ![在这里插入图片描述](https://img-blog.csdnimg.cn/2019072621541014.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FiYzEyM2x6Zg==,size_16,color_FFFFFF,t_70)  
 上述是客户端向服务器的代理请求报文格式，通过记录客户端与应用程序的 `ChannelID` ，保证服务器在传回目标服务器的响应时，客户端能够通过其 `ChannelID` 找到对应的 `Channel` 并将响应写到应用程序。客户端只需要记录 `ChannelID` 与 `Channel` 对象的映射关系就可以了。  
 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190726215929153.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FiYzEyM2x6Zg==,size_16,color_FFFFFF,t_70)  
 上图就是响应报文格式，其中 `ChannelID` 与请求报文中的 `ChannelID` 是一致的，状态字段表示 `flyingsocks` 服务器是否成功收到了目标服务器的响应信息，如果字段结果是收到了，那么就会读取后续的响应长度字段，然后再将响应信息提取出来发回应用程序。

 `ProxyHandler` 主要功能是解析服务器发回的 `ProxyResponseMessage` ，核心功能实现在了 `channelRead` 方法：

 
```
@Override
public void channelRead(ChannelHandlerContext ctx, Object msg) {
	if(msg instanceof ByteBuf) {
		try {
			ProxyResponseMessage resp;
			try {
				resp = new ProxyResponseMessage((ByteBuf) msg);  //反序列化为对象
			} catch (SerializationException e) {
				log.warn("Serialize ProxyResponseMessage error", e);
				ctx.close();
				return;
			}

			if (resp.getState() == ProxyResponseMessage.State.SUCCESS) {
				//根据ChannelID找到对应的ProxyRequest对象，其中包含了Channel对象
				ProxyRequest req = activeProxyRequestMap.get(resp.getChannelId());
				if (req == null)
					return;
				Channel cc;
				if ((cc = req.getClientChannel()).isActive())
`					cc.writeAndFlush(resp.getMessage());
			}
		} finally {
			ReferenceCountUtil.release(msg);
		}
	} else {
		ctx.fireChannelRead(msg);
	}
}

```
 该方法将 `ByteBuf` 反序列化为 `ProxyResponseMessage` ，提取其中的 `ChannelID` 并获取 `Channel` 对象，再将 `ProxyResponseMessage` 中的消息写入到 `Channel` 中。

 
### []()服务器功能实现

 服务器的实现实际上比客户端要简单，服务器只需要接收客户端的请求，与目标服务器发起连接，发送客户端的请求消息，并将其响应组装好写回客户端。

 
##### []()1、端口的绑定

 要想接收客户端的连接，需要通过 `ServerBootstrap` 来建立一个端口，负责接收连接请求，过程就不详细介绍了。连接建立之后，其初始 `ChannelPipeline` 管道如下：  
 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190726233301243.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FiYzEyM2x6Zg==,size_16,color_FFFFFF,t_70)  
  `SslHandler` 负责和客户端建立SSL连接，并通过RSA算法加密出站信息、解密进站信息。  
  `ClientSessionHandler` 负责构造、维护 `ClientSession` 对象，该对象维护了一组客户端信息。  
  `FixedLengthFrameDecoder` 即定长消息解码器，在连接刚刚建立的阶段，需要由客户端生成一个16字节的分隔符发送给服务器，该解码器就是将该16字节的分隔符传递给 `InitialHandler` 。 `InitialHandler` 收到该消息后，会将分隔符记录下来，然后删除 `FixedLengthFrameDecoder` ，添加一个 `DelimiterBasedFrameDecoder` 和 `DelimiterOutboundHandler` （非Netty自带），分别负责将入站字节流根据分隔符分隔，或对出站信息加上分隔符。最后添加 `AuthHandler` ，负责检验客户端的认证信息。

 
```
private final class InitialHandler extends SimpleChannelInboundHandler<ByteBuf> {
	@Override
	protected void channelRead0(ChannelHandlerContext ctx, ByteBuf msg) {
		byte[] key = new byte[DelimiterMessage.DEFAULT_SIZE];
		msg.readBytes(key);  //读取来自客户端生成的分隔符
		if(log.isTraceEnabled())
			log.trace("Receive DelimiterMessage from client.");
		//获取由ClientSessionHandler生成的ClientSession对象
		ClientSession state = getParentComponent().getClientSession(ctx.channel());
		state.setDelimiterKey(key);  //将分隔符保存在Session中
		
		ChannelPipeline cp = ctx.pipeline();
		cp.remove(this).remove(FixedLengthFrameDecoder.class);  //移除FixedLengthFrameDecoder
		//向客户端回复分隔符，达到握手的目的
		ByteBuf keyBuf = Unpooled.buffer(DelimiterMessage.DEFAULT_SIZE);
		keyBuf.writeBytes(key);
		ctx.writeAndFlush(keyBuf.copy());
		//添加DelimiterOutboundHandler、DelimiterBasedFrameDecoder
		cp.addLast(new DelimiterOutboundHandler(keyBuf));
		cp.addLast(new DelimiterBasedFrameDecoder(102400, keyBuf));
		//添加AuthHandler，负责进行认证
		cp.addLast(new AuthHandler(state));
	}
}

```
 完成上述操作后，此时的管道结构：  
 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190726235237725.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FiYzEyM2x6Zg==,size_16,color_FFFFFF,t_70)  
  `AuthHandler` 负责接收客户端的认证报文：

 
```
@Override
protected void channelRead0(ChannelHandlerContext ctx, ByteBuf buf) {
	AuthMessage msg;
	try {
		msg = new AuthMessage(buf);  //反序列化
	} catch (SerializationException e) {
		log.info("Deserialize occur a exception", e);
		ctx.close();
		return;
	}

	boolean auth = doAuth(msg);  //核实认证信息
	if(!auth) {
		if(log.isTraceEnabled())
			log.trace("Auth failure, from client {}", ((SocketChannel)ctx.channel()).remoteAddress().getHostName());
			ctx.close();
			return;
	} else {
		if(log.isTraceEnabled()) {
			log.trace("Auth success, from client {}", ((SocketChannel)ctx.channel()).remoteAddress().getHostName());
		}
	}

	clientSession.passAuth();  //在Session对象上标注该会话已经通过认证

	ChannelPipeline cp = ctx.pipeline();
	cp.remove(this).remove(DelimiterBasedFrameDecoder.class);   //删除旧的DelimiterBasedFrameDecoder
	//添加一个容许长度更大的DelimiterBasedFrameDecoder
	byte[] b = getParentComponent().getClientSession(ctx.channel()).getDelimiterKey();
	cp.addLast(new DelimiterBasedFrameDecoder(1024 * 1000 * 50,
				Unpooled.buffer(DelimiterMessage.DEFAULT_SIZE).writeBytes(b)));
	//添加ProxyHandler
	cp.addLast(new ProxyHandler(clientSession));
}

```
 `AuthHandler` 在收到认证请求消息后会将其反序列化为 `AuthMessage` ，然后调用 `doAuth` 方法与配置文件中的密码进行比对，若没有通过认证则直接关闭连接。若通过认证，则删除旧的 `DelimiterBasedFrameDecoder` ，添加一个新的 `DelimiterBasedFrameDecoder` ，这个 `DelimiterBasedFrameDecoder` 能够容许更长的报文长度，以便正常接收客户端的代理请求消息。最后再添加 `ProxyHandler` 正式为客户端提供代理服务。  
 此时的管道如下：  
 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190727001210935.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FiYzEyM2x6Zg==,size_16,color_FFFFFF,t_70)

 
##### []()2、代理请求的接收

 `ProxyHandler` 在收到客户端发送的 `ClientSession` 后，会将 `ProxyRequestMessage` 和 `ClientSession` 封装为一个 `ProxyTask` 对象，并发送到 `ProxyProcessor` ：

 
```
@Override
protected void channelRead0(ChannelHandlerContext ctx, ByteBuf buf) {
	ProxyRequestMessage msg;
	try {
		msg = new ProxyRequestMessage(buf);  //反序列化
	} catch (SerializationException e) {
		if(log.isWarnEnabled())
			log.warn("Serialize request occur a exception", e);
		ctx.close();
		return;
	}

	if(log.isTraceEnabled())
		log.trace("Receiver client ProxyRequest from {}", 
				clientSession.remoteAddress().getAddress().getHostAddress());
	ProxyTask task = new ProxyTask(msg, clientSession);
	//发布代理任务
	getParentComponent().publish(task);  //调用了父组件的publish方法
}

```
 父组件的 `publish` 方法会将代理任务发布给 `ProxyTaskSubscriber` ：

 
```
@Override
public void publish(ProxyTask task) {
	if(proxyTaskSubscribers.size() == 0)
		log.warn("No ProxyTaskSubscriber register.");

	int count = 0;
	for(ProxyTaskSubscriber subscriber : proxyTaskSubscribers) {
		if(count == 0) {
			subscriber.receive(task);
		} else {
			try {
				subscriber.receive(task.clone());
			} catch (CloneNotSupportedException e) {
				throw new IllegalStateException(e);
			}
		}

		count++;
	}
}

```
 
##### []()2、与目标服务器的通信

 当 `ProxyProcessor` 收到代理请求后，会将 `ProxyTask` 转发到 `DispathcerProcessor` 的一个内部类 `DispathcherTask` 。该内部类负责建立与目标服务器的连接、发送请求内容和接收响应内容，是整个服务器的核心组件。

 当 `DispathcherTask` 收到 `ProxyTask` 后，需要考虑两种情况：

  
  * 对于一个特定的 `ChannelID` ，当前组件并没有和 `ProxyRequestMessage` 当中包含的目标服务器建立起连接，需要建立一个新的连接。 
  * 对于一个特定的 `ChannelID` ，当前组件已经和 `ProxyRequestMessage` 当中包含的目标服务器建立起连接，为了保证消息对于目标服务器来说是完整和有序的，需要复用这个连接，并将 `ProxyRequestMessage` 当中包含的消息通过这个连接的 `Channel` 发送给目标服务器。  为了保证连接能够复用，需要编写一个类 `ActiveConnection` ：

 
```
static final class ActiveConnection {
	final String host;          //目标主机IP/域名
	final int port;             //目标主机端口号
	final String clientId;      //客户端的客户端的ChannelID
	ChannelFuture future;       //该连接的ChannelFuture
	final Queue<ByteBuf> msgQueue;    //若上述future持有的Channel尚未Active，则该队列负责保存该连接的客户端数据

	ActiveConnection(String host, int port, String clientId) {
		this.host = host;
		this.port = port;
		this.clientId = clientId;
		msgQueue = new LinkedList<>();
	}

	@Override
	public int hashCode() {
		return host.hashCode() ^ port ^ clientId.hashCode();
	}

	@Override
	public boolean equals(Object obj) {
		if(this == obj)
			return true;
		if(obj instanceof ActiveConnection) {
			ActiveConnection c = (ActiveConnection) obj;
			return this.host.equals(c.host) && this.port == c.port && this.clientId.equals(c.clientId);
		}
		return false;
	}
}

```
 `ChannelFuture` 可以获知连接建立的情况，如果建立成功还可以获取对应的 `Channel` 对象。 `msgQueue` 就是在连接尚未构造好时负责保存客户端发送的消息，待连接构造好后一并发送。由于 `ActiveConnection` 对象是线程独享的，所以并不需要保证其线程安全性。

 了解完 `ActiveConnection` 后，我们来看看 `DispatcherTask` 是如何执行上述逻辑的：

 
```
private final class DispatcherTask implements Runnable, ProxyTaskSubscriber {
		private final Map<ClientSession, ReturnableSet<ActiveConnection>> activeConnectionMap = new LinkedHashMap<>(64);
		private final BlockingQueue<ProxyTask> taskQueue = new LinkedBlockingQueue<>();

		private DispatcherTask() {
			parent.registerSubscriber(this);  //注册到父容器中，表示订阅ProxyTask
		}

		@Override
		public void run() {
			try {
				Thread thread = Thread.currentThread();
				while (!thread.isInterrupted()) {
					ProxyTask task = taskQueue.poll(2500, TimeUnit.MILLISECONDS);
					if(task == null) {
						checkoutConnection();   //删除无用的ActiveConnection
						continue;
					}

					if(log.isTraceEnabled())
						log.trace("Receive ProxyTask at DispatcherTask thread");

					try {
						//从ProxyTask中提取出请求内容和会话对象
						ProxyRequestMessage prm = task.getProxyRequestMessage();
						ClientSession cs = task.getSession();
						//如果与客户端的连接失效了，那么忽略这个ProxyTask
						if(!cs.isActive())
							continue;
						//为当前客户端会话构造一个Set集合(如果不存在的话)
						ReturnableSet<ActiveConnection> set = activeConnectionMap.computeIfAbsent(cs, key -> new ReturnableLinkedHashSet<>(128));
						//获取目标服务器的主机名和端口
						String host = prm.getHost();
						int port = prm.getPort();
						//尝试构造一个ActiveConnection
						ActiveConnection conn = new ActiveConnection(host, port, prm.getChannelId());
						ActiveConnection sconn;
						//如果集合中不包含这个ActiveConnection
						if((sconn = set.getIfContains(conn)) != null)
							conn = sconn;
						if(sconn != null) {  //到这里说明已经存在与目标服务器的连接
							ChannelFuture f = conn.future;
							Channel c = f.channel(); 
							if(f.isDone() && f.isSuccess()) { //如果连接成功
								if (c.isActive()) { //如果连接仍处于活跃状态
									ByteBuf buf;
									//将ActiveConnection中msgQueue的消息依次发送到服务器
									while((buf = conn.msgQueue.poll()) != null) {
										c.write(buf);
									}
									c.writeAndFlush(prm.getMessage());
								}
							} else if(!f.isDone()) { //如果正处于连接状态，那么添加到队列中
								conn.msgQueue.add(prm.getMessage());
							} else { //如果连接建立失败，从Set集合中删除这个ActiveConnection
								set.remove(conn);
							}
						} else {  //这里说明不存在与目标服务器的连接
							Bootstrap b = bootstrap.clone();
							b.handler(new ChannelInitializer<SocketChannel>() {
								@Override
								protected void initChannel(SocketChannel ch) {
									ch.pipeline().addLast(new DispatchHandler(task));
								}
							});
							//向目标服务器发起连接，并将ChannelFuture保存到ActiveConnection
							conn.future = b.connect(host, port).addListener(future -> {
								if (!future.isSuccess()) { //如果连接没有建立成功，那么向客户端返回一个错误的消息
									if(log.isWarnEnabled())
										log.warn("Can not connect to {}:{}", host, port);
									ProxyResponseMessage resp = new ProxyResponseMessage(prm.getChannelId());
									resp.setState(ProxyResponseMessage.State.FAILURE);
									try {
										cs.writeAndFlushMessage(resp.serialize());
									} catch (IllegalStateException e) {
										if (log.isTraceEnabled())
											log.trace("Client from {} has disconnect.", cs.remoteAddress().getAddress());
									}
								} else {
									if(log.isTraceEnabled())
										log.trace("Connect to {}:{} success", host, port);
								}
							});

							set.add(conn);
						}
						//再次进行清除无用的ActiveConnection
						checkoutConnection();

					} catch (Exception e) {
						if(log.isWarnEnabled())
							log.warn("Exception occur, at RequestReceiver thread", e);
					}
				}
			} catch (InterruptedException e) {
				if (log.isInfoEnabled())
					log.info("RequestReceiver interrupt, from {}", getName());
			} finally {
				parent.removeSubscriber(this);
			}
		}

		@Override
		public void receive(ProxyTask task) {
			taskQueue.add(task); //接收来自ProxyProcessor的代理任务
		}

		/**
		 * 检查ActiveConnection对象
		 */
		private void checkoutConnection() {
			//......
		}
	}

```
 建立与目标服务器的连接后，会向管道添加唯一一个 `Handler` ： `DispatcherHandler` 。

 
```
private class DispatchHandler extends SimpleChannelInboundHandler<ByteBuf> {
	private final ProxyTask proxyTask;

	private DispatchHandler(ProxyTask task) {
		super(false);
		this.proxyTask = task;
	}

	@Override
	public void channelActive(ChannelHandlerContext ctx) throws Exception {
		//连接建立成功后第一步就将第一个收到的ProxyRequestMessage消息发送给服务器，因为这个消息没有保存到msgQueue中
		ByteBuf buf = proxyTask.getProxyRequestMessage().getMessage();
		ctx.writeAndFlush(buf);
		super.channelActive(ctx);
	}

	@Override
	protected void channelRead0(ChannelHandlerContext ctx, ByteBuf msg) throws Exception {
		if(log.isTraceEnabled())
			log.trace("Receive from {}:{} response.", proxyTask.getProxyRequestMessage().getHost(),
				proxyTask.getProxyRequestMessage().getPort());
		//组装响应对象，将目标服务器发送过来的响应进行包装发送到客户端
		ProxyResponseMessage prm = new ProxyResponseMessage(proxyTask.getProxyRequestMessage().getChannelId());
		prm.setState(ProxyResponseMessage.State.SUCCESS);
		prm.setMessage(msg);
		try { //写入到客户端
			proxyTask.getSession().writeAndFlushMessage(prm.serialize());
		} catch (IllegalStateException e) {
			ctx.close();
		} finally {
			ReferenceCountUtil.release(msg);
		}
	}
}

```
   
  