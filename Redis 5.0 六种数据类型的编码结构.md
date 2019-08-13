---
title: Redis 5.0 六种数据类型的编码结构
date: 2019-04-24 11:22:08
tags: CSDN迁移
---
 版权声明：尊重原创，转载请注明出处 [ https://blog.csdn.net/abc123lzf/article/details/89440006]( https://blog.csdn.net/abc123lzf/article/details/89440006)   
  #### []()字符串类型（String）

 
--------
 字符串类型是最基本的Redis数据类型，它可以存储任何类型的字符串或者二进制类型的数据，其底层实现有三种方式：  
 1、通过 `SDS` （简单动态字符串）实现，其内部编码表示为 `REDIS_ENCODING_RAW` 或者 `REDIS_ENCODING_EMBSTR` 。  
 2、 `long` 类型，当字符串可以用数字表示时，其内部编码表示为 `REDIS_ENCODING_INT` 。

 
##### []()1、简单动态字符串（SDS）

 []()简介 Redis并没有采用C语言的传统字符串表示方式（ `char*` 或者 `char[]` ），而是自己定义了一个成为简单动态字符串（simple dynamic string，即 `SDS` ）类型，这种字符串类型可以方便地实现字符串长度的变更、长度查询及其二进制安全。

 例如客户端执行命令：

 
```
redis> SET key "value"
OK

```
 则会在Redis中生成一个键值对，其中key和value都属于 `SDS` 类型。  
 除了用作键值对以外，Redis内部的各种缓冲区也是基于 `SDS` 实现的。

 []()定义 `SDS` 的定义在 `sds.h` 中：

 
```
struct sdshdr {
	int len;	//buf数组的长度
	int free;	//buf数组剩余的长度
	char buf[]; //记录数据的数组
};

```
 以字符串"Hello"为例， `sdshdr` 结构体在内存中的表现形式如下：  
 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190421213722582.png)  
  `SDS` 遵循了C语言中字符串以’\0’结尾的习惯，其成员变量 `len` 不会将末尾的’\0’记录在内，并且会主动为空字符预留1个字节。这样的好处在于可以重用一部分C语言当中的字符串操作函数。

 []()相比C字符串的优点 **1、可以以 `O(1)` 的时间复杂度计算字符串长度**  
 C字符串本身是以数组的形式存储的，它并不会像Java那样会记录数组的长度。为了获取C字符串的长度，程序必须从数组的首地址依次往后遍历直到找到空字符为止，其时间复杂度为 `O(N)` 。对于Redis的 `SDS` 来说，获取其字符串长度只需要返回成员变量 `len` 就好了。

 **2、可以防止缓冲区溢出（数组越界）**  
 在 `string.h` 定义的字符串拼接函数 `char* strcat(char* dest, const char* src)` 中，如果 `dest` 字符串的不足以容纳 `src` 字符串，那么就会产生缓冲区溢出（即数组越界）。  
  `SDS` 则杜绝了这种情况的发生，Redis专门为 `SDS` 的操作定制了一套API，例如 `SDS` 的字符串拼接函数 `sdscat` 中，sds会检查 `SDS` 是否有充足的空间容纳，如果没有足够的空间，那么 `sdscat` 会重新给 `SDS` 分配一个足够长的数组，并将原先数组的内容复制进去，然后再进行拼接操作。

 如果在操作 `SDS` 的时候需要对 `buf` 数组长度进行修改的时候，Redis不仅会为 `SDS` 分配所必须的空间，还会预留一定的未使用空间，这样可以减少在频繁修改的字符串的时候内存重分配的次数。

 **3、二进制安全**  
 传统C字符串除了末尾可以为空字符外，字符串中不可包含空字符，这种限制使得C字符串只可保存文本，不可存储二进制数据。为了确保Redis能够适用更多的场景， `SDS` 的API都是二进制安全的，其API都会以处理二进制的方式来处理 `SDS` 中的数据。

 []()RAW和EMBSTR的区别 `RAW` 和 `EMBSTR` 都属于 `SDS` ，在一个字符串的长度小于44字节的时候，其默认类型为 `EMBSTR` ，当长度大于44字节的时候便会转换为 `RAW` 形式。它们在内存中的表现形式为：  
 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190422184458371.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FiYzEyM2x6Zg==,size_16,color_FFFFFF,t_70)  
 可以看出， `EMBSTR` 在内存中的格局是连续的，其指针 `ptr` 指向 `buf` 的首地址。而 `RAW` 在内存中不是连续的，其指针 `ptr` 指向 `sdshdr` 结构体实例。

 使用 `EMBSTR` 的形式保存字符串有3大好处：  
 1、创建 `EMBSTR` 编码形式的字符串所需内存分配次数从 `RAW` 的两次降低为一次。  
 2、释放 `EMBSTR` 字符串对象只需要调用一次内存释放函数， `RAW` 则需要两次。  
 3、 `EMBSTR` 和Redis对象结构体在内存当中是连续的，可以充分利用CPU缓存带来的优势。

 此外， `EMBSTR` 是只读的，当我们尝试修改它的时候，它会转换为 `RAW` 编码形式，再对其进行修改，修改完成后就不会再变成 `EMBSTR` 编码的形式了。

 
##### []()2、整数编码

 整数编码适合字符串可以转换为数字的情况，它的内存格局如下：  
 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190422203609897.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FiYzEyM2x6Zg==,size_16,color_FFFFFF,t_70)  
 可以看出，其指针 `ptr` 直接指向了一个数字，可以通过 `INCR` 等命令对它进行运算，此时并不会改变它的编码格式，如果尝试对它进行字符串追加操作，那么它就会转换为 `RAW` 编码格式，此时执行 `INCR` 等命令便会报错。

 

 
#### []()列表类型（List）

 
--------
 列表类型可以在一个对象中存储多个数据（字符串），这些数据是有序的。在Redis内部，列表的实现有两种方式：  
 1、压缩列表，当一个列表只含有少量的数据的时候会采用这个编码，表示为 `REDIS_ENCODING_ZIPLIST` 。  
 2、双端链表，当一个列表含有比较多的数据时候会采用此编码，表示为 `REDIS_ENCODING_LINKEDLIST` 。

 
##### []()1、压缩列表

 []()简介 压缩列表不仅使用在列表类型，哈希类型（ `Hash` ）、有序集合（ `ZSet` ）类型在数据量较少的时候也会采用这个编码形式。压缩列表是Redis为了节约内存空间而开发的，它是由一系列特殊编码的连续内存块组成的有序数据结构。一个压缩列表可以保存多个节点，每个节点可以存储字符串或者二进制数据。

 []()数据结构 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190422211744173.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FiYzEyM2x6Zg==,size_16,color_FFFFFF,t_70)

 
     属性名     | 大小      | 作用                                          
     ------- | ------- | -------------------------------------------- 
     zlbytes | 4 Bytes | 记录整个压缩列表所占用的字节数                             
     zltail  | 4 Bytes | 记录压缩列表起始地址到表尾节点的偏移量，通过它可以快速找到表尾节点，无需遍历整个压缩列表
     zllen   | 2 Bytes | 记录压缩列表节点数量                                  
     entryX  | 不确定     | 压缩列表的各个节点                                   
     zlend   | 1 Byte  | 其值为0xFF，标记压缩列表的尾端                           

对于每个节点 `Entry` 具有以下属性：

 
     属性名                   | 大小                        | 作用               
     --------------------- | ------------------------- | ----------------- 
     previous_entry_length | 1 Byte 或 5 Bytes          | 记录压缩列表前一个节点的占用的字节
     encoding              | 1 Byte 或 2 Bytes或 5 Bytes | 记录节点数据的类型及其长度    
     content               | 不定                        | 记录数据本身           

如果前一节点长度小于254字节，那么 `previous_entry_length` 长度为1字节，如果大于254字节，那么长度为5字节，其中前1个字节为0xFE，后4个字节才记录长度。

 对于 `encoding` 属性，如果其值的最高两位是以11开头，那么表示数据类型为整数。如果最高位是00、01、10，那么数据类型为字节数组，00表示 `encoding` 本身占用1字节（可以表示小于63字节的数据），01表示占用2字节（可以表示小于16383字节的数据），10表示占用5字节（可以表示小于4294967295字节的数据）。

 []()2、双端链表 []()简介 除了采用压缩列表实现列表以外，还可通过双向链表的方式来实现列表。

 []()定义 Redis通过 `adlist.h` 中的 `list` 结构体定义了双向链表：

 
```
typedef struct list {
	//头结点
	listNode *head;
	//尾结点
	listNode *tail;
	//列表长度
	unsigned long len;
	//指向节点值复制的函数
	void *(*dup)(void *ptr);
	//指向节点值释放的函数
	void (*free)(void *ptr);
	//指向节点值对比的函数
	void (*match)(void *ptr, void *key);
} list;

```
 其中， `listNode` 结构体定义了双向链表中的结点：

 
```
typedef struct listNode {
	//前驱指针
	struct listNode *prev;
	//后驱指针
	struct listNode *tail;
	//节点的值
	void *value;
} listNode;

```
 结构图：  
 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190423145725170.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FiYzEyM2x6Zg==,size_16,color_FFFFFF,t_70)

 []()特性 Redis的链表特性可以总结如下：  
 1、双端：获取头结点和尾结点的时间复杂度都是 `O(1)` 。  
 2、无环：表头结点 `prev` 指针和表尾结点 `next` 指针都指向 `NULL` 。  
 3、带有长度计数器：可以以 `O(1)` 的时间复杂度获取数据  
 4、多态：链表结点通过 `void*` 指针指向存储的数据。如果链表用于列表存储数据，那么只会指向一个 `SDS` 字符串。

 

 
#### []()哈希类型（Hash）

 
--------
 哈希类型可以在一个键值对中存储一个哈希表，可以根据 `Key` 的值快速找到其 `Value` ，它是一个无序的数据结构。在Redis内部，哈希类型的实现有两种方式：  
 1、基于压缩列表实现，即 `REDIS_ENCODING_ZIPLIST` 。  
 2、基于字典实现，即 `REDIS_ENCODING_HT` 。

 
##### []()1、压缩列表

 压缩列表除了可以保存列表类型，同样可以保存哈希类型，在这种情况下其数据是有序的。其数据保存方式与列表类似，每一个键值对需要占用两个 `Entry` ，前一个 `Entry` 用来保存 `Key` ，后一个 `Entry` 用来保存 `Value` ，这两个 `Entry` 紧挨在一起。  
 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190423203457799.png)

 
##### []()2、字典

 []()简介 Redis的字典类型使用基于拉链法的哈希表实现，一个哈希表中有多个哈希表节点，每个哈希表节点保存了一个键值对。

 []()定义 Redis字典的结构体定义在头文件 `dict.h` 中：

 
```
typedef struct dictht {
	dictEntry **table; 		//哈希表节点数组
	unsigned long size;		//数组长度
	unsigned long sizemask; //掩码，等于size - 1
	unsigned long used;		//持有的节点数量
} dictht;

```
 `table` 是一个 `dictEntry` 指针数组，每一个 `dictEntry` 保存了一个键值对， `size` 记录了数组的长度，其大小始终为2的整数次幂， `sizemask` 等于 `size - 1` ，作用是在查找键值对时计算索引值，因为在数组长度为2的整数次幂时，其 `hash & sizemask == hash % size` 。 `used` 变量等于持有的节点数量。

 
```
typedef struct dictEntry {
	void *key;
	union {
		void *val;
		uint64_t u64;
		int64_t s64;
	} v;
	struct dictEntry *next;
} dictEntry;

```
 `key` 属性保存着键值对中的键，联合 `v` 保存着键值对中的值，其值的类型可以为 `SDS` 类型、 `long` 类型和无符号 `long` 类型。 `next` 指针指向了下一个 `dictEntry` ，用来解决哈希冲突。  
 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190423200516692.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FiYzEyM2x6Zg==,size_16,color_FFFFFF,t_70)

 

 
#### []()集合类型（Set）

 
--------
 集合类型可以看成是仅保存键的哈希类型，类似于Java中的 `Set` 。在Redis内部有两种编码方式：  
 1、基于整数集合实现，即 `REDIS_ENCODING_INTSET` 。  
 2、基于字典实现，即 `REDIS_ENCODING_HT` 。

 
##### []()1、整数集合

 []()简介 当一个集合类型的数据满足所有元素都是整数值并且不超过512个（默认情况，可通过配置文件修改上限数量）的情况下才会采用这种编码方式。整数集合可以保存类型为 `int16_t` 、 `int32_t` 和 `int64_t` 的整数值，并且可以保证元素中不会出现重复的元素。

 []()定义 整数集合定义在头文件 `intset.h` 中：

 
```
typedef struct intset {
	uint32_t encoding;	//编码方式
	uint32_t length;	//集合包含的元素数量
	int8_t contents[];	//保存元素的数组
}

```
 `contents` 数组用于保存元素，这些元素会按值的大小从小到大排列。虽然 `contents` 数组声明为 `int8_t` 类型，但实际上 `contents` 数组并不会保存任何 `int8_t` 类型的值， `contents` 数组的保存的元素的真正类型取决于 `encoding` 属性的值：  
 1、如果 `encoding` 等于 `INTSET_ENC_INT16` ，那么 `contents` 就是一个 `int16_t` 类型的数组，其值范围是 `-32768` ~ `32767` 。  
 2、如果 `encoding` 等于 `INTSET_ENC_INT32` ，那么 `contents` 就是一个 `int32_t` 类型的数组，其值范围是 `-2147483648` ~ `2147483647` 。  
 3、如果 `encoding` 等于 `INTSET_ENC_INT64` ，那么 `contents` 就是一个 `int64_t` 类型的数组，其值范围是 `-9223372036854775808` ~ `-9223372036854775807` 。  
 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190423193538932.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FiYzEyM2x6Zg==,size_16,color_FFFFFF,t_70)

 []()整数集合的升级 如果在插入元素的时候发现 `contents` 数组的类型无法容纳时，会进行升级操作，其步骤为：  
 1、根据新元素的类型，扩展整数集合 `contents` 数组长度。  
 2、将 `contents` 数组中所有元素的类型转换为与新元素相同的类型，并将转换后的元素放在正确的位置上。  
 3、替换旧的 `contents` 数组。

 升级操作有两大好处：  
 1、提升灵活性  
 C语言是静态类型的语言，为了避免发生类型错误一般都不会将两种不同类型的数据放到同一个数据结构中，整数集合通过升级操作改善了这一点，可以根据需要进行类型的调整。

 2、节约内存  
 整数集合并没有采用直接通过 `int64_t` 类型的数组来保存数据，而是根据需要动态更改类型，这样可以省下一些内存空间。

 此外整数集合不会进行降级操作。

 
##### []()2、字典

 和哈希类型中的字典类似，集合类型也可以通过字典来实现，其元素排列是无序的。其结构如下：  
 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190423200734927.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FiYzEyM2x6Zg==,size_16,color_FFFFFF,t_70)  
   
  


 
#### []()有序集合类型（ZSet）

 
--------
 有序集合相比普通集合能够确保元素的有序性。在Redis内部，有两种编码方式：  
 1、基于压缩列表实现，即 `REDIS_ENCODING_ZIPLIST` 。  
 2、基于跳跃表实现，即 `REDIS_ENCODING_SKIPLIST` 。

 
##### []()1、压缩列表

 压缩列表同样也可以用在有序集合中，每个集合元素使用两个紧挨在一起的压缩列表节点来保存，第一个节点保存元素成员，第二个节点保存元素分值，这些元素在压缩列表中有序地排列着，分值较小的节点在前面，分值较大的排在后面。  
 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190423203555751.png)

 
##### []()2、跳跃表

 []()简介 跳跃表是一种有序的数据结构，它通过在每个节点中维持多个指向其他节点的指针，从而达到快速访问元素的目的。它支持平均 `O(log N)` 、最坏 `O(N)` 的节点查找。大部分情况下它的查找效率可以和红黑树媲美，并且相比红黑树可以更方便地实现并发操作（例如Java中的 `ConcurrentSkipListMap` ）。

 如果一个有序集合包含的元素比较多（128个），或者单个元素本身占用的空间比较大（64字节以上），Redis就会采用跳跃表来实现有序集合。

 []()定义 跳跃表及跳跃表节点结构体实现在头文件 `server.h` 中：

 
```
typedef struct zskiplist {
    struct zskiplistNode *header, *tail; //指向表头、表尾节点
    unsigned long length; 	//跳跃表包含的节点数量
    int level; 	//跳跃表层数
} zskiplist;

```
 跳跃表节点定义：

 
```
typedef struct zskiplistNode {
	struct zskiplistNode *backward;	//后退指针
	double score;	//分值
	robj *obj;		//成员对象
	struct zskiplistLevel {
		struct zskiplistNode *forward;	//前进指针
		unsigned int span;	//跨度
	} level[];
} zskiplistNode;

```
 `level` 表示层，每个层都具有两个属性：前进指针和跨度，前进指针用于访问后面的节点，跨度则记录了当前节点和前进指针指向的节点的距离。  
 后退指针指向了当前的节点的前一个结点，一般用于从表尾遍历到表头。  
 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190423213045506.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FiYzEyM2x6Zg==,size_16,color_FFFFFF,t_70)

 []()利用跳跃表实现的有序集合 `REDIS_ENCODING_SKIPLIST` 编码的有序集合使用 `zset` 结构体作为其底层实现：

 
```
typedef struct zset {
	zskiplist *zsl;
	dict *dict;
} zset;

```
 `zset` 除了用跳跃表存储所有的集合元素外，还使用了 `dict` 字典为有序集合创建了一个成员到分值的映射表，其中，键保存了元素的内容，值保存了元素的分值，通过这个字典可以实现以 `O(1)` 的时间复杂度查找分值。值得注意的是，虽然 `zset` 使用了跳跃表和字典保存元素，但是这两种数据结构都会通过指针来共享数据，并不会造成双倍的空间占用。  
 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190423215204297.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FiYzEyM2x6Zg==,size_16,color_FFFFFF,t_70)

 

 
#### []()Stream类型

 
--------
 `Stream` 类型是Redis 5.0的新数据类型，可以用来实现消息队列，和其它类型的数据一样， `Stream` 类型的数据也是通过键值对的形式存储在Redis字典中的。

 其操作命令有：  
 ![在这里插入图片描述](https://upload-images.jianshu.io/upload_images/3068329-53aaab080e40a359.png)  
 图片转自：[https://www.jianshu.com/p/e5751c2ac9c8](https://www.jianshu.com/p/e5751c2ac9c8)

 []()定义 Stream数据类型定义在结构体 `stream` 中：

 
```
typedef struct stream {
    rax *rax;			//rax树,用来存储消息
    uint64_t length;	//这个stream的消息长度
    streamID last_id;	//当前stream的最后一个id
    rax *cgroups; 		//当前stream有哪些group
} stream;

```
 每个消息都有一个唯一ID，通过结构体 `streamID` 表示：

 
```
typedef struct streamID {
	uint64_t ms;	//UNIX时间戳
	uint64_t seq;	//序列号 
} streamID;

```
 `rax` 树结构体定义，位于头文件 `rax.h` 中：

 
```
typedef struct rax {
    raxNode *head;		//rax树头节点
    uint64_t numele;	//元素数量
    uint64_t numnodes;	//rax树节点数量
} rax;

```
 `rax` 树节点 `raxNode` ：

 
```
typedef struct raxNode {
	uint32_t iskey:1;	//这个节点是否包含Key
	uint32_t isnull:1;	//
	uint32_t iscompr:1;
	uint32_t size:29;
	unsigned char data[];
} raxNode;

```
 `Stream` 类型的数据通过一个称为 `radix tree` 的数据结构存储消息，其实现细节可以参考这篇博客：[https://www.cnblogs.com/wuchanming/p/3824990.html](https://www.cnblogs.com/wuchanming/p/3824990.html)

 []()要点 `Stream` 消息队列有以下几个特性：  
 1、支持多个消费者获取数据，意味着一条消息可以被多个消费者收到。  
 2、消费者请求的消息一定是最新的。  
 3、消费一条消息后需要进行确认操作。  
 4、支持消息的持久化。类似于其它数据类型， `Stream` 类型的数据同样也支持主从复制操作，也可以通过 `AOF` 和 `RDB` 进行持久化。

 

 
##### []()参考资料

 部分内容参考自《Redis设计与实现》黄健宏著

   
  