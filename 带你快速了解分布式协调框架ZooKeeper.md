---
title: 带你快速了解分布式协调框架ZooKeeper
date: 2019-04-07 13:53:06
tags: CSDN迁移
---
 版权声明：尊重原创，转载请注明出处 [ https://blog.csdn.net/abc123lzf/article/details/89043472]( https://blog.csdn.net/abc123lzf/article/details/89043472)   
  ### []()一、初识ZooKeeper

 `ZooKeeper` 最早起源于雅虎，雅虎很多大型服务器都需要一个类似的分布式协调系统来进行工作，但是这些系统往往存在着单点问题（单点问题指系统某一服务失效会导致整个系统处于不可用的状态），所以这些雅虎的技术大牛们就尝试去开发一款专门的分布式协调系统，提供一些基本的API，让开发人员能够将精力放在业务逻辑的开发上而不要考虑各种分布式协调问题。

 如今 `ZooKeeper` 已成为Apache软件基金会的一个软件项目，它曾经是著名的分布式计算框架Hadoop的一个子项目，但如今是一个独立的顶级项目。

 
### []()二、ZooKeeper入门

 可以把 `ZooKeeper` 看成一个特殊的分布式存储系统，它提供一种类似于Linux文件系统的数据结构（图片来自网络）：  
 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190405150647678.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FiYzEyM2x6Zg==,size_16,color_FFFFFF,t_70)  
 看上去是不是很像Linux系统的目录结构？用 / 表示根节点，然后可以创建很多个子节点。在 `ZooKeeper` 中，节点称之为数据节点（ `ZNode` ）， `ZNode` 不像Linux文件系统那样有文件夹和文件之分，每个 `ZNode` 都可以存入数据（不超过1MB），也可以建立很多子节点。

 下面我们来讲解 `ZNode` ：

 
##### []()2.1 数据节点——ZNode

 在 `ZooKeeper` 中， `ZNode` 可以分为**持久节点**和**临时节点**。客户端在创建持久节点后，除非主动进行删除操作，否则这个节点会一直保留在 `ZooKeeper` 服务器中。而对于临时节点就不一样了，它的生命周期和客户端会话绑定，当客户端连接到 `ZooKeeper` 服务器并主动创建临时节点后，只要客户端断开了连接并且会话失效，那么这个 `ZNode` 也随之会删除。

 此外， `ZNode` 的节点还可以细分为2种： `Sequential` 节点和非 `Sequential` 节点， `Sequential` 节点在创建后节点名称会自动加上一个递增的整型数字。例如客户端调用API创建一个 `Sequential` 的持久节点"/test"后，其节点实际名称为"/test0000000001"

 
##### []()2.2 集群角色

 了解数据节点 `ZNode` 后，我们来了解一下 `ZooKeeper` 集群的角色。  
  `ZooKeeper` 集群有三种角色： `Leader` 、 `Follower` 和 `Observer` ，每个 `ZooKeeper` 服务器在启动时需要绑定三个端口：用于客户端连接的端口、用于选举 `Leader` 的端口、 `ZooKeeper` 服务器集群通信端口。  
 ![在这里插入图片描述](https://img-blog.csdnimg.cn/2019040516130334.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FiYzEyM2x6Zg==,size_16,color_FFFFFF,t_70)  
 （图片来自网络）  
  `ZooKeeper` 集群当中的所有服务器都是通过一个 `Leader` 选举过程来选定一台称之为 `Leader` 的服务器， `Leader` 服务器为客户端提供读服务、写服务（读和写指的是对 `ZNode` 的操作）。  
 除了 `Leader` 以外， `Follower` 和 `Observer` 能够提供读服务（不提供写服务，客户端发起的写服务转交给 `Leader` 处理）。 `Follower` 和 `Observer` 唯一的区别在于， `Observer` 不参与 `Leader` 的选举，也不会参与写操作的“过半写成功”策略。

 
##### []()2.3 会话

 这里的会话指的是客户端与 `ZooKeeper` 服务器建立的会话。客户端与 `ZooKeeper` 的连接方式为**TCP长连接**，在客户端与 `ZooKeeper` 建立起TCP连接后，其会话的生命周期便开始了。在这个TCP长连接中，客户端可通过心跳检测来保证与服务器的有效会话，能够向 `ZooKeeper` 发起各种请求，同时 `ZooKeeper` 也可以向客户端发送事件通知（前提是客户端向 `ZooKeeper` 注册了事件监听器 `Watcher` ）。

 
##### []()2.4 事件监听器——Watcher

 事件监听器是 `ZooKeeper` 的一个非常重要的特性，很多分布式协调工作离开不了它。它实现了 `ZooKeeper` 的一对多的订阅发布模式，能够让多个订阅者（客户端）同时监听某个对象，当这个被监听的对象状态发生变化时， `ZooKeeper` 服务器会主动通知订阅者，然后根据订阅者实现的逻辑处理这个事件。  
 例如：客户端可以在 `ZNode` 节点注册一些 `Watcher` ，当有特定事件（例如 `ZNode` 的删除、添加、数据的变更等）触发时， `ZooKeeper` 服务器会主动根据 `Watcher` 向客户端发起通知。

 `Watcher` 机制是 `ZooKeeper` 实现分布式协调服务的十分重要的特性，这个机制让基于 `ZooKeeper` 实现的分布式应用得到了解耦，这些应用只需要对 `ZooKeeper` 服务器进行操作，而无需关注各种分布式的协调问题。

 
##### []()总结

 1、 `ZooKeeper` 本身就是一个为分布式而生的程序，只要有半数以上的 `ZooKeeper` 服务器存活， `ZooKeeper` 服务就能正常使用。  
 2、 `ZooKeeper` 将数据保存在内存中，保证了它的高吞吐量和低延迟特性，这也是为什么 `ZooKeeper` 要限制 `ZNode` 最大大小为1MB。  
 3、 `ZooKeeper` 在读操作大于写操作的情况下是高性能的，因为写操作会导致所有 `ZooKeeper` 服务器的同步。  
 4、对于分布式理论CAP， `ZooKeeper` 保证了CP（一致性和分区容错性），舍弃了A（可用性）。  
 5、 `ZooKeeper` 实际上只有2个功能：负责管理客户端提交的数据、为客户端提供 `ZNode` 的监听服务。

 
### []()三、ZAB协议

 
##### []()3.1 简介

 ZAB协议是 `ZooKeeper` 的核心， `ZooKeeper` 通过ZAB协议，实现了数据的强一致性（即实现了CAP理论中的CP），它面向ZooKeeper提出了四个需求：  
 **（1）可靠的提交**：一个消息一旦被一个节点提交，它一定最终被所有服务提交  
 **（2）全局顺序**：如果在一个节点上，一个消息A在消息B之前被提交，那么对于其它所有节点，一定也是消息A在消息B之前提交。  
 **（3）因果顺序**：如果消息B依赖于消息A，那么消息A的提交必须在消息B之前  
 **（4）前序性**：如果一个消息被提交，那么这个消息之前提出的所有消息都应该被提交。

 基于上述需求， `ZooKeeper` 实现了一种主备模式的系统架构来保持集群中各节点的数据一致性，其基本思路是：  
 所有来自客户端的事务的请求都必须由 `Leader` 服务器处理， `Leader` 服务器负责将客户端的事务请求转换为一个**事务提议**（ `Proposal` ），并将这个事务提议分发给 `Follower` 服务器。之后， `Leader` 服务器便等待 `Follower` 服务器的响应，一旦有超过半数的 `Follower` 服务器给出了正确的反馈后，那么 `Leader` 就会向所有的 `Follower` 分发事务提交消息。

 简单介绍了ZAB协议后，我们来了解一下ZAB协议的**两种基本模式**：消息广播和崩溃恢复  
 ![在这里插入图片描述](https://img-blog.csdnimg.cn/2019040713513150.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FiYzEyM2x6Zg==,size_16,color_FFFFFF,t_70)

 
##### []()3.2 消息广播

 消息广播分为了4个步骤：  
 （1） `Leader` 服务器接收到来自客户端的请求后，按照其接受到请求的顺序，给请求分配一个**ZXID**（每个请求唯一），随后 `Leader` 服务器会将请求转发给 `Follower` ，同时按照发送顺序写入本地日志。  
 （2） `Follower` 收到来自 `Leader` 的 `Proposal` 消息后，那么将按照消息收到的顺序写入本地日志持久化，持久化完成后向 `Leader` 服务器发送ACK消息。  
 （3）对于一个请求，如果 `Leader` 收到了超过半数的 `Follower` 的ACK消息，那么 `Leader` 服务器就会提交这个事务，并随后发送 `Commit` 消息给各个 `Follower`   
 （4） `Follower` 收到 `Commit` 消息后，提交对应的请求。

 我们以一个客户端连接 `Follower` 服务器并向 `Follower` 服务器发起 `ZNode` 写请求来分析这一过程：  
 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190407134550301.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FiYzEyM2x6Zg==,size_16,color_FFFFFF,t_70)

 在ZAB协议的事务编号 `ZXID` 设计中， `ZXID` 是一个64位的整型数字，其中低32位可以看成是一个递增的计数器（类似 `AtomicInteger` ），每当 `Leader` 产生一个事务后，就会从对该数字进行一个加1操作。高32位则表示 `Leader` 周期，称之为**epoch**，当老 `Leader` 崩溃后重新选举了一个新的 `Leader` 服务器的时候，就会从新 `Leader` 上取出其本地日志中最大 `ZXID` 的事务，并从该 `ZXID` 中取出其 `epoch` 值并对其进行加1操作，之后就以这个数字作为新的 `epoch` ，然后将低32位重置为0。

 
##### []()3.3 崩溃恢复

 当 `Leader` 服务器处于刚启动的状态、出现故障、或者网络连接发生异常导致失去与过半的 `Follower` 服务器的联系时，那么就会进入崩溃恢复模式。崩溃恢复模式的目的是选举出一个新的 `Leader` 服务器，因此ZAB协议需要的是一个高效且可靠的 `Leader` 选举算法，确保能够快速选出 `Leader` 服务器。

 崩溃恢复需要满足以下两个需求：  
 1、已经被处理的消息不能被丢弃。  
 2、没有被处理的消息不能再出现。

 根据上述需求，ZAB协议将崩溃恢复过程划分了两个阶段：  
 **1、Leader选举阶段：** 当检测到 `Leader` 发生崩溃或者网络连接超时后，选举出一个新的 `Leader`   
 （1） `Follower` 向 `Leader` 候选者（准 `Leader` ）发送包含自己最后接收到的事务的 `epoch` 值的消息，该消息称之为 `Cepoch` 消息。  
 （2） `Leader` 候选者接收到过半的 `Follower` 发出的 `Cepoch` 消息后，设置自己的 `epoch` 为所有 `Cepoch` 消息中最大的值加1。随后，准 `Leader` 会发送 `Newepoch` 消息（内容包含新的 `epoch` 值）给这些过半的 `Follower` 。  
 （3） `Follower` 收到 `Newepoch` 消息后，如果本地的 `epoch` 值小于消息中的 `epoch` 值，那么就更新本地的 `epoch` 值为消息中的值，这样 `Follower` 就不能再接受之前 `epoch` 的 `Leader` 消息。接着向 `Leader` 候选者发送 `Ack-E` 消息，这个消息包含了原先的 `epoch` 值和该 `Follower` 的本地日志（内容包含它处理过的历史事务）  
 （4）当 `Leader` 候选者收到 `Ack-E` 消息后，从中选择一个 `Follower` 的日志文件作为本地日志，优先选择 `epoch` 最大的日志，如果 `epoch` 相同，则选择 `ZXID` 最大的日志。此时 `Leader` 候选者正式成为了 `Leader` 服务器，但是需要经历同步阶段才能接受新的客户端请求。

 ZAB协议并没有直接规定如何选择一个节点作为未来的 `Leader` 候选人，在 `ZooKeeper` 的实现中采取了以下规则：选择了多数派中 `ZXID` 最大的节点作为 `Leader` 候选人，这样可以让新的 `Leader` 日志文件包含了最多的被提交的请求，简化了同步阶段的流程，减少 `Follower` 向 `Leader` 发生日志的消息开销。

 **2、同步阶段**：新的 `Leader` 服务器和多数 `Follower` 的日志保持一致。  
 （1） `Leader` 服务器以消息的形式发送整个日志和本地 `epoch` 给所有的 `Follower` ，这个消息称之为 `Newleader` 。  
 （2） `Follower` 收到 `Newleader` 消息后，如果发现本地的 `epoch` 和消息中的 `epoch` 相同，那么会回复ACK给 `Leader` 服务器。  
 （3） `Leader` 收到了超过一半的 `Follower` 服务器的 `ACK` 后，就发送 `Commit` 消息给 `Follower` ，此时， `Leader` 服务器开始正式接受来自客户端的请求。  
 （4） `Follower` 收到 `Commit` 消息后提交本地日志中的所有内容。

 
### []()四、典型应用场景

 `ZooKeeper` 能够很好地保证分布式环境中数据的一致性，基于这种特性 `ZooKeeper` 成为了解决分布式一致性问题的利器。

 
##### []()4.1 配置中心

 配置中心顾名思义就是将负责存储配置数据并将配置提供给其它的应用。  
 配置信息有以下3个特性：  
 1、数据量比较小  
 2、数据内容运行时可能会发生变化  
 3、需要各个服务器共享

 针对上述特性，可以将一些应用程序的配置信息存储在 `ZNode` 中。在客户端程序初始化阶段需要配置信息的时候，可以直接通过获取 `ZNode` 上的信息来拿到配置完成初始化。  
 如果客户端还需要关注配置信息的变化，可以通过注册一个 `Watcher` 到 `ZooKeeper` 服务器，当配置发生变化时， `ZooKeeper` 会根据 `Watcher` 主动通知客户端，客户端只需要通过API重新获取 `ZNode` 的内容即可。

 
##### []()4.2 分布式协调/通知

 对于一个在多台服务器上部署的应用程序而言，通常需要一个协调者来控制整个系统的运转。在应用程序中通过使用 `ZooKeeper` 进行协调，可以很好地减少系统之间的耦合性，而且能够显著系统的可扩展性。

 `ZooKeeper` 中的 `Watcher` 机制能够实现分布式环境下不同的服务器之间的协调与通知。应用程序通过连接 `ZooKeeper` 服务器将一些 `Watcher` 进行注册，监听 `ZNode` 的变化， `ZNode` 在发生变化后，客户端能够及时收到 `Watcher` 通知并做出相应的处理。

 
##### []()4.3 集群管理

 集群管理包含两个部分：集群控制和集群监控。  
 在基于分布式的应用程序运行期间，我们常常有以下需求：  
 1、需要获知集群有多少服务器正在工作  
 2、监控服务器或者应用程序的状态

 对于需求一，我们可以通过API创建一个临时的 `ZNode` ，比如创建一个/server/10.0.0.1节点，那么我只需要知道/server下有多少个 `ZNode` ，就可以知道有多少台服务器正在工作。

 对于需求二，我们同样可以创建一个 `ZNode` ，并将服务器状态写入 `ZNode` 中，监控中心只需要注册一个 `Watcher` 并关注这些节点的变化，然后将状态记录下来。

   
  