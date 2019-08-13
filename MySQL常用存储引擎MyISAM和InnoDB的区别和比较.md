---
title: MySQL常用存储引擎MyISAM和InnoDB的区别和比较
date: 2019-04-11 18:31:06
tags: CSDN迁移
---
 版权声明：尊重原创，转载请注明出处 [ https://blog.csdn.net/abc123lzf/article/details/89206008]( https://blog.csdn.net/abc123lzf/article/details/89206008)   
  ##### []()引言

 现在，MySQL数据库常用的存储引擎为MyISAM和InnoDB。MyISAM在MySQL 5.5之前一直是默认的存储引擎，不过在近些年来MySQL的发展下，InnoDB逐渐替代了MyISAM存储引擎。本文从锁机制、事务、存储结构、索引等角度来分析这两个存储引擎之间的差异。

 
--------
 
##### []()1、锁机制

 MySQL的锁机制相对于其它数据库产品而言比较简单，可以分为3类：  
 1、表级锁：开销小，加锁快，不会出现死锁，锁粒度大（整张表），并发度低，发生锁竞争概率大  
 2、行级锁：开销大，加锁慢，会出现死锁，锁粒度最小（一行数据），并发度高，发生锁竞争概率小  
 3、页面锁：开销、加锁速度、锁粒度、并发度都介于表级锁和行级锁之间，会出现死锁。

 很难说那种锁更好，只能说就具体应用程序的特点说哪种锁更合适。对于查询操作远大于修改、插入操作的表而言，采用表级锁更加合适。行级锁更加适合有大量按索引条件查询并发更新少量不同数据，同时又有并发查询的应用。

 []()MyISAM和InnoDB的比较： 1、MyISAM**只支持表锁**  
 MyISAM的表级锁有两种模式：**表共享读锁**和**表独占写锁**。  
 对MyISAM的读操作，不会阻塞其它连接对同一张表的读请求，只会阻塞写请求。只有对MyISAM的写操作，才会阻塞其它连接对这张表的读和写操作。

 MyISAM的加锁解锁都是自动的，在用户执行SELECT语句时，会自动给整张表加上读锁，在执行UPDATE、INSERT、DELETE操作时会自动加上写锁。

 在默认情况下，如果同一时间有一个连接需要进行读操作，另外一个连接需要进行写操作，那么MyISAM会优先让写操作的连接获取到写锁，这也是MyISAM表不适合有大量写入操作和读操作应用的缘由。

 2、InnoDB支持**表级锁**和**行级锁**  
 InnoDB实现了两种类型的行锁：共享锁、排他锁。  
 共享锁：允许一个事务去读一行，阻止其它事务获取相同数据集的排他锁。  
 排他锁：允许获得排他锁的事务更新数据，阻止其他事务取得相同数据集的共享读锁和排他写锁。

 此外，为了允许表级锁和行级锁的共存，实现多粒度的锁，InnoDB还实现了两种内部使用的意向锁（属于表锁）：  
 意向共享锁：当一个事务想要给数据行加上行共享锁的时候，必须先取得该表的意向共享锁。  
 意向排他锁：当一个事务想要给数据行加上行排他锁的时候，必须先取得该表的意向排他锁。

 意向锁是InnoDB存储引擎自动加上的，无需SQL语句的控制。

 InnoDB的行锁是通过给索引上的索引项加锁来实现的，如果没有索引，InnoDB会通过隐藏的聚簇索引来对记录进行加锁。这也就意味着：如果不通过索引条件检索数据，那么InnoDB将会对表中所有记录加锁，其效果等同于表锁。

 
--------
 
##### []()2、事务

 MyISAM存储引擎不支持事务，它强调的是高性能的查询，适合读多写少、原子性要求低的情形。

 对于InnoDB存储引擎则提供了较为完善的事务支持，一共支持4个等级的事务隔离级别：未提交读（ISOLATION_READ_UNCOMMITTED）、已提交读（ISOLATION_READ_COMMITTED）、可重复读（ISOLATION_REPEATABLE_READ）、可序列化（ISOLATION_SERIALIZABLE）。

 未提交读是最低的隔离级别，允许读取未提交的事务变更，会导致脏读、不可重复读和幻读。  
 已提交读是MySQL默认的隔离级别，允许读取事务提交后的结果，但是依然可能发生不可重复读和幻读问题。  
 可重复读对同一字段多次读取都是一致的，但是由于Inno DB的Gap Lock算法的存在，并不会导致幻读问题，但是对于Oracle或者SQL Server，则会发生幻读。  
 可序列化是最高级别的隔离级别，它要求同一时间最多只能有一个事务在执行，但是会严重影响数据库的性能。

 同时，对于较高的隔离级别可能更容易造成锁冲突和死锁。

 
--------
 
##### []()3、索引

 MySQL一共提供了四种索引实现：  
 B-Tree索引：最常见的索引类型，通过B树来实现数据的快速访问  
 HASH索引：通过构造哈希表来实现数据的快速访问，只有Memory引擎支持  
 R-Tree索引：通过空间索引来实现数据快速访问  
 Full-Text索引：全文索引，用于快速检索较长的文本。

 []()MyISAM和InnoDB的比较： 1、索引的支持情况：

 
     索引/存储引擎   | MyISAM | InnoDB         | Memory
     --------- | ------ | -------------- | ------ 
     B-Tree    | 支持     | 支持             | 支持    
     HASH      | 不支持    | 不支持            | 支持    
     R-Tree    | 支持     | 支持(5.7.4以后的版本) | 不支持   
     Full-Text | 支持     | 支持(5.6以后的版本)   | 不支持   

2、主键  
 MyISAM可以支持没有主键的表存在。  
 InnoDB必须设置主键，如果用户没有指定主键，那么会在表文件中设置一个6字节长的ROWID，并以此为主键。之所以这么做是因为和InnoDB的表文件结构密切相关，待会我们会提到。

 3、自增列（自增主键）  
 对于InnoDB的自增列，InnoDB规定必须包含只有该字段的索引。  
 如果是MyISAM的自增列，则可以将该字段与其他字段组成联合索引。

 
--------
 
##### []()4、存储结构

 []()（1）MyISAM存储引擎 MyISAM将一张表的结构和内容分为三个文件：  
 1、表结构文件，后缀名为frm  
 2、索引文件，后缀名为MYI  
 3、数据文件，后缀名为MYD

 索引文件保存记录所在的数据文件的位置，通过读取索引文件获取到位置信息后，再通过读取数据文件快速取得数据。  
 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190411170233117.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FiYzEyM2x6Zg==,size_16,color_FFFFFF,t_70)  
 上图中，user_id为主键，绿色的表格代表数据文件，索引文件只记录user_id的值及其数据文件的物理位置。

 []()（2）InnoDB存储引擎 相比MyISAM，InnoDB只有两个文件：  
 1、表结构文件，后缀名为frm  
 2、数据、索引文件，后缀名为ibd

 InnoDB采用了聚簇索引的方式实现B-Tree索引，聚簇索引的实现方式是将数据行和相邻主键紧凑地存入文件中。  
 对于主键索引，即存储索引值（主键的值），也存储每列的数据。如果没有设置主键，那么会尝试将外键作为主键。如果也没有外键，那么会自动生成一个隐藏的ROWID做主键。  
 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190411172453286.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FiYzEyM2x6Zg==,size_16,color_FFFFFF,t_70)  
 可以看出，数据文件本身也是索引文件。

 
--------
 
##### []()5、其它区别

 1、InnoDB支持外键，MyISAM不支持外键。

 2、对于SQL语句SELECT COUNT(*) FROM table，InnoDB需要对整张表进行读取才能得出结果。而MyISAM不同，MyISAM文件保存了整张表的记录，可以直接给出结果。但是如果加上了where条件，InnoDB和MyISAM都需要扫描整表给出结果。

 3、对多CPU的优化  
 在单线程的情况下，MyISAM读取、修改速度要比InnoDB快，但是在多线程的情况下，InnoDB读取、修改速度显著提升，而MyISAM在多CPU的情况下几乎没有提升：  
 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190411173516163.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FiYzEyM2x6Zg==,size_16,color_FFFFFF,t_70)  
 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190411173523573.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FiYzEyM2x6Zg==,size_16,color_FFFFFF,t_70)  
 4、对于SQL语句DELETE FROM table，InnoDB不会直接删除数据文件，而是一行行删除。对于MyISAM则是直接删除文件重新建立表  
 5、InnoDB存储引擎的表有着可靠的崩溃恢复机制，数据较容易恢复。MyISAM数据恢复较为困难。

 
--------
 
##### []()使用场景

 1、如果应用程序对数据的一致性要求比较高，那么需要选择InnoDB，因为InnoDB支持事务和外键  
 2、以读操作为主的业务，适合使用MyISAM。对于读多写多的业务，适合使用InnoDB。

 
##### []()参考资料

 《深入浅出MySQL》 第二版

   
  