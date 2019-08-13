---
title: 谈谈Spring框架的事务管理体系
date: 2019-04-09 20:58:53
tags: CSDN迁移
---
 版权声明：尊重原创，转载请注明出处 [ https://blog.csdn.net/abc123lzf/article/details/89106829]( https://blog.csdn.net/abc123lzf/article/details/89106829)   
  ### []()一、引言

 Spring的事务管理框架将代码中需要使用到事务的地方进行分离，无需关心所使用的数据访问技术及其访问什么类型的数据资源，它的设计理念就是**让事务管理的关注点与数据访问的关注点分离**。

 回顾一下事务的四大特性ACID：  
 **原子性**：事务确保操作要么全部完成，要么全部取消不执行。  
 **一致性**：在事务开始和完成时，数据都必须保持一致状态，事务结束时，所有的内部数据结构都必须正确。  
 **隔离性**：数据库需要提供一定的隔离机制，保证事务不受外部并发操作的影响的“独立”环境执行。  
 **持久性**：事务一旦被Commit后，对数据库的改变是持久的，即使数据库系统发生故障也能够保持。

 在没有Spring的事务管理前，我们都是直接通过JDBC或者ORM框架、JTA来管理事务的：  
 **先来看看JDBC的事务操作：**

 
```
public void test() throws Exception {
	Class.forName("com.mysql.cj.jdbc.Driver"); //加载数据库驱动
    Connection conn = DriverManager.getConnection("jdbc:mysql://localhost:3306/mysql?" +
       "useUnicode=true&characterEncoding=utf8&serverTimezone=GMT%2B8&useSSL=false", "root", "123456");
    conn.setAutoCommit(false); //关闭事务的自动提交，显式管理事务
    try {
        PreparedStatement ps = conn.prepareStatement("update user set password=password('123') where user = 'root'");
        ps.executeUpdate();
        ps.close();
        conn.commit();
    } catch (Exception e) {
        conn.rollback();
    } finally {
        conn.close();
    }
}

```
 传统JDBC编程的流程分为：  
 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190409152801896.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FiYzEyM2x6Zg==,size_16,color_FFFFFF,t_70)  
 JDBC对Java程序进行数据库的操作提供了最基本的API，它的主要优点是API简单，性能相对较好，但是JDBC的缺点显而易见：冗长、重复、事务的显式处理。

 **再来看看ORM框架MyBatis的事务操作：**

 
```
public void test2() {
    InputStream resource = getClass().getResourceAsStream("mybatis.xml");
    SqlSessionFactory sf = new SqlSessionFactoryBuilder().build(resource);
    SqlSession sqlSession = sf.openSession();
    try {
        Student s = new Student();
        s.setStudentName("lzf");
        s.setPhoneNumber("123456");
        sqlSession.insert("com.testapp.StudentMapper", s);
        sqlSession.commit();
    } catch (Exception e) {
        sqlSession.rollback();
    } finally {
        sqlSession.close();
    }
}

```
 可以看出，单纯的使用MyBatis进行事务操作也显得代码十分的冗余、复杂，可扩展性和可移植性也不好。

 **最后是Spring的事务操作：**

 
```
@Autowired private StudentMapper studentMapper;

@Transactional(rollbackFor = Exception.class)
public void test3() {
	Student s = new Student();
    s.setStudentName("lzf");
    s.setPhoneNumber("123456");
    studentMapper.insert(s);
}

```
 仅需一个 `@Transactional` 注解，就完成了所有的事务操作，业务编写人员只需要将事务属性（事务传播属性、事务隔离等级、回滚条件等）通过注解配置，并将业务逻辑写到方法中，无需考虑各种数据库连接的获取、提交、回滚操作。这充分体现了Spring的事务框架设计理念：**让事务管理的关注点与数据访问的关注点分离**。

 了解完这些后，我们来分析Spring的事务管理是如何实现的

 
### []()二、Spring的事务管理接口体系

 
##### []()2.1 PlatformTransactionManager

 `org.springframework.transaction.PlatformTransactionManager` 是Spring事务抽象架构的核心接口，它为应用程序提供了事务界定的统一方式：

 
```
public interface PlatformTransactionManager {
	TransactionStatus getTransaction(@Nullable TransactionDefinition definition) throws TransactionException;
	void commit(TransactionStatus status) throws TransactionException;
	void rollback(TransactionStatus status) throws TransactionException;
}

```
 这三个方法定义了事务最基本的操作：  
  `TransactionStatus getTransaction(@Nullable TransactionDefinition definition) throws TransactionException`   
 根据指定的事务属性，创建一个新的事务或者返回当前活动的事务

 `void commit(TransactionStatus status) throws TransactionException`   
 通过当前的事务状态，提交事务。

 `void rollback(TransactionStatus status) throws TransactionException`   
 通过当前的事务状态，回滚事务。

 在实际的应用开发中（Spring Boot + MyBatis），我们通常使用的 `PlatformTransactionManager` 实现类是 `DataSourceTransactionManager` ，它的继承结构如下：  
 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190409161551882.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FiYzEyM2x6Zg==,size_16,color_FFFFFF,t_70)  
 除了 `DataSourceTransactionManager` 以外，还有：  
  `HibernateDataSourceTransactionManager` ，对应ORM框架Hibernate  
  `JdoTransactionManager` ，对应JDO  
  `JpaTransactionManager` ，对应JPA  
  `TopLinkTransactionManager` ，对应TopLink  
  `JmsTransactionManager` ，对应JMS  
  `CciLocalTransactionManager` ，对应JCA Local Transaction

 
##### []()2.2 TransactionDefinition

 `org.springframework.transaction.TransactionDefinition` 接口定义了事务的基本的属性：

 
```
public interface TransactionDefinition {
	//事务的传播行为
	int PROPAGATION_REQUIRED = 0;
	int PROPAGATION_SUPPORTS = 1;
	int PROPAGATION_MANDATORY = 2;
	int PROPAGATION_REQUIRES_NEW = 3;
	int PROPAGATION_NOT_SUPPORTED = 4;
	int PROPAGATION_NEVER = 5;
	int PROPAGATION_NESTED = 6;
	//事务的隔离级别
	int ISOLATION_DEFAULT = -1;
	int ISOLATION_READ_UNCOMMITTED = Connection.TRANSACTION_READ_UNCOMMITTED;
	int ISOLATION_READ_COMMITTED = Connection.TRANSACTION_READ_COMMITTED;
	int ISOLATION_REPEATABLE_READ = Connection.TRANSACTION_REPEATABLE_READ;
	int ISOLATION_SERIALIZABLE = Connection.TRANSACTION_SERIALIZABLE;

	int TIMEOUT_DEFAULT = -1;

	int getPropagationBehavior(); //获取该事务的传播行为
	int getIsolationLevel();	  //获取该事务的隔离级别
	int getTimeout();			  //获取该事务的超时时间
	boolean isReadOnly();		  //该事务是否只读
	@Nullable String getName();   //获取事务的名称
}

```
 TransactionDefinition接口定义了4个事物的基本属性：  
 **1、事务的隔离级别**  
 事务的隔离级别有4种，分别是未提交读（ `ISOLATION_READ_UNCOMMITTED` ）、已提交读（ `ISOLATION_READ_COMMITTED` ）、可重复读（ `ISOLATION_REPEATABLE_READ` ）、可序列化（ `ISOLATION_SERIALIZABLE` ）。

 未提交读是最低的隔离级别，允许读取未提交的事务变更，会导致脏读、不可重复读和幻读。  
 已提交读是MySQL默认的隔离级别，允许读取事务提交后的结果，但是依然可能发生不可重复读和幻读问题。  
 可重复读对同一字段多次读取都是一致的，但是依然可能发生幻读问题。  
 可序列化是最高级别的隔离级别，它要求同一时间最多只能有一个事务在执行，但是会严重影响数据库的性能。

 **2、事务的传播行为**  
 事务的传播行为表示**整个事务处理过程所跨越的业务对象，将以什么样的行为参与事务**，例如ServiceA在调用ServiceB的时候，如果指定事务的传播行为为 `REQUIRED` ，那么如果ServiceA开启了一个事务，ServiceB的业务方法会加入ServiceA开启的事务。

 `TransactionDefinition` 定义了7种事务传播行为：

 `TransactionDefinition.PROPAGATION_REQUIRED` ：  
 如果当前存在事务，则加入该事务；如果当前不存在事务，则创建一个新的事务。  
  `TransactionDefinition.PROPAGATION_SUPPORTS` ：  
 如果当前存在事务，则加入该事务；如果当前没有事务，则直接执行不创建事务。  
  `TransactionDefinition.PROPAGATION_MANDATORY` ：  
 如果当前存在事务，则加入该事务；如果当前没有事务，则抛出异常。  
  `TransactionDefinition.PROPAGATION_REQUIRES_NEW` ：  
 无论当前有没有事务，都会创建一个新的事务。如果当前存在事务，则挂起当前事务。  
  `TransactionDefinition.PROPAGATION_NOT_SUPPORTED` ：  
 以非事务方式运行，如果当前存在事务，则挂起当前事务。  
  `TransactionDefinition.PROPAGATION_NEVER` ：  
 以非事务方式运行，如果当前存在事务，则抛出异常。  
  `TransactionDefinition.PROPAGATION_NESTED` ：  
 如果当前存在事务，则创建一个事务作为当前事务的嵌套事务来运行；如果当前没有事务，其行为与 `PROPAGATION_REQUIRED` 类似。

 **3、事务的超时**  
 事务的超时属性指定了事务所允许执行的最长时间，若在规定时间内还没有完成事务，那么事务会自动回滚。getTimeout方法返回的单位是秒，若返回-1，则规定事务不存在超时时间。

 **4、事务的只读**  
 事务的只读属性表示当前事务会不会对数据源进行只读操作，如果需要进行写入操作，那么isReadOnly()方法应当返回false。

 此外，在实际开发过程中，我们可能还需要对事物进行回滚操作，但是TransactionDefinition并没有定义事务的回滚规则，而是交给了它的子接口TransactionAttribute：

 
```
public interface TransactionAttribute extends TransactionDefinition {
	@Nullable String getQualifier();
	boolean rollbackOn(Throwable ex);
}

```
 rollbackOn方法定义了当执行事务的流程抛出异常的时候是否决定执行事务回滚操作。

 
##### []()2.3 TransactionStatus

 `org.springframework.transaction.TransactionStatus` 接口定义了事务处理过程中的基本状态，在事务的处理中，我们可以通过 `TransactionStatus` 完成以下功能：  
 1、查询事务的状态  
 2、通过 `setRollbackOnly` 标记当前事务最终执行回滚操作  
 3、如果对应得 `PlatformTransactionManager` 支持Savepoint，那么可以通过 `TransactionStatus` 创建嵌套事务（MySQL尚不支持嵌套事务）。

 `TransactionStatus` 实现类及其继承结构：  
 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190409181930139.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FiYzEyM2x6Zg==,size_16,color_FFFFFF,t_70)  
 SavepointManager是在JDBC3.0的基础上，增加的对Savepoint的支持提供的一个接口。Spring事务框架内部大多采用了DefaultTransactionStatus作为TransactionStatus的实现类。

 
### []()三、@Transactional注解

 `@Transactional` 注解可以修饰类和public方法，并且可定义事务的基本属性：

 
```
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Documented
public @interface Transactional {
	@AliasFor("transactionManager")
	String value() default "";

	@AliasFor("value")
	String transactionManager() default "";
	Propagation propagation() default Propagation.REQUIRED;
	Isolation isolation() default Isolation.DEFAULT;
	int timeout() default TransactionDefinition.TIMEOUT_DEFAULT;
	boolean readOnly() default false;
	Class<? extends Throwable>[] rollbackFor() default {};
	String[] rollbackForClassName() default {};
	Class<? extends Throwable>[] noRollbackFor() default {};
	String[] noRollbackForClassName() default {};
}

```
 `@Transactional` 可定义以下基本属性：  
 1、事务管理器  
 2、事物的传播行为  
 3、事务的隔离级别  
 4、事务的超时时间及其是否只读  
 5、事务的自动回滚条件

 当事务开始时，Spring会通过AOP的动态代理模块，生成一个代理对象，当调用 `@Transactional` 修饰的方法时，会被代理对象的 `TransactionInterceptor` 拦截， `TransactionInterceptor` 包含了 `PlatformTransactionManager` ，代码中定义了控制事务的逻辑，会根据注解的内容调用事务管理器的相关方法来执行事务。

 `TransactionInterceptor` 继承关系图：  
 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190409195340900.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FiYzEyM2x6Zg==,size_16,color_FFFFFF,t_70)  
 可以看出， `TransactionInterceptor` 实现了 `Advice` 接口。  
 回顾一下Spring AOP的内容， `Advice` 是指作用在 `Pointcut` （切入点）的逻辑，那么可以把 `@Transactional` 注解看成是一个 `Pointcut` ，它和 `TransactionInterceptor` 共同组成了一个 `Around Advisor` ，将 `@Transactional` 修饰的方法中的逻辑包裹起来。

 `TransactionInterceptor` 会通过 `ThreadLocal` 为每个线程创建一个 `TransactionInfo` 对象，这个对象包含了我们熟知的事务管理器（ `PlatformTransactionManager` ）、事务状态对象（ `TransactionStatus` ）和事务基本属性（ `TransactionAttribute` ）对象。并通过这个 `TransactionInfo` 实现当前事务的操作。

 关于 `ThreadLocal` 的工作原理，可以参考我的博客：[https://blog.csdn.net/abc123lzf/article/details/81978210](https://blog.csdn.net/abc123lzf/article/details/81978210)

   
  