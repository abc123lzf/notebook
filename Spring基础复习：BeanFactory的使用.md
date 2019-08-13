---
title: Spring基础复习：BeanFactory的使用
date: 2018-10-11 16:16:48
tags: CSDN迁移
---
 版权声明：尊重原创，转载请注明出处 [ https://blog.csdn.net/abc123lzf/article/details/82981382]( https://blog.csdn.net/abc123lzf/article/details/82981382)   
  **本文根据书籍《Spring揭秘》王福强著 总结**

 
### []()一、引言

 说到BeanFactory，就必然要提起IoC，IoC全名Inversion of Control，翻译为控制反转或者依赖注入，它是一种程序设计思想。

 当我们依赖于某个类或者服务时，最简单而又有效地方式就是直接通过代码（即set方法或者通过构造方法）将其依赖在一起，这都是需要我们去主动地获取依赖对象。

 相对于主动获取依赖对象，我们更希望能够被动地获取，IoC就可以达到这个目的。IoC容器会主动地将你需要的对象注入到对象中。在大型程序中，这样做的好处在于：以前对象与对象之间需要通过代码来产生耦合关系，现在只需要通过配置文件来进行调整，而通过调整配置文件又可以简单而又轻松地将对象与对象之间进行解耦。  
 Spring的BeanFactory就是一个IoC容器，它属于Spring框架中的Core（核心）部分 ，Spring AOP、Spring MVC等都是基于Spring Core之上的。

 除了BeanFactory，Spring还提供了一个容器类型：ApplicationContext，两者之间区别在于：  
 1、BeanFactory是基础类型的IoC容器，默认采取延迟初始化策略，只有当程序需要访问其中一个被容器管理的对象的时候才会对其进行初始化及其注入操作。  
 2、ApplicationContext在BeanFactory基础上构建，除了拥有BeanFactory的所有特性，它还支持事件发布、国际化信息的支持等。另外，ApplicationContext管理的对象会在容器初始化的时候将其全部初始化。  
 ![在这里插入图片描述](https://img-blog.csdn.net/20181009150856762?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FiYzEyM2x6Zg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)  
 BeanFactory子接口：  
 ![在这里插入图片描述](https://img-blog.csdn.net/20181009194621361?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FiYzEyM2x6Zg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)  
 其中：  
 AutowireCapableBeanFactory主要定义了集成其它框架的功能  
 HierarchicalBeanFactory定义了父容器的访问功能  
 ConfigurableBeanFactory定义了BeanFactory的配置功能  
 ListableBeanFactory定义了容器内Bean的枚举功能（枚举出来的Bean不会包含父容器）  
 ConfigurableListableBeanFactory：包含上述所有接口的功能

 
--------
 
### []()二、BeanFactory的XML基本配置

 BeanFactory作为Spring IoC容器，为了能够明确各个对象间的依赖关系，需要通过某种途径来记录和管理这些消息。Spring提供了3种方式来管理：直接编码方式、外部文件配置方式、注解方式。  
 相对来说，直接编码方式和注解方式用的比较少，这里我们就直接讨论外部文件配置方式中最为常用的XML配置方式。

 为了方便描述，我们定义一个用户类：

 
```
public class User {	
	private int id;
	private String name;
	private Friends friends;	
	
	public User() {}
	public int getId() { return id; }
	public void setId(int id) { this.id = id; }
	public String getName() { return name; }
	public void setName(String name) { this.name = name; }
	public Friends getFriends() { return friends; }
	public void setFriends(Friends friends) { this.friends = friends; }
}


```
 Friends类保存这个用户的好友列表：

 
```
public class Friends {
	private List<String> names;
	public Friends() { }
	public List<String> getNames() { return names; }
	public void setNames(List<String> names) { this.names = names; }
}

```
 比如：

 
```
<beans xmlns="http://www.springframework.org/schema/beans"  
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:p="http://www.springframework.org/schema/p" 
    xmlns:context="http://www.springframework.org/schema/context" 
    xmlns:mvc="http://www.springframework.org/schema/mvc" 
    xmlns:task="http://www.springframework.org/schema/task"
    xsi:schemaLocation="
        http://www.springframework.org/schema/beans 
        http://www.springframework.org/schema/beans/spring-beans-4.2.xsd 
        http://www.springframework.org/schema/context 
        http://www.springframework.org/schema/context/spring-context-4.2.xsd 
        http://www.springframework.org/schema/mvc 
        http://www.springframework.org/schema/mvc/spring-mvc-4.2.xsd 
        http://www.springframework.org/schema/task 
        http://www.springframework.org/schema/task/spring-task-4.2.xsd">
	<bean id="User" class="com.test.User">
		<property name="friends" ref="UserFriends" />
	</bean>
	<bean id="UserFriends" class="com.test.Friends">
		<property name="names">
			<list>
				<value>"Li"</value>
				<value>"Liu"</value>
			</list>
		</property>
	</bean>
</beans>

```
 测试代码：

 
```
public static void main(String[] args) {
	ClassPathXmlApplicationContext ctx = new ClassPathXmlApplicationContext("applicationContext.xml");
	User user = ctx.getBean("User", User.class);
	for(String friend : user.getFriends().getNames()) {
		System.out.println(friend);
	}
	ctx.close();
}

```
 输出结果：

 
```
Li
Liu

```
 
##### []()2.1 <beans> 标签

 beans标签是XML的根元素，beans标签中需要引入XML文档声明：

 
```
<beans xmlns="http://www.springframework.org/schema/beans"  
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:p="http://www.springframework.org/schema/p" 
    xmlns:context="http://www.springframework.org/schema/context" 
    xmlns:mvc="http://www.springframework.org/schema/mvc" 
    xmlns:task="http://www.springframework.org/schema/task"
    xsi:schemaLocation="
        http://www.springframework.org/schema/beans 
        http://www.springframework.org/schema/beans/spring-beans-4.2.xsd 
        http://www.springframework.org/schema/context 
        http://www.springframework.org/schema/context/spring-context-4.2.xsd 
        http://www.springframework.org/schema/mvc 
        http://www.springframework.org/schema/mvc/spring-mvc-4.2.xsd 
        http://www.springframework.org/schema/task 
        http://www.springframework.org/schema/task/spring-task-4.2.xsd">
</beans>

```
 每个beans标签下可以配置最多一个<description>标签，可以配置多个<import>、<alias>、<bean>标签。  
 其中：description标签主要用于描述这个XML文件，无实际作用。  
 alias标签可以给bean标签指定一个别名，比如：

 
```
<alias name="UserFriends" alias="Friends"/>

```
 import标签可以将别的Spring XML配置文件引入到这个配置文件中，然而Spring IoC容器也可以加载多个配置文件，所以该标签在多配置文件的情况下不是必须的。

 bean标签是Spring XML配置文件中最重要的元素，每个业受管的业务对象都是与bean元素一一对应的。  
 如果一个bean标签需要指示一个Java类，那么它往往需要一个id属性和class属性，id属性是这个bean的唯一标识符，不可重复，class属性则是这个Java类的类名：

 
```
<bean id="User" class="com.test.User" />

```
 
##### []()2.2 构造方法注入

 如果我们希望直接通过构造方法来给User对象和Friends对象互相依赖起来，我们可以这样配置：

 
```
<beans> <!--省略文档声明-->
	<bean id="User" class="com.test.User">
		<constructor-arg ref="UserFriends" />
	</bean>
	<bean id="UserFriends" class="com.test.Friends">
		<property name="names">
			<list>
				<value>"Li"</value>
				<value>"Liu"</value>
			</list>
		</property>
	</bean>
</beans>

```
 然后给User类添加一个构造方法即可：

 
```
public User(Friends friends) {
	this.friends = friends;
}

```
 如果构造方法有多个并且需要传入的参数数量相同，那么可以通过type属性来区分：

 
```
<constructor-arg value="1" type="int" />
<constructor-arg value="1.0" type="double" />

```
 
##### []()2.3 setter方法注入

 Spring为setter方法注入提供了<property>标签，示例可以参考最上面那个例子。  
 property需要有一个name属性，用来指定该<property>将要注入的对象所对应的变量名。如果需要的话，property标签和constructor-arg标签是可以一起使用的。

 
##### []()2.4 <property>标签和<construct-arg>标签中的配置项

 （1）value和ref  
 value可以给对象注入一个简单的数据类型：String还有Java的基本数据类型及其包装类，比如：

 
```
<property name="name" value="Li" />
<!--或者-->
<property name="name">
	<value>Li</value>
</property>

```
 ref标签可以用来引用容器中的其它对象实例，其值为bean的id或name，可以通过ref的local、parent、bean属性来制定bean具体引用的是什么，例如：

 
```
<property name="friends">
	<ref local="UserFriends" />
</property>

```
 local只可指定与当前配置的对象在同一个配置文件的对象定义的名称。  
 parent只可指定当前IoC容器的父容器中定义的bean  
 bean则两者皆可。

 （2）内部的<bean>  
 如果依赖的对象只有当前一个对象引用，或者我们不希望其它对象能够依赖它，我们可以用内嵌bean标签的方式达到这种目的：

 
```
<beans> <!--省略文档声明-->
	<bean id="User" class="com.test.User">
		<constructor-arg>
			<bean class="com.test.Friends">
				<property name="names">
					<list>
						<value>Li</value>
						<value>Liu</value>
					</list>
				</property>
			</bean>
		</constructor-arg>
	</bean>
</beans>

```
 （3）<list>、<set>、<map>标签  
 list标签可以将其中的值整合为一个List集合，其用法可以参考上面那个例子。  
 set标签也同样可以整合为一个Set集合，用法和list标签相同。  
 map标签则需要指定键和值：

 
```
<map>
	<entry key="key">
		<value>value</value>
	</entry>
</map>

```
 如果entry的键为引用类型，可以将key属性改为key-ref并指定bean的名称，对于value也相同。

 （4）depends-on属性  
 depends-on属性可以保证一个Bean在初始化前保证depends-on指定的Bean已经完成初始化。比如User类加载前需要Friends类先加载，那么可以：

 
```
<beans>
	<bean id="User" class="com.test.User" depends-on="UserFriends">
		<!--省略其它配置-->
	</bean>
	<bean id="UserFriends" class="com.test.Friends">
		<!--省略其它配置-->
	</bean>
</beans>

```
 
##### []()2.5 bean的继承

 
```
<beans>
    <bean id="User0" class="com.test.User">
    	<property name="name" value="Li"/>
		<property name="friends" ref="UserFriends" />
	</bean>
	<bean id="User1" class="com.test.User">
		<property name="name" value="Liu"/>
		<property name="friends" ref="UserFriends" />
	</bean>
	<bean id="UserFriends" class="com.test.Friends">
		<!--省略-->
	</bean>
</beans>

```
 可以看到，User0和User1有一个共同的配置，就是都依赖于UserFriends。如果按照上面那样配置，就看起来有些冗余。  
 所以，Spring引入了parent属性：

 
```
<bean id="User0" class="com.test.User">
	<property name="name" value="Li"/>
	<property name="friends" ref="UserFriends" />
</bean>
	
<bean id="User1" class="com.test.User" parent="User0">
	<property name="name" value="Liu"/>
</bean>

```
 这样，User1就继承了User0定义的默认值。  
 还有一种方法就是指定一个模板bean：

 
```
<bean id="User" abstract="true">
    <property name="friends" ref="UserFriends" />
</bean>
    
<bean id="User0" class="com.test.User" parent="User">
    <property name="name" value="Li"/>
</bean>
	
<bean id="User1" class="com.test.User" parent="User">
	<property name="name" value="Liu"/>
</bean>

```
 指定这个模板bean的abstract属性为true，说明这个bean无需实例化。

 
--------
 
### []()三、容器中的Bean

 
##### []()3.1 Bean的作用域

 Bean的作用域可以通过scope属性来指定。scope用来声明容器中的对象应该处的限定场景或者该对象的存活时间，当对象不处于scope的限定之后，容器会销毁这些对象。  
 Spring默认提供了5种scope类型：singleton、prototype、request、session、global session，后三种scope用于Web应用。  
 （1）singleton  
 scope声明为singleton的Bean在容器中只存在一个实例。这个Bean会在第一次请求后被实例化，只要容器没有被销毁，那么这个实例会一直存在容器中。  
 （2）prototype  
 scope声明为prototype的Bean在每次请求时都会生成一个新的对象，请求方需要自己管理这个对象的生命周期。  
 （3）request、session、global session  
 可以把这些对象的作用域理解为Web应用中Request、Session、Application的作用域。

 除了这些scope以外，我们还可以自定义作用域。如果需要实现自己的作用域，首先容器必须要实现ConfigureBeanFactory接口，满足这个条件后我们需要给出一个Scope接口的实现类，比如下面这个对象的作用域和单个线程的生命周期相同：

 
```
public class ThreadScope implements Scope {
	//每个线程维护一个Map，其键为Bean的名称，值为这个Bean的实例，该Map保存了这个线程拥有的所有Bean的实例
	private static final ThreadLocal<Map<String, Object>> scope = new ThreadLocal<Map<String, Object>>() {
		protected Map<String, Object> initialValue() {
			return new HashMap<>();
		}
	};
	//获取Bean的实例
	@Override public Object get(String name, ObjectFactory<?> objectFactory) {
		//获取这个线程维护的Map
		Map<String, Object> map = scope.get();
		Object o = map.get(name);
		//如果没有找到实例，那么生成一个实例并放入map后返回
		if(o == null) {
			o = objectFactory.getObject();
			map.put(name, o);
		}
		return o;
	}
	//销毁这个Bean的实例
	@Override public Object remove(String name) {
		return scope.get().remove(name);
	}
	//注册销毁对象时的回调方法
	@Override public void registerDestructionCallback(String name, Runnable callback) { }
	@Override public Object resolveContextualObject(String key) {
		return null;
	}
	//返回一个会话ID
	@Override public String getConversationId() {
		return null;
	}	
}

```
 然后，调用ConfigureBeanFactory方法的registerScope方法即可。

 
##### []()3.2 工厂方法

 Spring IoC容器支持通过工厂方法来构造对象并注入需要被依赖的对象  
 （1）静态工厂方法  
 以上面为例，如果我们需要一个新的User对象拥有一个任意的ID，我们可以调用工厂方法生成一个随机数并将其注入到id变量中：

 
```
public final class UserIDFactory {
	private static final Random random = new Random();
	public static int getRandomID() {
		return random.nextInt();
	}
}

```
 XML配置如下：

 
```
<beans>  
    <bean id="User" class="com.test.User">
    	<property name="id" ref="randomIDFactory"/>
    	<property name="friends" ref="UserFriends" />
    </bean>
    
    <bean id="randomIDFactory" class="com.test.UserIDFactory" factory-method="getRandomID" />
    
	<bean id="UserFriends" class="com.test.Friends">
		<!--省略配置-->
	</bean>
</beans>

```
 这样，每个生成的User对象就会有一个随机生成的ID。

 （2）非静态工厂方法  
 比如下面这个例子：

 
```
public class UserIDFactory {
	private final Random random = new Random();
	public int getRandomID() {
		return random.nextInt();
	}
}

```
 XML配置：

 
```
<bean id="User" class="com.test.User">
    <property name="id" ref="randomID"/>
    <property name="name" value="Li"/>
    <property name="friends" ref="UserFriends" />
</bean>
    
<bean id="randomIDFactory" class="com.test.UserIDFactory" />
<bean id="randomID" factory-bean="randomIDFactory" factory-method="getRandomID" />

```
 相比于静态工厂的配置，非静态工厂需要添加一个bean，并指定这个非静态工厂的Bean的名称和非静态方法。

 
##### []()3.3 FactoryBean

 FactoryBean是Spring IoC容器提供的一种可以扩展Bean实例化逻辑的接口，可以将它看成是一个生产Bean的工厂。  
 FactoryBean适用于对象实例化过于繁琐、XML配置过于复杂时使用，FactoryBean接口定义了3种方法：

 
```
public interface FactoryBean<T> {
	//返回生产出来的对象
	@Nullable T getObject() throws Exception;
	//获取生产对象的Class
	@Nullable Class<?> getObjectType();
	//是否在容器中是单例
	default boolean isSingleton() {
		return true;
	}
}

```
 泛型参数T为生产的对象类型，getObject方法返回FactoryBean生产的实例，getObjectType返回生产的对象类型，isSingleton表示这个Bean的scope是否为singleton。  
 比如我们将上面那个RandomIDFactory改为FactoryBean的实现：

 
```
public class RandomIDFactory implements FactoryBean<Integer> {
	private final Random rd = new Random();
	@Override
	public Integer getObject() throws Exception {
		return rd.nextInt();
	}
	@Override
	public Class<?> getObjectType() {
		return Integer.class;
	}
	public boolean isSingleton() {
		return false;
	}
}

```
 XML配置为：

 
```
<bean id="User" class="com.test.User">
    <property name="id" ref="RandomIDFactory"/>
    <property name="name" value="Li"/>
    <property name="friends" ref="UserFriends" />
</bean>

<bean id="RandomIDFactory" class="com.test.RandomIDFactory" />

```
 Spring会自动识别FactoryBean并将生产出来的实例注入到变量id中，如果一定需要取得com.test.RandomIDFactory本身这个对象，在BeanFactory的getBean方法的name参数中字符串前面加一个"&“即可，比如getBean(”&RandomIDFactory")

 
##### []()3.4 方法注入

 Spring容器可以通过方法注入来达到一种重写方法的效果。  
 比如我们现在想要调用User类的getId方法时每次都返回一个随机数而不是User对象的id属性，那么我们可以修改上面那个XML文件

 
```
<bean id="User" class="com.test.User">
	<property name="id" ref="RandomIDFactory"/>
    <property name="name" value="Li"/>
    <property name="friends" ref="UserFriends" />
    <!--只需要添加这个就可以了-->
    <lookup-method name="getId" bean="RandomIDFactory" />
</bean>
    
<bean id="RandomIDFactory" class="com.test.RandomIDFactory" />

```
 lookup-method的name属性需要指定注入的方法名，bean属性指定需要注入的对象，name的格式可以为：  
 <public | protected> [abstract] <return-type> method(no-argument)  
 说白了就是要求子类能够重写这个方法，如果用了final关键字修饰了方法是不能够被注入的。

 
##### []()3.5 方法替换

 方法替换和方法注入不一样，方法替换则更多地体现在方法的实现层上，它可以达到一种将原来的方法实现直接覆盖，或者说是一种方法拦截。  
 要想实现方法拦截，我们需要定义一个MethodReplacer接口的实现类，这个接口定义了一个方法：

 
```
public interface MethodReplacer {
	//obj为被替换的方法中的this引用，method为原先的方法对象，args为方法参数数组
	Object reimplement(Object obj, Method method, Object[] args) throws Throwable;
}

```
 reimplement方法就是新方法的逻辑。

 例如，要将getId方法覆盖掉：

 
```
public class GetIDReplacer implements MethodReplacer {
	private final Random rd = new Random();
	@Override
	public Object reimplement(Object obj, Method method, Object[] args) throws Throwable {
		return rd.nextInt();
	}
}

```
 XML配置：

 
```
<bean id="User" class="com.test.User">
    <property name="name" value="Li"/>
    <property name="friends" ref="UserFriends" />
    <replaced-method name="getId" replacer="NewGetId" />
</bean>

<bean id="NewGetId" class="com.test.Test.GetIDReplacer" />

```
 replace-method中，name为方法名，replacer为MethodReplacer实现类的Bean的名称。

 
--------
 
### []()四、容器的启动与Bean的初始化

 Spring IoC容器在启动过程中，会对配置信息进行解析，把每个bean的配置信息定义为一个BeanDefinition的实现类，并注册到BeanDefinitionRegistry中。当请求方需要获取Bean时，就会触发Bean的初始化（ApplicationContext默认会在初始化过程中全部初始化），随后，容器会根据BeanDefinition提供的信息实例化Bean，并为其注入依赖等一系列操作，装配完毕后，返回给调用方。

 
##### []()4.1 BeanFactoryPostProcessor

 Spring提供了一种基于BeanFactoryPostProcessor接口的容器扩展机制，它可以让请求方在获取对象前，对BeanDefition注册的信息做一定的修改。BeanFactoryPostProcessor接口只定义了一个方法：

 
```
@FunctionalInterface
public interface BeanFactoryPostProcessor {
	void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException;
}

```
 PropertyPlaceholderConfigure就是一个典型的BeanFactoryPostProcessor实现类，PropertyPlaceholderConfigure允许我们通过XML配置文件中使用占位符，并将这些占位符代表的资源配置到properties文件中来加载：

 
```
<bean class="org.springframework.beans.factory.config.PropertyPlaceholderConfigure">
	<property name="locations">
		<list>
			<value>res/jdbcinfo.properties</value>
		</list>
	</property>
</bean>

```
 然后定义一个JDBCDataSource的Bean来存储这些信息：

 
```
<bean id="JDBCDataSource" class="com.test.JDBCDataSource">
	<property name="url" value="${jdbc.url}">
	<property name="driveClass" value="${jdbc.driveClass}">
	<property name="username" value="${jdbc.username}">
	<property name="password" value="${jdbc.password}">
</bean>

```
 然后在classpath下建立一个res文件夹，新建一个jdbcinfo.properties文件：

 
```
jdbc.url=jdbc:mysql://localhost:3306/database
jdbc.driveClass=com.mysql.jdbc.Driver
jdbc.username=username
jdbc.password=password

```
 BeanFactory在完成所有BeanDefition的注册后，它的属性依然是以占位符的形式存在的，只有当实例化Bean后，properties文件定义的信息才会被注入到实例中。

 
##### []()4.2 Bean的生命周期

 Bean的生命周期可以概括为：  
 实例化Bean对象并设置对象属性  
 检查Aware接口并设置相关依赖  
 BeanPostProcesser前置处理  
 检查是否是InitializingBean以决定是否调用afterPropertiesSet方法  
 检查是否配置了自定义的init-method  
 BeanPostProcesser后置处理  
 注册必要的Destruction相关回调接口  
 对象使用完成后，检查是否实现DisposableBean

 （1）BeanWrapper  
 容器在实例化Bean时采用策略模式来决定用何种方式来实例化Bean。通常可以通过反射或者动态字节码技术来生成相关Bean或者动态子类。  
 BeanWrapper接口通常使用在Spring IoC容器内部，它的实现类为BeanWrapperImpl，可以对某个Bean进行包裹，然后对这个包裹的Bean进行操作。使用BeanWrapper对bean的操作很方便，无需直接通过反射API来操作对象。

 （2）Aware接口  
 对象实例化完成之后，会检查对象是否实现了Aware接口或其相关子接口，Spring提供了很多Aware子接口：  
 比如BeanNameAware、BeanClassLoaderAware、BeanFactoryAware。  
 Aware接口主要功能就是提醒容器将这个Bean需要依赖的对象注入，比如User类实现了BeanFactoryAware接口，那么容器就会将当前容器注入到对象中。

 （3）BeanPostProcesser  
 BeanPostProcesser和BeanFactoryPostProcessor不同，前者应用在对象实例化阶段，后者则是应用在容器初始化阶段。  
 BeanPostProcesser接口源码如下：

 
```
public interface BeanPostProcessor {
	@Nullable
	default Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
		return bean;
	}
	@Nullable
	default Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
		return bean;
	}
}

```
 BeanPostProcesser接口定义了两个方法，分别在不同的时机执行。postProcessBeforeInitialization方法会在BeanPostProcesser前置处理过程中执行，postProcessAfterInitialization则会在BeanPostProcesser后置处理过程中执行。

 在ApplicationContext中，如果需要添加自己的BeanPostProcesser，直接在XML文件中配置一个bean即可。

 （4）init-method  
 在BeanPostProcesser后置处理执行结束前，init-method指定的方法会被调用，init-method可以作为bean标签的属性。

 （5）destory-method  
 在BeanPostProcesser后置处理完成后，如果Bean实现类DisposableBean接口或指定了destory-method属性，那么容器会注册一个用于销毁对象的回调接口。比如对于singleton作用域的Bean而言，当ApplicationContext关闭后，destory-method定义的方法就会被执行。

   
  