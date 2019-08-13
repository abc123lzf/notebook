---
title: InnoDB存储引擎关键特性简要介绍：插入缓冲、两次写、自适应哈希索引、异步IO、邻接页刷新
date: 2019-08-11 01:26:49
tags: CSDN迁移
---
  我们先来回顾下InnoDB整体结构：

 
#### []()InnoDB整体结构

 InnoDB存储引擎由多个**内存块**和多个**后台线程**组成，其中：

  
  * 内存块包含三个部分： `redo log_buffer` 、 `innodb_buffer_pool` 、 `innodb_additional_mem_pool_size` 。  
     ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190805191501705.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FiYzEyM2x6Zg==,size_16,color_FFFFFF,t_70)  
      `redo log_buffer` 用来缓存重做日志， `innodb_additional_mem_pool_size` 一般用来缓存LRU、锁等数据结构。  
      `innodb_buffer_pool` 即**缓冲池**，包含了数据页、索引页、undo页、插入缓冲、锁信息、自适应哈希索引和数据字典。其中数据页和索引页占据了缓冲池很大一部分空间。缓冲池通过内存快速的读写性能来弥补磁盘的IO以提高数据库的性能。
    
      
  * 后台线程类型有： `Master Thread` 、 `IO Thread` 、 `Purge Thread` 、 `Page Cleaner Thread` 。  
      `Master Thread` 是InnoDB存储引擎非常核心的后台线程，主要负责将缓冲池中的数据异步刷新到硬盘，保证数据一致性，同时也负责脏页刷新、合并插入缓冲、undo页回收。  
      `IO Thread` 负责处理IO回调，分为4种类型： `write` 、 `read` 、 `insert buffer` 、 `log IO Thread` ，Windows系统可以自定义 `read` 和 `write` 线程的数量，Linux则不可以。  
      `Purge Thread` 用于在事务提交后，回收已经使用并分配的undo页，该线程在一个MySQL进程中只可有1个。  
      `Page Cleaner Thread` 用来处理缓冲池中脏页的刷新操作，以减轻 `Master Thread` 负担。
    
       
#### []()关键特性

 InnoDB存储引擎包含了以下几个关键特性

  
  * 插入缓冲( `Insert Buffer` ) 
  * 两次写( `double write` ) 
  * 自适应哈希索引（ `Adaptive Hash Index` ） 
  * 异步IO（AIO） 
  * 刷新领接页  
##### []()插入缓冲（Insert Buffer）

 `Insert Buffer` 不仅仅是缓冲池的一部分，它和数据页一样也是物理页的组成部分。 `Insert Buffer` 是为了增加**非唯一**（Unique）、**非聚集索引**的插入或更新操作的性能。

 InnoDB存储引擎的表中主键（属于聚集索引）是行的唯一标识符，一般行记录的插入时按照主键递增的顺序插入的，并且，InnoDB存储数据时，将主键和数据通过聚簇索引的方式存储在一个文件中，无需硬盘进行随机读写：  
 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190411172453286.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FiYzEyM2x6Zg==,size_16,color_FFFFFF,t_70)  
 很多情况下，一张表上可能有多个非聚集辅助BTREE索引，辅助索引叶子节点的插入不再是顺序的了，这时就需要对这个BTREE进行随机访问，导致插入性能下降。所以， `Insert Buffer` 也就随之诞生了。

 `Insert Buffer` 在执行非聚集索引的插入操作时并不会每次都插入到索引页中，它会首先判断非聚集索引页是否在缓冲池中，如果在就直接插入，如果不在就先放入一个 `Insert Buffer` 对象中，然后以一定的频率和具体情况进行 `Insert Buffer` 和辅助索引页子节点进行**合并**操作，从而将多个插入操作合并为一个插入操作（也就是将多次随机IO缩减为一次），大大增加了非聚集索引的插入性能。

 除了 `Insert Buffer` ，InnoDB还有 `Delete Buffer` 、 `Purge Buffer` ，它们都属于 `Change Buffer` 的范畴，可以对所有DML操作进行缓冲。将一个记录进行修改操作包含两个步骤：

  
  2. 将记录标记为已删除 
  4. 真正将记录删除  `Delete Buffer` 进行第一个过程， `Purge Buffer` 进行第二个过程。

 
##### []()两次写（doublewrite）

 InnoDB两次写的特性带来的数据页的可靠性。例如：当InnoDB正在将某个页写入到表文件中，此时数据库宕机导致这个页只写了一半到表文件（称之为部分写失效），在没有两次写技术前，就有可能因为这个导致数据丢失。需要注意的是，对于部分写失效，是无法通过重做日志来恢复的，因为重做日志记录的是页的物理操作，如果这个页本身发生了损坏对其进行重做是没有用的。

 所以，在应用重做日志前，需要一个原页面的拷贝，当写入失效发生时，就可以通过这个拷贝来恢复这个页，再进行重做，这就是 `doublewrite` 所做的事。

 `doublewrite` 有两部分组成：

  
  2. 内存中的 `doublewrite buffer` ，大小为2MB 
  4. 物理磁盘共享表空间连续128个页（即2个区），大小为2MB  在对缓冲池脏页进行刷新时，首先会将脏页复制到内存的 `doublewrite buffer` ，之后通过 `doublewrite buffer` 再分两次，每次1MB，顺序地写入到共享表空间的物理硬盘上，然后调用 `fsync` 函数同步磁盘，避免缓冲写带来的问题，由于 `doublewrite` 页是连续的，因此这个过程是顺序的。在完成 `doublewrite` 页的写入后，再将 `doublewrite buffer` 中的页写入到各个表空间文件（ibd）中，此时写入是离散的。

 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190811012552847.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FiYzEyM2x6Zg==,size_16,color_FFFFFF,t_70)

 
##### []()自适应哈希索引

 一般情况下，哈希查找的时间复杂度为 `O(1)` ，仅需要1次就能定位到数据，而对于B+树一般高度为3至4层，需要3~4次查询。InnoDB为了能够利用到哈希索引所带来的查询性能优势，会监控表上各个索引页的查询，如果建立哈希索引可以带来速度提升，则建立哈希索引，这种操作称之为**自适应哈希索引**。

 自适应哈希索引通过缓冲池中的B+树页构造而来，因此建立的速度很快，无需对整张表构建哈希索引，只需要根据热点页建立。

 自适应哈希索引的构造有以下几个原则：

  
  2. 索引页的连续访问模式必须是一样的，对于(a,b)这样的联合索引页，其访问模式有 `WHERE a=xxx` 或者是 `WHERE a=xxx and b=xxx` 。如果上述两个访问模式都经常使用，那么不会对这个索引页构造自适应哈希索引。 
  4. 以上述访问模式访问了100次 
  6. 页通过该模式访问了N次，N等于页中记录除以16  通过自适应哈希索引，读取和写入速度可以提高2倍，辅助索引的连接操作性能可以提高5倍。

 
##### []()异步IO

 为了提高磁盘的操作性能，InnoDB采用了异步IO（AIO）来处理其磁盘操作，以解决同步IO在进行IO操作时需要等待操作结束才能继续后续流程的缺陷。

 在InnoDB执行SQL语句时，可能需要扫描多个索引页，也就是需要执行多个IO操作，这就会造成后一个IO操作的执行必须要等待前一个IO操作完成所造成时间开销过大的问题，而AIO可以提交多个IO操作，并等待所有IO操作的完成。此外，AIO还可以进行IO Merge操作，也就是将多个IO操作合并为1个，这样可以提高性能。

 InnoDB的AIO基于操作系统的内核实现，所以需要操作系统能够支持AIO，Windows和Linux均支持AIO，而Mac OS未提供支持。

 
##### []()刷新邻接页

 InnoDB存储引擎还提供了刷新邻接页（Flush Neighbor Page）的特性：  
 当刷新一个脏页时，InnoDB会检测该页所属区的所有页，如果是脏页那么就一起刷新，这种方式可以利用AIO将多个IO操作合并为一个IO操作，如果硬盘属于机械硬盘，那么会有显著的性能提升，对于SSD而言则没有必要，可以通过参数 `innodb_flush_neighbors` 设为0来关闭此项功能。

 参考资料：《MySQL技术内幕——InnoDB存储引擎》

   
  