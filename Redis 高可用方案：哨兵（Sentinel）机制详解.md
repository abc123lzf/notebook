---
title: Redis 高可用方案：哨兵（Sentinel）机制详解
date: 2019-06-15 18:30:32
tags: CSDN迁移
---
  ### []()概述

 [上一篇文章](https://blog.csdn.net/abc123lzf/article/details/89964253 )中我们讲到了Redis的主从复制机制。在实际应用中，单纯采用主从复制模型有以下缺点：

  
  * 默认配置下， `master` 节点可以进行读和写操作， `slave` 节点只能进行读操作，写操作被禁止（因为即使在 `slave` 上进行写操作也无法同步到 `master` 节点）。所以，主从复制模型在写操作比较频繁的使用场景下是不大合适的。 
  * 一旦 `master` 节点宕机后，整个Redis集群就无法对外提供写操作了，因为在主从复制模型下无法进行 `master` 的选举。  针对第一个问题，我们可以采用Redis的集群，Redis集群我们下一篇文章再进行讲解。针对第二个问题，我们就可以采用Redis的哨兵（ `Sentinel` ）功能了。

 `Sentinel` 是Redis高可用性的解决方案，由一个或多个 `Sentinel` 节点组成的系统可以监视多个 `master` 服务器和属下所有的 `slave` 服务器， `Sentinel` 可以在 `master` 节点宕机时，选举一个 `slave` 充当新的 `master` 继续对外服务。

 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190615005121464.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FiYzEyM2x6Zg==,size_16,color_FFFFFF,t_70)

 
### []()Sentinel启动

 可以通过以下命令启动 `Sentinel` ：

 
```
[root@localhost]# redis-sentinel sentinel.conf

```
 或者：

 
```
[root@localhost]# redis-server sentinel.conf --sentinel

```
 `Sentinel` 内部启动步骤和一般的Redis服务器有所不同，它本质上是一个运行在特殊模式下的Redis服务器。在 `Sentinel` 初始化服务器后，会将普通Redis服务器使用代码替换为 `Sentinel` 专用代码，并通过上述给定的配置文件，与监视的服务器取得连接。

 `Sentinel` 服务器在初始化时会创建一个 `sentinel.c/sentinelState` 结构体，它保存了服务器中所有和 `Sentinel` 功能有关的状态：

 
```
struct sentinelState {
	uint64_t current_epoch;    //当前纪元
	dict *masters;			   //一个字典，保存了当前Sentinel所监视的服务器
	int tilt;				   //是否进入了TILT模式
	int running_scripts;	   //目前正在执行的脚本数量
	mstime_t tilt_start_time;  //进入TITL模式的时间
	mstime_t previous_time;	   //最后一次执行时间处理器的时间
	list *scripts_queue;	   //队列，包含所有需要执行的用户脚本
} sentinel;

```
 上述结构体中最重要的就是 `masters` 字典，字典的键是服务器的名称，值是一个指向 `sentinelRedisInstance` 结构的指针。每个 `sentinelRedisInstance` 结构代表了一个Redis服务器节点，这个节点可以为主服务器、从服务器，也可以是 `Sentinel` 。  
  `sentinelRedisInstance` 结构源码如下：

 
```
typedef struct sentinelRedisInstance {
	int flags;	  //记录节点类型及节点状态
	char *name;	  //节点的名称
	char *runid;  //节点的运行ID
	uint64_t config_epoch; //配置纪元，以实现故障转移
	sentinelAddr *addr; //节点的地址，持有节点的主机名和端口
	mstime_t down_after_period; //节点没有响应后多少毫秒才会被判定为主观下线
	int quorum; //判断这个节点客观下线所需投票数量
	//...
} sentinelRedisInstance;

```
 `Sentinel` 最后一个启动步骤是创建连向被监视的主服务器的网络连接，这里包含了两个连接：

  
  * 命令连接， `Sentinel` 通过该连接向主服务器发送命令，并接收命令回复。 
  * 订阅连接，用来订阅监视服务器的 `__sentinel__:hello` 频道。  
### []()获取服务器信息

 `Sentinel` 默认会以每10秒一次的频率向服务器，向被监视的服务器发送 `INFO` 命令，服务器收到 `INFO` 命令后会将自己的运行信息发送给 `Sentinel` 。运行信息一般包含了以下内容：

  
  * 该服务器本身的信息：例如 `run_id` 等 
  * 该服务器持有的从服务器信息  如果发现了这个服务器有着新的从服务器出现， `Sentinel` 除了为这个新的从服务器创建相应的结构外，还会和这个从服务器建立命令连接和订阅连接。

 除此之外， `Sentinel` 还会以每2秒一次的频率，通过命令连接向所以被监视的服务器的 `__sentinel__:hello` 频道发送消息，每条消息包含：

  
  * `Sentinel` 本身的 `run_id` 、IP地址、端口号、当前配置纪元 `epoch`  
  * 主服务器的名称、IP地址、端口号、当前配置纪元 `epoch`   对于监视同一个服务器的多个 `Sentinel` 来说，一个 `Sentinel` 发送的消息会被其它  `Sentinel` 接收到，这些消息会被用于其他 `Sentinel` 获取该 `Sentinel` 的情况，也可用于获取该 `Sentinel` 监视的服务器的大致信息。  
  `Sentinel` 对 `__sentinel__:hello` 频道的订阅会一直持续到 `Sentinel` 与服务器连接断开为止。

 
### []()Sentinel之间的通信

 每个 `Sentinel` 服务器都保存了同样监视这个服务器的其它 `Sentinel` 的信息，这些 `Sentinel` 的信息通过本机的 `sentinels` 字典保存。 `sentinels` 字典的键是这个 `Sentinel` 的IP地址+端口号拼接起来的字符串，键则是这个 `Sentinel` 的详细信息结构。

 前面提到过， `Sentinel` 通过频道 `__sentinel__:hello` 来感知其它 `Sentinel` 的存在。每当 `Sentinel` 从频道中收到消息时，会进行以下操作：

  
  * 如果源 `Sentinel` 的实例结构在本机已经存在，那么对这个实例结构进行更新。 
  * 如果源 `Sentinel` 的实例结构不存在，那么说明源 `Sentinel` 刚刚开始监视这个服务器，那么本机会给这个源 `Sentinel` 创建一个新的实例结构，并将这个结构添加到 `sentinels` 字典。并且，本机 `Sentinel` 还会和这个源 `Sentinel` 创建一个命令连接（并**不会**创建订阅连接）。  
### []()检查服务器下线

 `Sentinel` 有两种方式来检验服务器是否处于下线状态，即通过**主观判定下线**和**客观判定下线**。  
  `Sentinel` 会以每秒一次的频率向所有和它创建了命令连接的服务器（包括普通的 `Redis` 服务器和其它 `Sentinel` ）发送 `PING` 命令，并通过其回复来判断这个服务器是否处于在线状态。

 `PING` 命令的回复分为两种情况

  
  * **有效回复**：服务器返回 `+PONG` 、 `-LOADING` 、 `-MASTERDOWN` 三种回复中的一种 
  * **无效回复**：服务器返回上述之外的回复，或是在规定时间内没有任何回复。  `Sentinel` 的配置文件中 `down-after-milliseconds` 选项指定了 `Sentinel` 判断服务器进入**主观下线**所需的时间长度，单位为**毫秒**。如果在这个时间段内目标服务器连续向当前 `Sentinel` 返回无效回复，那么 `Sentinel` 会认定这个服务器进入主观下线状态。

 在 `Sentinel` 判定一个服务器进入主观下线后，为了确认这个服务器是否真的属于下线状态，它会向同样监视这个服务器的其它 `Sentinel` 进行询问，这个命令是：

 
```
SENTINEL is-master-down-by-addr <ip> <port> <current_epoch> <runid>

```
 `Sentinel` 服务器收到上述命令后，会根据上述命令持有的服务器信息检查主服务器是否下线，然后向源 `Sentinel` 返回一条包含三个参数的 `Multi Bulk` 回复，这个回复包含以下信息：

  
  * `down_state` ：目标 `Sentinel` 对该服务器的检查结果，1代表服务器已经下线，0代表服务器没有下线。 
  * `leader_runid` ：可以为 `*` 符号或者目标 `Sentinel` 的局部 `Leader Sentinel` 的运行ID。 `*` 符号表示仅仅用于检查服务器下线状态，局部 `Leader Sentinel` 的运行ID用于选举 `Leader Sentinel` 服务器。 
  * `leader_epoch` ：目标 `Sentinel` 的 `Leader Sentinel` 的配置纪元，用于选举 `Leader Sentinel` ，仅在 `leader_runid` 不为 `*` 时有效。  根据其他 `Sentinel` 的回复，本机 `Sentinel` 将统计同意该服务器已经下线的数量，当达到了客观判定下线的数量时， `Sentienl` 就会将该服务器的结构标记为客观下线状态。

 
### []()Sentinel选举Leader

 当一个主服务器被判断为主观下线时，监视这个下线主服务器的各个 `Sentinel` 会进行协商并选举一个Leader，由Leader对下线的服务器进行故障转移操作。

 所有在线的 `Sentinel` 服务器都有被选举为Leader的资格，每次在进行Leader选举之后，不论其选举是否成功，所有Sentinel的配置纪元的值都会增加。并且Leader一旦设置，在这个配置纪元中就不可修改。

 具体的选举流程可以参考[这篇博客](https://blog.csdn.net/weixin_38071106/article/details/88033764)

 
### []()故障转移

 在选举出Leader Sentinel后，Leader将会对已经下线的主服务器实行故障转移操作，包含以下几个步骤：

  
  2. 在已经下线的主服务器属下的所有从服务器中，选出一个从服务器转换为主服务器 
  4. 已经下线的主服务器属下所有从服务器改为复制新的主服务器 
  6. 将已下线的主服务器改为新的从服务器，当这个服务器上线时成为从服务器。  那么如何选举新的主服务器呢？

  
  2. 首先在从服务器中挑选一个状态良好的从服务器，然后向这个服务器发送 `SLAVEOF no one` 命令，将这个从服务器转换为主服务器。 
  4. 发送命令后，Leader Sentinel会以每秒一次的频率向这个被升级的从服务器发送 `INFO` 命令，并观察命令回复中的 `Role` 信息，当 `Role` 信息显示为 `master` 时，Leader就知道该服务器已经顺利升级成了主服务器。  新的主服务器升级完成后，Leader会让原主服务器属下所有的从服务器（不包含新的主服务器）去复制新的主服务器，也就是通过发送我们熟知的 `SLAVEOF` 命令。最后，Leader会将已下线的旧主服务器设为新的主服务器的从服务器，待服务器重新上线后，Sentinel会立刻向它发送 `SLAVEOF` 命令

 
### []()参考资料

 《Redis设计与实现》黄建宏著

   
  