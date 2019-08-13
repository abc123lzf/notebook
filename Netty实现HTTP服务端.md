---
title: Netty实现HTTP服务端
date: 2018-08-07 15:08:12
tags: CSDN迁移
---
 版权声明：尊重原创，转载请注明出处 [ https://blog.csdn.net/abc123lzf/article/details/81479493]( https://blog.csdn.net/abc123lzf/article/details/81479493)   
  ### 一、处理HTTP请求

 Netty作为一款优秀的网络编程框架，自然提供了实现HTTP服务的解决方案。   
 对于HTTP服务端而言，其引导过程和很多Netty服务端应用程序几乎一摸一样，唯一不同的是编解码器。   
 Netty提供了以下编解码器实现对HTTP请求的解析。   
 1、HttpRequestDecoder：一般放在进站处理器当中的第一个，解码完成后会将处理好的HttpRequest对象传送给下一个进站处理器。   
 2、HttpObjectAggregator：用来将多个零碎的HttpRequest对象拼接成一个FullHttpRequest对象，表示一个完整的HTTP请求，其构造参数为缓冲区的字节数。也可将出战方向的HttpResponse对象拼接成FullHttpResponse对象。   
 3、HttpResponseEncoder：将HttpResponse中的信息转换成标准的HTTP响应。

 下面是HTTP服务端接收请求的实现代码：

 
```
public class NettyHandler {
    private static final Log log = LogFactory.getLog(NettyHandler.class);
    //业务逻辑线程池
    private final Executor executor = Executors.newFixedThreadPool(Runtime.getRuntime().availableProcessors());
    //连接接收线程组
    private final EventLoopGroup acceptGroup = new NioEventLoopGroup();
    //进站出站处理线程组
    private final EventLoopGroup workerGroup = new NioEventLoopGroup();
    //服务器Socket通道
    private ServerSocketChannel serverChannel = null;
    //绑定的端口
    private int port = 80;

    //引导过程
    public void bootstrap() throws LifecycleException, HandlerException {
        ServerBootstrap boot = new ServerBootstrap();
        boot.group(acceptGroup, workerGroup).channel(NioServerSocketChannel.class)
                .childHandler(new ChannelInitializer<SocketChannel>() {
                    @Override
                    protected void initChannel(SocketChannel ch) throws Exception {
                        //HTTP响应编码器
                        ch.pipeline().addLast(new HttpResponseEncoder());
                        //HTTP请求解码器
                        ch.pipeline().addLast(new HttpRequestDecoder());
                        //HTTP请求聚合器，这里是为了防止NIO模式出现的请求不完整问题
                        ch.pipeline().addLast(new HttpObjectAggregator(512 * 1024));
                        //该处理器为自定义处理器，用来实现具体的业务逻辑
                        ch.pipeline().addLast(new HttpServerInboundHandler());
                    }
                }).option(ChannelOption.SO_BACKLOG, 1024) //设置最大连接数为1024，如果连接数量超出则阻塞接收连接的线程
                .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 5000); //连接超时时长（单位：毫秒）
        try {
            ChannelFuture future = boot.bind(port).sync();
            if(future.isSuccess()) {
                serverChannel = (ServerSocketChannel)future.channel();
            } else {
                log.error("绑定端口失败", e);
            }
        } catch (InterruptedException e) {
            log.error("线程被终止", e);
        }
    }

    public void start() {
        executor.execute(new NettyHandlerProcesser());
    }

    //接收客户端连接的线程    
    protected class NettyHandlerProcesser implements Runnable {
        @Override
        public void run() {
            try {
                serverChannel.closeFuture().sync();
            } catch (InterruptedException e) {
                log.warn("", e);
            }
        }
    }

    protected class RequestProcesser implements Runnable {
        private final FullHttpRequest request;
        private final ChannelHandlerContext ctx;
        public RequestProcesser(FullHttpRequest request, ChannelHandlerContext ctx){
            this.request = request;
            this.ctx = ctx;
        }
        @Override
        public void run() {
            //处理HTTP请求的业务逻辑
        }
    }

    //执行HTTP请求处理业务逻辑
    private void runRequestProcesser(FullHttpRequest request, ChannelHandlerContext ctx) {
        executor.execute(new RequestProcesser(request, ctx));
    }

    //最后一个进站处理器，用来实现业务逻辑
    class HttpServerInboundHandler extends ChannelInboundHandlerAdapter {
        @Override
        public void channelRead(final ChannelHandlerContext ctx, Object msg) {
            if(msg instanceof FullHttpRequest) {
                FullHttpRequest request = (FullHttpRequest) msg;
                runRequestProcesser(request, ctx);
            }
        }
    }
}

```
 对于一个FullHttpRequest请求对象，我们可以通过以下方法获取该请求的信息   
 1、请求行   
 请求行包含请求方法、请求URI、请求协议版本   
 获取请求行信息的代码如下：

 
```
FullHttpRequest req = ...;
String method = req.getMethod().name(); //获取请求方法，如：GET、POST 
String uri = req.getUri(); //获取请求URI
String protocol = req.getProtocolVersion().text(); //获取请求协议版本，如HTTP/1.1
```
 2、请求头   
 在Netty中，HttpHeaders表示一个请求头对象，FullHttpRequest自然包含这样一个对象   
 我们可以通过以下代码获取HttpHeaders对象：

 
```
FullHttpRequest req = ...;
HttpHeaders header = req.headers();
```
 请求头是典型的键值对结构，我们可以这样获取单个请求头字段信息

 
```
String val = header.get(HttpHeaders.Names.ACCEPT_CHARSET); //获取请求头字段Accept-Charset对应的值
```
 HttpHeaders.Names是一个保存请求头键字符串的类。

 如果想要获取所有的请求头信息，我们可以调用HttpHeaders对象的entries方法获取保存所有请求头字段的List并用一个Map来保存这些信息：

 
```
HttpHeader header = req.headers();
Map<String, String> headerMap = new LinkedHashMap<>();
List<Map.Entry<String, String>> list = header.entries();

for(Map.Entry<String, String> entry : list) {
    headerMap.put(entry.getKey(), entry.getValue());
}
```
 3、请求体   
 请求体内容可以通过以下代码获取：

 
```
FullHttpRequest req = ...;
ByteBuf buf = req.content();
```
 FullHttpRequest默认将请求体信息保存在了ByteBuf对象中。如果想要转换成byte[]字节数组，可以这样：

 
```
byte[] content = new byte[buf.capacity()];  
buf.readBytes(content);
```
 
### 二、构造并发送HTTP响应

 对于服务端而言、HTTP响应需要在服务端程序里面构建完成并主动发送给客户端。   
 Netty提供了FullHttpResponse类用以构建响应信息。   
 1、新建FullHttpResponse对象   
 Netty提供了FullHttpResponse接口的实现类：DefaultFullHttpResponse   
 该类有两个常用的构造方法，分别是：

 
```
 public DefaultFullHttpResponse(HttpVersion version, HttpResponseStatus status);
 public DefaultFullHttpResponse(HttpVersion version, HttpResponseStatus status, ByteBuf content);
```
 其中：HttpVersion代表HTTP请求协议的版本，比如HttpVersion.HTTP_1_1或HttpVersion.HTTP_1_0   
 HttpResponseStatus代表状态码，如404、200   
 ByteBuf代表响应体的内容

 可以直接new一个DefaultFullHttpResponse对象

 
```
FullHttpResponse response = new DefaultFullHttpResponse(HttpVersion.HTTP_1_1, HttpResponseStatus.OK);
```
 2、向HTTP响应添加响应头信息   
 和HTTP请求类似，可以通过以下代码添加响应头信息

 
```
FullHttpResponse res = ...;
res.headers().add(key, val);
```
 3、添加响应体

 
```
FullHttpResponse res = ...;
ByteBuf buf = res.content();
buf.write(c);

```
 ByteBuf包含了写入字节、字符、字符串等多个方法。   
 最后不要忘了在请求头中添加Content-Length字段。

 4、发送响应   
 直接调用客户端对应的ChannelHandlerContext对象的writeAndFlush并传入FullHttpResponse对象即可。编码器会自动将FullHttpResponse对象转换成标准的HTTP响应

 
```
ChannelHandlerContext ctx = ...;
ctx.writeAndFlush(res);
```
   
  