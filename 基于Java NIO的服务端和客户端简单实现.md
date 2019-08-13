---
title: 基于Java NIO的服务端和客户端简单实现
date: 2018-05-24 19:30:16
tags: CSDN迁移
---
 版权声明：尊重原创，转载请注明出处 [ https://blog.csdn.net/abc123lzf/article/details/80435511]( https://blog.csdn.net/abc123lzf/article/details/80435511)   
  ### 一、服务端

 
#### （1）初始化

 在服务器启动前我们需要进行一系列初始化操作，包括创建选择器、ServerSocketChannel通道、绑定端口。   
 下面是初始化操作：

 
```
private final Selector selector;
private final ServerSocketChannel ssc;
private final SelectionKey mainKey;
public ServerNIOThread(int port) throws IOException {
    //创建选择器
    selector = Selector.open();
    //创建通道
    ssc = ServerSocketChannel.open();
    //将该通道设置为非阻塞模式，该步骤必须在设置端口之前设置
    ssc.configureBlocking(false);
    //将该通道绑定到指定端口
    ssc.socket().bind(new InetSocketAddress(port));
    //将通道注册到选择器上，并设置为OP_ACCEPT模式(代表该通道对连接事件感兴趣)
    mainKey = ssc.register(selector, SelectionKey.OP_ACCEPT);
}
```
 建议在服务端对象里面创建一个线程池对象，用于管理发送数据包和接收数据包的线程。例如：

 
```
private final ExecutorService exec = Executors.newCacheThreadPool();
```
 
#### （2）服务器运行

 当初始化操作完成后，我们就可以启动服务器处理线程了。

 
```
@Override
public void run() {
    try {
        execute();
    } catch (IOException e) {
        e.printStackTrace();
    }
}

private void execute() throws IOException{
        //当选择器检测到有IO事件等待处理时
        //selector.select()的返回值代表该选择器等待处理的IO事件数量，该方法会一直阻塞直到有待处理IO事件
        while(selector.select() > 0) {
            //获取选择器待处理的IO事件集合，并获取该集合迭代器用于遍历
            Iterator<SelectionKey> it = selector.selectedKeys().iterator();
            while(it.hasNext()) {
                SelectionKey key = it.next();
                //如果该IO事件是可接受连接的（代表有客户端尝试进行连接）
                if(key.isAcceptable()) {
                    //调用ServerSocketChannel实例的accpet方法，会返回一个对应客户端的SocketChannel实例
                    SocketChannel socket = ssc.accept();
                    System.out.println("客户端连入，IP：" + socket.socket().getInetAddress().getHostAddress());
                    //将对应客户端的SocketChannel设置为非阻塞模式
                    socket.configureBlocking(false);
                    //将客户端SocketChannel注册到选择的上，并设置为选择器对该通道的读事件感兴趣
                    socket.register(selector, SeleionKey.OP_READ);
                    //去除对连接事件感兴趣（不然会死循环）
                    key.interestOps(SelectionKey.OP_ACCEPT);
                } else if(key.isReadable()) {    //如果该IO事件是可读的（代表客户端向服务器发送了一个数据包）
                    //调用处理线程
                    exec.submit(new ReadTCPPackage(key));
                }
                //事件处理完毕，去除该SelectionKey（不然会死循环）
                it.remove();
            }
        }
    }
}

```
 
#### (3)处理可读事件线程

 当服务器接收到了一个数据包后，我们需要对这个数据包进行识别、处理。这里建议写一个内部类来负责这件事。

 
```
private static class ReadTCPPackage implements Runnable {
    private final SelectionKey key;

    private ReadTCPPackage(SelectionKey key){
        System.out.println("已收到客户端TCP包");
        this.key = key;
    }

    @Override
    public void run() {
        //获取客户端的SocketChannel
        SocketChannel socket = (SocketChannel)key.channel();
        //新建一个1024大小的字节缓冲区
        ByteBuffer buffer = ByteBuffer.allocate(1024);
        buffer.clear();
        try {
            //将客户端发送过来的数据包存入缓冲区
            int count = socket.read(buffer);
            if(count > 0) {
                ByteArrayInputStream input = new ByteArrayInputStream(buffer.array());
                //如果客户端仅发送字符串的话也可采用字符输入流，根据具体需求选择
                ObjectInputStream ois = new ObjectInputStream(input);

                try {
                    Object obj = ois.readObject();
                    //调用处理、识别对象的方法...
                    //ObjDecode.decode(obj);
                } catch (ClassNotFoundException e) {
                    e.printStackTrace();
                }
                key.interestOps();
            } else if(count == -1) { //当客户端关闭时可能会造成buffer缓冲区无数据可读，这时count会等于-1
                System.out.println("客户端已断开连接");
                key.cancel();
                socket.close();
            }
        }catch (IOException e) {  //如果抛出一个IO异常，则说明客户端已断开连接
            System.out.println("IOException:客户端已断开连接");
            //取消选择器对该通道的注册
            key.cancel();
            try {
                //关闭客户端对应的SocketChannel
                socket.close();
            } catch (IOException e1) {
                e1.printStackTrace();
            }
        }
    }
}
```
 
### (4)向客户端发送数据包

 向其中一个客户端发送数据包必须要知道对应客户端的SocketChannel。   
 下面是实现方法：

 
```
private static class SendTCPPackage implements Runnable {
    private final SocketChannel clientSocket;
    private final Object tcpPackage;

    private SendTCPPackage(SocketChannel clientSocket, Object tcpPackage) {
        this.clientSocket = clientSocket;
        this.tcpPackage = tcpPackage;
    }

    @Override
    public void run() {
        System.out.println("向客户端发送TCP包");
        try {
            //创建一个字节输出流并采用对象输出流包装，将对象写入输出流即可。
            //如果仅发送字符串的话也可采用字符输出流，根据具体需求选择
            ByteArrayOutputStream byteOut = new ByteArrayOutputStream();
            ObjectOutputStream oos = new ObjectOutputStream(byteOut);
            oos.writeObject(tcpPackage);
            oos.flush();    
            //向客户端发送对象，这里需要使用到NIO的ByteBuffer
            clientSocket.write(ByteBuffer.wrap(byteOut.toByteArray()));
        }catch (IOException e) {
            e.printStackTrace();
            System.out.println("与服务器断开连接");
            System.exit(0);
        }
    }
}
```
 
#### （5）注意事项

 建议不要同时发送两个数据包，因为这样有很大概率会产生一个粘包问题（即两个数据包混在一起），客户端如果接收到这样一个数据包会无法解析。因为NIO可能会将一个数据包拆分成几块用不同线程异步发送，如果一定要同时发送，建议对其中一个线程采用Thread.sleep(200)来解决，或者可以采用Netty框架来解决。   
   


 
### 二、客户端

 
#### （1）初始化

 客户端和服务端不同，客户端不需要ServerSocketChannel来管理连接，只需要一个SocketChannel对象即可建立起对服务器的连接。   
 下面是初始化的代码:

 
```
//创建SocketChannel对象
SocketChannel clientSocket = SocketChannel.open();
//设置为非阻塞模式
clientSocket.configureBlocking(false);
//创建选择器
Selector selector = Selector.open();    
//将SocketChannel对象注册到选择器上，并设置为选择器对连接事件感兴趣
clientSocket.register(selector, SelectionKey.OP_CONNECT);
//与服务器连接
clientSocket.connect(new InetSocketAddress(host, port));
```
 和服务端初始化过程类似，需要在启动连接前创建选择器和通道，并将通道设置为**非阻塞模式**然后向选择器注册该通道。

 
#### （2）客户端运行

 
```
@Override
public void run() {
    try{
        //当选择器检测到有IO事件等待处理时
        while(selector.select() > 0){
            //获取选择器待处理的IO事件集合，并遍历
            Iterator<SelectionKey> it = selector.selectedKeys().iterator();
            while(it.hasNext()){
                SelectionKey key = it.next();
                it.remove();
                //如果是可连接的(这里和服务端不同，服务端为isAcceptable，代表可接受连接的)
                if(key.isConnectable()){
                    SocketChannel clientSocket = (SocketChannel)key.channel();
                    //这里需要检测是否完成连接
                    if(clientSocket.finishConnect()){
                        //连接成功后需要将选择器设置为对该通道的读事件感兴趣（不然同样会无限循环）
                        key.interestOps(SelectionKey.OP_READ);
                    }else{
                        System.out.println("连接失败");
                        System.exit(1);
                    }
                }else if(key.isReadable()){
                    exec.submit(new ReadTCPPackage(key));
                }
            }
        }
    }catch(IOException e){
        try {
            clientSocket.close();
        } catch (IOException e1) {
            e1.printStackTrace();
        }
        e.printStackTrace();
    }
}
```
 
### （3）处理可读事件线程

 这里同样建立一个内部类来处理服务器发送过来的数据包。

 
```
private static class ReadTCPPackage implements Runnable{
    private final SelectionKey key;

    private ReadTCPPackage(SelectionKey key){
        this.key = key;
    }

    @Override
    public void run(){
        System.out.println("收到服务器TCP包");
        //获取对应的Socket通道
        SocketChannel socket = (SocketChannel)key.channel();
        //创建一个字节缓冲区
        ByteBuffer buffer = ByteBuffer.allocate(1024);
        buffer.clear();

        try{
            if(socket.read(buffer) > 0) {
                ByteArrayInputStream input = new ByteArrayInputStream(buffer.array());
                ObjectInputStream ois = new ObjectInputStream(input);
                try {
                    Object obj = ois.readObject();
                    //调用解析对象方法...
                    //ObjDecode.decode(obj);
                } catch (ClassNotFoundException e) {
                    e.printStackTrace();
                }
                key.interestOps();
            }catch(IOException e){ //如果连接出现异常则会抛出IOException
                e.printStackTrace();
                System.out.println("与服务器断开连接");
                System.exit(0);
        }
    }
}
```
 
#### （4）向服务器发送数据包

 向服务器发送数据包需要获取对应的SocketChannel

 
```
private static class SendTCPPackage implements Runnable{
    private final SocketChannel clientSocket;
    private final Object tcpPackage;

    private SendTCPPackage(SocketChannel clientSocket, Object tcpPackage){
        System.out.println("向客户端发送TCP包");
        this.clientSocket = clientSocket;
        this.tcpPackage = tcpPackage;
    }

    @Override
    public void run() {
        try {
            //这里和服务端类似，不再赘述
            ByteArrayOutputStream byteOut = new ByteArrayOutputStream();
            ObjectOutputStream oos = new ObjectOutputStream(byteOut);
            oos.writeObject(tcpPackage);
            oos.flush();

            clientSocket.write(ByteBuffer.wrap(byteOut.toByteArray()));
        }catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```
 
### 三、基于NIO的聊天软件参考源码

 如果大家对NIO依然有很多疑问的话可以下载我的源码学习   
 服务端用了MyBatis框架，如果对MyBatis框架不了解的话改成JDBC也是很简单的   
 链接: [https://pan.baidu.com/s/1cDDoZZ-ui2TNolMNWXy6JQ](https://pan.baidu.com/s/1cDDoZZ-ui2TNolMNWXy6JQ) 密码: bw56

 
### 四、与阻塞式IO对比

 Java NIO最大的特点就是可以一个线程通过一个Selector管理多个客户端对应的SocketChannel，而阻塞式IO只能一个线程对应一个客户端连接，要知道线程是很贵重的资源，每个线程需要一定的内存，线程调度同样需要时间，创建过多的线程非常不利于高并发的实现。而Java非阻塞式IO很好的解决了这个问题。如果大家对这方面有兴趣的话建议可以学习一下Netty、Mina之类的框架。

   
  