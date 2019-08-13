---
title: Netty4.0源码解析：TCP粘包半包问题的解决方案
date: 2018-11-03 13:33:40
tags: CSDN迁移
---
 版权声明：尊重原创，转载请注明出处 [ https://blog.csdn.net/abc123lzf/article/details/83626359]( https://blog.csdn.net/abc123lzf/article/details/83626359)   
  ### []()一、引言

 TCP是一个基于流的协议，TCP作为传输层协议并不知道应用层协议的具体含义，它会根据TCP缓冲区的实际情况进行数据包的划分，所以在应用层上认为是一个完整的包，可能会被TCP拆分成多个包进行发送，也有可能把多个小的包封装成一个大的数据包发送，这就是所谓的TCP粘包和半包问题。

 Netty提供了多个进站处理器来处理这个问题：  
 1、LineBasedFrameDecoder：通过换行符来区分每个包  
 2、DelimiterBasedFrameDecoder：通过特殊分隔符来区分每个包  
 3、FixedLengthFrameDecoder：通过定长的报文来分包  
 4、LengthFieldBasedFrameDecoder：跟据包头部定义的长度来区分包

 这几个类都拥有一个共同的父类：ByteToMessageDecoder  
 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20181102203823943.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FiYzEyM2x6Zg==,size_16,color_FFFFFF,t_70)

 需要注意的是，ByteToMessageDecoder的子类不允许使用@Sharable注解，否则在构造阶段会抛出IllegalStateException异常。

 
### []()二、ByteToMessageDecoder

 ByteToMessageDecoder提供了最基本的字节转换为可识别消息的功能，也就是将多个ByteBuf转换为一个可识别的ByteBuf。一般放在ChannelPipeline管道的头部。

 ByteToMessageDecoder持有以下成员变量：

 
```
//每次和其它ByeBuf消息碎片合并后的缓冲区
ByteBuf cumulation;
//合并策略，这里默认为通过一次内存复制操作来完成cumulation和读入的ByteBuf的合并
private Cumulator cumulator = MERGE_CUMULATOR;
//是否仅解码一条消息
private boolean singleDecode;
//是否在没有字节可读时尝试进行获取更多的字节进行解码
private boolean decodeWasNull;
//cumulation是否为null
private boolean first;
//本次解码的状态
private byte decodeState = STATE_INIT;
//当读到多少个零碎的ByteBuf时就将当前cumulation作为坏包丢弃
private int discardAfterReads = 16;
//本次解码读到的ByteBuf的数量
private int numReads;

```
 
##### []()1、channelRead方法

 要了解ByteToMessageDecoder解码机制，我们可以从它的channelRead方法开始分析：

 
```
@Override
public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
    if (msg instanceof ByteBuf) { //接收Channel读到的数据，此时的ByteBuf数据可能是不可读的
    	//构造一个List，用于存放每个ByteBuf解码的结果
        CodecOutputList out = CodecOutputList.newInstance();
        try {
            ByteBuf data = (ByteBuf) msg;
            first = cumulation == null;
            if (first) //如果cumulation为null，就无需进行ByteBuf的合并
                cumulation = data;
            else //否则调用Cumulator的cumulate方法将当前ByteBuf和cumulation合并
                cumulation = cumulator.cumulate(ctx.alloc(), cumulation, data);
            callDecode(ctx, cumulation, out);
        } catch (DecoderException e) {
            throw e;
        } catch (Exception e) {
            throw new DecoderException(e);
        } finally {
        	//如果cumulation中的字节已经全部解码成功，那么将当前ByteToMessageDecoder复位
            if (cumulation != null && !cumulation.isReadable()) {
                numReads = 0;
                cumulation.release();
                cumulation = null;
            //否则cumulation还仍有零散的消息碎片
            } else if (++ numReads >= discardAfterReads) {
                numReads = 0;
                discardSomeReadBytes(); //丢弃已经读取到的字节
            }
            int size = out.size();
            decodeWasNull = !out.insertSinceRecycled();
            fireChannelRead(ctx, out, size);
            out.recycle();
        }
    } else { //如果不是ByteBuf，就忽略这个消息传递给下一个管道
        ctx.fireChannelRead(msg);
    }
}

```
 channelRead方法的执行可以分为以下几个步骤：

  
  * 获取一个CodecOutputList，用于存放每次channelRead方法调用完成后的解码结果  CodecOutputList实现了java.util.List接口，并通过FastThreadLocal存放在InternalThreadLocalMap中，每个线程都默认持有16个CodecOutputList实例，通过CodecOutputList池CodecOutputLists来维护，如果所需的CodecOutputList超出16个，那么会默认实例化一个新的CodecOutputList实例。

  
  * 将当前ByteBuf与之前读到的ByteBuf（成员变量cumulation）进行合并  如果cumulation为空，那么直接将当前ByteBuf引用赋给cumulation  
 如果cumulation不为空，那么将根据成员变量cumulator定义的合并策略进行ByteBuf的合并。  
 Cumulator是ByteToMessageDecoder的内部接口，定义了一个方法：

 
```
ByteBuf cumulate(ByteBufAllocator alloc, ByteBuf cumulation, ByteBuf in);

```
 参数alloc为ByteBuf分配器，用于分配一个新的ByteBuf，可以通过调用ChannelHandlerContext的alloc方法获得。cumulation为原ByteBuf缓冲区，in为需要被合并的ByteBuf缓冲区，其返回值为合并后的ByteBuf缓冲区。

 ByteToMessageDecoder默认定义了2种Cumulator实现类：ByteToMessageDecoder.MERGE_CUMULATOR和ByteToMessageDecoder.COMPOSITE_CUMULATOR。

 MERGE_CUMULATOR的合并策略是通过ByteBufAllocator分配一个大小为cumulation加上in的可读字节数，然后将cumulation和in的数据复制到缓冲区中，所以MERGE_CUMULATOR需要一次内存复制操作。ByteToMessageDecoder默认采用这种策略合并缓冲区。

 COMPOSITE_CUMULATOR的合并策略是通过CompositeByteBuf来完成ByteBuf的合并，它可以持有多个ByteBuf实例，所以不需要进行内存复制操作。但是CompositeByteBuf的索引算法实现较为复杂，可能会比MERGE_CUMULATOR要慢。

  
  * 合并完成后，对合并后的ByteBuf缓冲区（cumulation）的数据进行解码  
```
protected void callDecode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) {
    try {
        while (in.isReadable()) {
            int outSize = out.size(); //注：在第一次循环中outSize为0
            if (outSize > 0) { //如果List的长度大于0，说明已经有解码好的消息
            	//产生一个ChannelRead事件，并将集合out的每个元素传播到管道
                fireChannelRead(ctx, out, outSize);
                out.clear(); //清空这个List
                if (ctx.isRemoved()) //如果已经从管道中移除，那么退出循环结束
                    break;
                outSize = 0;
            }
            //获取可读字节数量
            int oldInputLength = in.readableBytes();
            decodeRemovalReentryProtection(ctx, in, out);
            if (ctx.isRemoved()) //如果this已经从管道移除，那么退出循环
                break;
            if (outSize == out.size()) { //如果集合out元素数量在本次循环中没有改变
                //如果在decodeRemovalReentryProtection没有处理任何数据
                if (oldInputLength == in.readableBytes())
                    break;
                else
                    continue;
            }
            if (oldInputLength == in.readableBytes()) //一般不会发生
                throw new DecoderException(StringUtil.simpleClassName(getClass()) +
                                ".decode() did not read anything but decoded a message.");
            if (isSingleDecode()) //如果仅解码一条消息，那么退出循环
                break;
        }
    } catch (DecoderException e) {
        throw e;
    } catch (Exception cause) {
        throw new DecoderException(cause);
    }
}

```
 callDecode通过一个while循环，不断尝试从cumulation中读入数据。对于每次循环，可以分为以下几个步骤：  
 （1）获取集合out的元素数量，如果集合out元素数量大于0，说明已经有解码成功的消息（ByteBuf对象），那么产生一个channelRead事件并将每个解码好的消息传递到管道的下一个ChannelHandler，然后清空这个out集合。如果元素数量为0，那么就意味着没有解码完成的消息。  
 （2）获取cumulation的可读字节数oldInputLength，并调用decodeRemovalReentryProtection方法对cumulation中的二进制数据进行解码。如果成功从cumulation中分离出可读的消息，那么该方法会往out集合中添加这个消息的ByteBuf对象。  
 （3）检查this有没有从当前管道移除，如果移除则退出循环，方法结束。  
 （4）如果集合out中的元素数量在本次循环中没有改变并且decodeRemovalReentryProtection方法没有处理cumulation中的字节，那么退出循环方法结束。如果集合out中的元素没有改变但是处理了cumulation中的字节，那么重新开始循环。  
 （5）如果仅仅需要解码一条消息，那么退出循环。

 decodeRemovalReentryProtection方法负责调用decode方法尝试对合并后消息进行解码：

 
```
final void decodeRemovalReentryProtection(ChannelHandlerContext ctx, ByteBuf in, List<Object> out)
        throws Exception {
    decodeState = STATE_CALLING_CHILD_DECODE;
    try {
    	//调用抽象方法decode进行解码，由子类进行实现
        decode(ctx, in, out);
    } finally {
    	//如果已经成功将多个ByteBuf转换为一个可识别消息
        boolean removePending = decodeState == STATE_HANDLER_REMOVED_PENDING;
        decodeState = STATE_INIT; //将解码状态修改为STATE_INIT
        if (removePending)
            handlerRemoved(ctx);
    }
}

```
 decode方法为抽象方法，需要子类根据自己的分离消息的策略进行实现。  
 ByteToMessageDecoder规定：如果decode方法成功从cumulation中分离出一条可读的消息，那么会将这个消息添加到集合out中。如果尚未提取出可读的消息，则无需改动集合out中的消息。

 
##### []()2、channelReadComplete方法

 当JDK的SocketChannel已经读完本次客户端发送过来的所有字节后，channelReadComplete方法随即会被调用：

 
```
@Override
public void channelReadComplete(ChannelHandlerContext ctx) throws Exception {
    numReads = 0;
    discardSomeReadBytes(); //丢弃cumulation已经读完的字节
    if (decodeWasNull) {
        decodeWasNull = false;
        if (!ctx.channel().config().isAutoRead())
            ctx.read(); //尝试从Channel获取更多的字节
    }
    //将channelReadComplete事件传递给下一个Handler
    ctx.fireChannelReadComplete();
}

```
 channelReadComplete方法实现比较简单，首先丢弃cumulation中已经读取完成的字节（也就是丢弃0~readIndex范围内的字节）。然后，调用ChannelHandlerContext的read方法请求从Channel通道获取更多的字节。  
 回顾下DefaultChannelPipeline管道模型，read方法最终会调用到管道头结点HeadContext引用的Unsafe对象的beginRead方法，beginRead方法会将OP_READ事件添加到这个JDK Channel感兴趣的事件中。  
 最后，再将ChannelReadComplete事件传递到下一个ChannelHandler。

 
### []()三、FixedLengthFrameDecoder

 FixedLengthFrameDecoder通过定长的报文来分包。在构造FixedLengthFrameDecoder时，需要传入一个int型的变量，代表单个包的字节数。  
 FixedLengthFrameDecoder的实现比较简单：

 
```
public class FixedLengthFrameDecoder extends ByteToMessageDecoder {
    private final int frameLength; //每个包的字节数

    public FixedLengthFrameDecoder(int frameLength) {
        if (frameLength <= 0)
            throw new IllegalArgumentException("frameLength must be a positive integer: " + frameLength);
        this.frameLength = frameLength;
    }

    @Override
    protected final void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) throws Exception {
        Object decoded = decode(ctx, in);
        if (decoded != null)
            out.add(decoded);
    }

    protected Object decode(ChannelHandlerContext ctx, ByteBuf in) throws Exception {
        if (in.readableBytes() < frameLength) {
            return null;
        } else {
            return in.readSlice(frameLength).retain();
        }
    }
}

```
 FixedLengthFrameDecoder直接通过ByteBuf的readSlice方法来截取包。  
 **decode方法执行步骤如下：**  
 （1）如果传入的ByteBuf的刻度字节数小于包的长度，那么方法返回结束  
 （2）否则，调用这个ByteBuf的readSlice方法将当前ByteBuf的缓冲区数组的readIndex ~ readIndex+frameLength处的字节串截取下来生成一个新的ByteBuf实例，然后将这个新ByteBuf的引用计数器加1。这个ByteBuf实例包含的数据就是一个可读的包。然后，将这个ByteBuf实例添加到out集合中。解码完成。

 
### []()四、LineBasedFrameDecoder

 LineBasedFrameDecoder通过换行符"\r\n"或"\n"来划分每个包。  
 LineBasedFrameDecoder提供了两个public构造方法：

 
```
public LineBasedFrameDecoder(final int maxLength) {
    this(maxLength, true, false);
}

public LineBasedFrameDecoder(final int maxLength, final boolean stripDelimiter, final boolean failFast) {
    this.maxLength = maxLength;
    this.failFast = failFast;
    this.stripDelimiter = stripDelimiter;
}

```
 这三个变量的含义是：  
 maxLength：包的最大长度  
 stripDelimiter：是否在分包时去除掉换行符  
 failFast：当字节流长度超过maxLength并且仍然没有找到换行符时，是否向管道传入一个ExceptionCaught事件，并伴随TooLongFrameException异常对象。

 我们来看它的decode方法：

 
```
@Override
protected final void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) throws Exception {
    Object decoded = decode(ctx, in);
    if (decoded != null) {
        out.add(decoded);
    }
}

protected Object decode(ChannelHandlerContext ctx, ByteBuf buffer) throws Exception {
    final int eol = findEndOfLine(buffer); //找到换行符在缓冲区buffer的位置
    if (!discarding) { //如果之前没有包因为找不到换行符并且超出最大长度而被丢弃
        if (eol >= 0) { //如果找到了
            final ByteBuf frame;
            final int length = eol - buffer.readerIndex(); //获取消息的长度
            final int delimLength = buffer.getByte(eol) == '\r'? 2 : 1; //判断换行符是"\r\n"还是"\n"
            if (length > maxLength) { //如果消息长度大于允许的最大长度
                buffer.readerIndex(eol + delimLength); //将刚才读到的数据标记为已读
                fail(ctx, length); //向管道传入ExceptionCaught事件
                return null;
            }
            if (stripDelimiter) { //如果需要移除换行符
                frame = buffer.readSlice(length); //将单个包数据从buffer截取下来
                buffer.skipBytes(delimLength); //将buffer的换行符所在的区域标记为已读
            } else { //如果无需移除换行符，则直接从buffer将包信息截取下来
                frame = buffer.readSlice(length + delimLength);
            }
            return frame.retain(); //将截取下来的ByteBuf的引用计数加1
        } else { //如果没有找到换行符
            final int length = buffer.readableBytes(); //获取buffer可读字节数
            if (length > maxLength) { //如果可读字节数超出了最大包长度
                discardedBytes = length; //记录这段数据的长度
                buffer.readerIndex(buffer.writerIndex()); //将所有数据标记为已读，即忽略这段数据
                discarding = true;
                offset = 0;
                if (failFast) //向管道传入ExceptionCaught事件
                    fail(ctx, "over " + discardedBytes);
            }
            return null;
        }
    } else { //如果有包因为找不到换行符并且超出最大长度而被丢弃
        if (eol >= 0) { //如果此时找到了换行符
        	//丢弃这个换行符之前的所有数据
            final int length = discardedBytes + eol - buffer.readerIndex();
            final int delimLength = buffer.getByte(eol) == '\r'? 2 : 1;
            buffer.readerIndex(eol + delimLength);
            discardedBytes = 0;
            discarding = false;
            if (!failFast) //向管道传入ExceptionCaught事件
                fail(ctx, length);
        } else { //如果依然没有找到换行符，那么仍然丢弃这段数据(通过更改读索引等于写索引的方式)
            discardedBytes += buffer.readableBytes();
            buffer.readerIndex(buffer.writerIndex());
        }
        return null;
    }
}

```
 decode方法的实现其实不难，我们来一步步分析：  
 （1）既然是通过换行符来分包，那么首先第一步是需要找到换行符在缓冲区中的位置，也就是下标。这里调用了findEndOfLine方法：

 
```
private int findEndOfLine(final ByteBuf buffer) {
    int totalLength = buffer.readableBytes(); //获取所有的字节数
    //通过buffer的forEachByte方法找出换行符'\n'的在缓冲区数组的中的位置
    int i = buffer.forEachByte(buffer.readerIndex() + offset, totalLength - offset, ByteBufProcessor.FIND_LF);
    if (i >= 0) { //如果大于等于0，说明找到换行符
        offset = 0;
        //检查换行符类型是不是"\r\n"类型，如果是，则将下标减1，即回退到'\r'的位置
        if (i > 0 && buffer.getByte(i - 1) == '\r') {
            i--;
        }
    } else { //如果小于0，则说明没有找到换行符
        offset = totalLength;
    }
    return i;
}

```
 如果没有找到换行符，那么直接返回-1。如果找到了换行符并且换行符是‘\n’类型，那么返回这个’\n’字符所在的缓冲区数组的下标。如果是"\r\n"类型，那么返回的数组下标指向’\r’

 （2）如果之前没有包因为找不到换行符并且超出最大长度而被丢弃，并且找到了换行符，那么判断这段数据有没有超过最大包长度，如果超出，那么丢弃这段数据，向管道传入ExceptionCaught事件并伴随TooLongFrameException异常对象。如果没有超出，那么将缓冲区的readIndex到换行符这段数据截取下来生成一个新的ByteBuf。这个新的ByteBuf就是一个完整的包，decode将它添加到集合out中后，方法结束。

 如果在缓冲区的readIndex ~ writeIndex范围内没有找到换行符，那么首先判断writeIndex减去readIndex的值有没有超出最大包长度，如果没有超出则什么也不做，方法返回结束。  
 如果超出最大长度，那么会将读索引readIndex移动到writeIndex位置，即丢弃这段数据，并将变量**discarding**标记为true，再下次调用decode方法的时候做特殊处理。

 （3）如果**discarding**变量为true，就说明上次调用decode方法的时候，缓冲区中的可读数据因为找不到换行符并且超过了最大包长度而被丢弃。因为没有找到换行符的原因，所以在本次处理的时候，需要将本次传入的缓冲区数据的readIndex到换行符下标的数据移除，因为无法保证这个包是完整可读的。最后将discarding标记为false。在下次调用decode方法的时候按照第二步的方式正常处理。

 但是，如果本次读取依然没有找到换行符，那么继续丢弃这段数据，discarding变量仍然为true。

 
### []()五、DelimiterBasedFrameDecoder

 DelimiterBasedFrameDecoder可以通过自定义分隔符来区分每个包。  
 DelimiterBasedFrameDecoder提供了6个public构造方法，可以指定4种类型的参数：  
 maxFrameLength：每个包的最大长度  
 stripDelimiter：是否不将分隔符加入到每个包中  
 failFast：当字节流长度超过maxFrameLength并且仍然没有找到换行符时，是否向管道传入一个ExceptionCaught事件，并伴随TooLongFrameException异常对象  
 delimiters：持有分隔符数据的ByteBuf对象，可以指定多个ByteBuf，每个ByteBuf对应一种类型的分隔符

 
```
public DelimiterBasedFrameDecoder(int maxFrameLength, boolean stripDelimiter, boolean failFast, 
		ByteBuf... delimiters) {
    validateMaxFrameLength(maxFrameLength); //maxFrameLength不能小于等于0
    if (delimiters == null)
        throw new NullPointerException("delimiters");
    if (delimiters.length == 0)
        throw new IllegalArgumentException("empty delimiters");
    //如果delimiters有基于换行符的分隔符，并且当前类不是DelimiterBasedFrameDecoder的子类
    if (isLineBased(delimiters) && !isSubclass()) {
        lineBasedDecoder = new LineBasedFrameDecoder(maxFrameLength, stripDelimiter, failFast);
        this.delimiters = null;
    } else { //将这些分隔符ByteBuf复制一份保存到delimiters数组中
        this.delimiters = new ByteBuf[delimiters.length];
        for (int i = 0; i < delimiters.length; i ++) {
            ByteBuf d = delimiters[i];
            validateDelimiter(d); //验证ByteBuf是否可读
            this.delimiters[i] = d.slice(d.readerIndex(), d.readableBytes());
        }
        lineBasedDecoder = null;
    }
    this.maxFrameLength = maxFrameLength;
    this.stripDelimiter = stripDelimiter;
    this.failFast = failFast;
}

```
 DelimiterBasedFrameDecoder持有以下成员变量：

 
```
//存放分隔符，每个ByteBuf代表一个分隔符
private final ByteBuf[] delimiters;
//包的最大允许长度
private final int maxFrameLength;
//包内容是否不包含分隔符
private final boolean stripDelimiter;
//是否在字节过长并且没有读到换行符时向管道传入ExceptionCaught事件
private final boolean failFast;
//在上次调用decode方法时是否有包因为太长导致被丢弃
private boolean discardingTooLongFrame;
private int tooLongFrameLength;
//只有当基于换行符实现时
private final LineBasedFrameDecoder lineBasedDecoder;

```
 同样，我们从decode方法开始解析：

 
```
@Override
protected final void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) throws Exception {
    Object decoded = decode(ctx, in);
    if (decoded != null) {
        out.add(decoded);
    }
}

protected Object decode(ChannelHandlerContext ctx, ByteBuf buffer) throws Exception {
    if (lineBasedDecoder != null)
        return lineBasedDecoder.decode(ctx, buffer);
    int minFrameLength = Integer.MAX_VALUE;
    ByteBuf minDelim = null;
    for (ByteBuf delim: delimiters) { //遍历所有分隔符
        int frameLength = indexOf(buffer, delim); //根据delim定义的分隔符找出分隔符位置
        if (frameLength >= 0 && frameLength < minFrameLength) { //选出离读索引最近的分隔符
            minFrameLength = frameLength;
            minDelim = delim;
        }
    }
	//省略其它代码...
}

```
 DelimiterBasedFrameDecoder的解码逻辑和LineBasedFrameDecoder非常类似。只是DelimiterBasedFrameDecoder在寻找分隔符的方式上和LineBasedFrameDecoder有所区别：  
 因为DelimiterBasedFrameDecoder可以定义多个分隔符，所以，DelimiterBasedFrameDecoder通过遍历所有的分隔符并与传入的缓冲区数据进行比对，找出离缓冲区读索引最近的分隔符。  
 indexOf方法可以找出分隔符在指定缓冲区中的位置：

 
```
private static int indexOf(ByteBuf haystack, ByteBuf needle) {
	//遍历缓冲区中的字节数据
    for (int i = haystack.readerIndex(); i < haystack.writerIndex(); i ++) {
        int haystackIndex = i;
        int needleIndex;
        //遍历分隔符数据
        for (needleIndex = 0; needleIndex < needle.capacity(); needleIndex ++) {
        	//如果比对不成功则退出循环
            if (haystack.getByte(haystackIndex) != needle.getByte(needleIndex)) {
                break;
            } else { //如果这个字节比对上
                haystackIndex++;
                //如果此时haystackIndex已经到达缓冲区可读数据尾部
                if (haystackIndex == haystack.writerIndex() &&
                    needleIndex != needle.capacity() - 1) {
                    return -1;
                }
            }
        }
        //如果全部比对成功，返回分隔符所在的下标
        if (needleIndex == needle.capacity()) {
            // Found the needle from the haystack!
            return i - haystack.readerIndex();
        }
    }
    return -1;
}

```
 
### []()六、LengthFieldBasedFrameDecoder

 LengthFieldBasedFrameDecoder支持在包首部加入一段描述长度信息，也就是长度字段，根据这些信息来分离出可读消息。  
 在介绍原理之前，我们先来回顾一下具体的使用方法：

 在使用LengthFieldBasedFrameDecoder时，需要根据消息的规范来指定下面**4种参数**：  
 **lengthFieldOffset**：长度字段的偏移量，也就是完整的消息中标记消息长度的字节离消息的首部相差多少个字节。  
 **lengthFieldLength**：长度字段的长度，也就是长度字段本身占据的容量。只允许值为1、2、3、4、8，使用其它值会在解码过程中抛出DecoderException异常。虽然名义上允许8个字节(long)的长度信息，但实际上消息长度最大不能超过int，因为maxFrameLength为int型。  
 **lengthAdjustment**：长度字段的修正值，如果长度字段给出的长度包含了长度字段本身占据的空间，那么就需要进行修正。比如长度字段内容为0x0E，消息长度为12，就需要指出lengthAdjustment = -2，即实际内容长度为0x0C。  
 **initialBytesToStrip**：对解码后的消息进行第一次删除的字节数，比如我们在传入下一个管道前，需要删去消息的头部的长度字段，如果这个长度字段长度为2，那么只需要指定这个值等于2即可。

 如果仍然有疑问，可以看看下面几个示例：  
 **场景1：包的第一个字段只描述了长度信息，后面为消息的内容**

 
```
+--------+----------------+
| Length | Actual Content |
| 0x000E | "HELLO, WORLD" | 
+--------+----------------+

```
 在这种情况下，我们需要指定：  
 lengthFieldOffset = 0  
 lengthFieldLength = 2 （长度字段的字节数为2）  
 lengthAdjustment = 0  
 initialBytesToStrip = 0

 解码后包内容不变，依旧包含包首部的长度信息。如果需要移除包首部的长度信息，我们可以指定：  
 lengthFieldOffset = 0  
 lengthFieldLength = 2  
 lengthAdjustment = 0  
 initialBytesToStrip = 2 （删去包首部的2个字节，也就是正好删去了长度字段）  
 这样，解码后传递给下一个管道的消息就变为：

 
```
+----------------+
| Actual Content |
| "HELLO, WORLD" | 
+----------------+

```
 **场景2：对于某些协议，包首部的长度字段为包的总大小（包内容字节数加上包首部字节数）：**

 
```
BEFORE DECODE (14 bytes)         AFTER DECODE (14 bytes)
+--------+----------------+      +--------+----------------+
| Length | Actual Content |----->| Length | Actual Content |
| 0x000E | "HELLO, WORLD" |      | 0x000E | "HELLO, WORLD" |
+--------+----------------+      +--------+----------------+

```
 这里的length(0x0E)等于消息内容HELLO,WORLD(0x0C)加上length包含的字节数(0x02)  
 对于这样的协议，我们需要制定：  
 lengthFieldOffset = 0  
 lengthFieldLength = 2（长度字段占据2个字节）  
 lengthAdjustment = -2 （长度字段数据包含了长度字段本身占据的空间）  
 initialBytesToStrip = 0

 **场景3：标识长度的字段位于消息头部或者尾部，就需要指定lengthFieldOffset：**

 
```
BEFORE DECODE (17 bytes)                      AFTER DECODE (17 bytes)
+----------+----------+----------------+      +----------+----------+----------------+
| Header 1 |  Length  | Actual Content |----->| Header 1 |  Length  | Actual Content |
|  0xCAFE  | 0x00000C | "HELLO, WORLD" |      |  0xCAFE  | 0x00000C | "HELLO, WORLD" |
+----------+----------+----------------+      +----------+----------+----------------+

```
 lengthFieldOffset = 2 （消息头占据了2个字节，长度字段在消息的第3个字节开始，所以为2）  
 lengthFieldLength = 3 （这里长度字段有3个字节）  
 lengthAdjustment = 0  
 initialBytesToStrip = 0

 **场景4：长度字段夹在两个消息头中间，如果想要忽略长度字段以及前面的其它消息头字段，可以通过设置initialBytesToStrip来解决。**

 
```
BEFORE DECODE (16 bytes)                       AFTER DECODE (13 bytes)
+------+--------+------+----------------+      +------+----------------+
| HDR1 | Length | HDR2 | Actual Content |----->| HDR2 | Actual Content |
| 0xCA | 0x000C | 0xFE | "HELLO, WORLD" |      | 0xFE | "HELLO, WORLD" |
+------+--------+------+----------------+      +------+----------------+

```
 lengthFieldOffset = 1 （消息头占据1个字节）  
 lengthFieldLength = 2 （长度字段占据2个字节）  
 lengthAdjustment = 1 （HDR2字段的长度）  
 initialBytesToStrip = 3 （删去包首部的3个字节，也就是HDR1 + Length）

 
##### []()源码解析

 按照惯例从decode方法开始：

 
```
@Override
protected final void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) throws Exception {
    Object decoded = decode(ctx, in);
    if (decoded != null)
        out.add(decoded);
}

protected Object decode(ChannelHandlerContext ctx, ByteBuf in) throws Exception {
    if (discardingTooLongFrame)
        discardingTooLongFrame(in);
    //如果可读字节数小于消息头长度加长度字段长度(lengthFieldEndOffset = lengthFieldOffset + lengthFieldLength)
    if (in.readableBytes() < lengthFieldEndOffset)
        return null;
    //计算出除去消息头的消息首部在缓冲区的位置
    int actualLengthFieldOffset = in.readerIndex() + lengthFieldOffset;
    //获取长度字段的数据
    long frameLength = getUnadjustedFrameLength(in, actualLengthFieldOffset, lengthFieldLength, byteOrder);
    //如果长度小于0，那么抛出CorruptedFrameException异常
    if (frameLength < 0)
        failOnNegativeLengthField(in, frameLength, lengthFieldEndOffset);
    //根据长度字段修正值计算出消息(整个包)的实际长度
    frameLength += lengthAdjustment + lengthFieldEndOffset;
    //如果整个包的长度比lengthFieldEndOffset还小，那么抛出异常
    if (frameLength < lengthFieldEndOffset)
        failOnFrameLengthLessThanLengthFieldEndOffset(in, frameLength, lengthFieldEndOffset);
    //如果消息长度大于最大允许的长度
    if (frameLength > maxFrameLength) {
        exceededFrameLength(in, frameLength); //丢弃这个消息
        return null;
    }
    int frameLengthInt = (int) frameLength; //不会发生溢出，因为maxFrameLength为int型变量
    //如果可读字节数小于消息长度，则说明还尚未读完消息，返回null，等待下次decode方法的调用
    if (in.readableBytes() < frameLengthInt)
        return null;
    //如果需要删除的字节数大于消息长度，那么抛出异常
    if (initialBytesToStrip > frameLengthInt)
        failOnFrameLengthLessThanInitialBytesToStrip(in, frameLength, initialBytesToStrip);
    //删除消息头部指定的字节数
    in.skipBytes(initialBytesToStrip);
    int readerIndex = in.readerIndex(); //获取缓冲区现在的读索引
    //计算出删去消息头部initialBytesToStrip字节后的消息长度
    int actualFrameLength = frameLengthInt - initialBytesToStrip; 
    //将消息从缓冲区截取下来生成一个ByteBuf对象
    ByteBuf frame = extractFrame(ctx, in, readerIndex, actualFrameLength);
    //在缓冲区中将这段数据标记为已读
    in.readerIndex(readerIndex + actualFrameLength);
    return frame;
}

```
 decode方法执行步骤可以分为以下几个步骤：  
 1、首先判断在上次调用decode方法的时候有没有因为包过长而被丢弃，如果有，就调用discardingTooLongFrame方法处理（搭配第4步一起看）：

 
```
private void discardingTooLongFrame(ByteBuf in) {
    long bytesToDiscard = this.bytesToDiscard; //获取需要忽略的字节数
    //比较缓冲区可读的字节数和需要忽略的字节数，选出一个最小值
    int localBytesToDiscard = (int) Math.min(bytesToDiscard, in.readableBytes());
    in.skipBytes(localBytesToDiscard); //忽略这段数据
    bytesToDiscard -= localBytesToDiscard; //算出实际忽略的字节数
    this.bytesToDiscard = bytesToDiscard; //将剩余需要忽略的字节数记录下来
    failIfNecessary(false);
}

```
 2、如果缓冲区可读字节数量小于消息头长度加长度字段长度(lengthFieldEndOffset)，那么则说明消息不完整或者有问题，直接返回null。

 3、获取消息的长度字段数据，如果小于0，那么调用failOnNegativeLengthField方法处理：

 
```
private static void failOnNegativeLengthField(ByteBuf in, long frameLength, int lengthFieldEndOffset) {
    in.skipBytes(lengthFieldEndOffset);
    throw new CorruptedFrameException("negative pre-adjustment length field: " + frameLength);
}

```
 该方法直接丢弃这段数据并抛出CorruptedFrameException异常。

 4、根据长度字段修正值（lengthAdjustment）计算出消息（整个包）的实际长度，如果这个消息的长度大于最大允许的消息长度（maxFrameLength），那么调用exceededFrameLength方法处理并返回null：

 
```
private void exceededFrameLength(ByteBuf in, long frameLength) {
	//比较这个消息的长度和缓冲区中实际可读的字节数
    long discard = frameLength - in.readableBytes();
    //将这个过长消息的大小记录下来
    tooLongFrameLength = frameLength; 
    if (discard < 0) { //如果缓冲区实际可读的消息大于这个包的长度，那么只将这个包的信息标记为已读
        in.skipBytes((int) frameLength);
    } else {
    	//否则在下次调用decode方法时做特殊处理
        discardingTooLongFrame = true;
        //将这个数记录下来，作为下次调用decode方法时忽略的字节
        bytesToDiscard = discard;
        //忽略缓冲区中所有的数据
        in.skipBytes(in.readableBytes());
    }
    //根据变量failFast的策略决定是否抛出TooLongFrameException异常
    failIfNecessary(true);
}

```
 5、如果缓冲区可读字节数小于消息长度，则说明消息尚未读完，返回null，等待下次decode方法的调用。  
 6、如果需要删除的字节数(initialBytesToStrip)大于消息长度，那么调用failOnFrameLengthLessThanInitialBytesToStrip方法处理：

 
```
private static void failOnFrameLengthLessThanLengthFieldEndOffset(ByteBuf in,
				long frameLength, int lengthFieldEndOffset) {
	in.skipBytes(lengthFieldEndOffset);
    throw new CorruptedFrameException("Adjusted frame length (" + frameLength + ") is less " +
              "than lengthFieldEndOffset: " + lengthFieldEndOffset);
}

```
 该方法会跳过lengthFieldEndOffset个字节然后抛出CorruptedFrameException异常。

 7、接着，根据变量initialBytesToStrip，从缓冲区中删除消息头部（通过增加读索引的方式），然后调用extractFrame方法将消息从缓冲区中截取下来生成一个新的ByteBuf对象：

 
```
protected ByteBuf extractFrame(ChannelHandlerContext ctx, ByteBuf buffer, int index, int length) {
    return buffer.slice(index, length).retain();
}

```
 8、截取成功后，将缓冲区中的这段数据标记为已读，然后将这个ByteBuf加入到集合out中。

   
  