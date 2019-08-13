---
title: Spring基础复习：AOP模块的使用及其基本结构
date: 2018-10-13 20:42:37
tags: CSDN迁移
---
 版权声明：尊重原创，转载请注明出处 [ https://blog.csdn.net/abc123lzf/article/details/83033681]( https://blog.csdn.net/abc123lzf/article/details/83033681)   
  **本文根据《Spring揭秘》 王福强著 总结**

 
### []()一、引言

 AOP全称Aspect-Oriented Programming，中文翻译为面向切面编程。通过使用AOP，我们可以简化项目需求与实现的对比关系，使整个系统更加模块化。Aspect之于AOP，相当于Class之于OOP。

 AOP在Java语言中有两种实现方式，一种是静态方式，另外一种是动态方式。静态AOP为第一代AOP，以AspectJ语言为代表，当相应的横切关注点以AspectJ语言实现后，会通过特定的编译器将这个横切逻辑以字节码形式织入到class文件中。  
 动态AOP有两种实现方式：一种是JDK自带的**动态代理**，Spring默认采用这种机制实现AOP，另外一种是通过**CGLIB库**动态生成class字节码并加载至虚拟机。

 在了解Spring AOP之前，我们先来介绍一些概念：

 
##### []()1.1 Joinpoint

 AOP在进行织入操作时必须要知道将逻辑织入在哪个执行点上，我们称这些将要在其之上**进行织入操作的系统执行点**就被称之为Joinpoint。完整的AOP功能有以下几个Joinpoint类型：  
 （1）方法调用：当某个方法被调用的时候所处的程序执行点。  
 （2）方法执行：这个Joinpoint类型代表的是某个方法内部开始执行的时点。和方法调用不同，方法调用要先于方法执行。比如在调用一个方法时，当这个方法还没有压入到方法栈时是"方法调用"的Joinpoint，压入到方法栈之后才是"方法执行"的Joinpoint。  
 （3）字段设置：对象的某个变量直接被设置的时点，比如：A.value = 10。  
 （4）异常处理执行：在程序执行过程中，某些异常类型抛出后，对应的异常处理逻辑执行的时点  
 （5）类初始化：在类中某些静态类型或者执行静态初始化块时点。

 
##### []()1.2 Pointcut

 Pointcut代表的是Joinpoint的**表述方式**。将横切逻辑织入到当前系统的过程中，需要参照Pointcut规定的Joinpoint信息，才可以知道应该往系统的哪些Joinpoint上织入横切逻辑。  
 Pointcut表述方式一般有：直接指定Joinpoint所在的方法名称，正则表达式和使用特定的Pointcut表述语法。  
 Pointcut与Pointcut之间还可以进行逻辑运算。

 
##### []()1.3 Advice

 Advice是**单一横切关注点逻辑**的载体，它代表将会织入到Joinpoint的横切逻辑。Aspect比作Java中的Class，那么Advice就相当于Java中的方法。按照Advice在Joinpoint执行时机的差异或者完成功能的不同，Advice可以分为多种形式：  
 （1）Before Advice  
 Before Advice是在Joinpoint指定位置之前执行的Advice类型。如果当前Before Advice将被织入到方法执行类型的Joinpoint，那么这个Before Advice就会先于方法执行而执行。  
 （2）After Advice  
 After Advice就是在相应连接点后执行的Advice类型，这种Advice还可以细分为：After Returning Advice（只有当前Joinpoint处执行流程正常结束没有抛出异常后，After Returning Advice才会正常地执行），After Throwing Advice（当前Joinpoint执行过程中抛出了异常的情况下才会执行），After Advice（不论是否抛出异常都会执行，类似finally语句）。  
 （3）Around Advice  
 Around Advice会对附加其上的Joinpoint进行**包裹**，可以在Joinpoint之前和之后都指定相应的逻辑，或者也可以中断或者忽略Joinpoint处原来程序流程的执行。  
 （4）Introduction  
 Introduction不是根据横切逻辑在Joinpoint处的执行时机来区分的，而是根据它的可以完成的功能而区别于其它Advice类型。Introduction可以为原有的对象添加新的特性或者行为。

 
##### []()1.4 Aspect

 Aspect是对系统中的横切关注点逻辑进行模块化的AOP概念实体。通常，一个Aspect可以包含多个Pointcut以及相关Advice定义。在Spring AOP中可以通过@Aspect注解来标明一个Aspect

 
--------
 
### []()二、Spring AOP中的Joinpoint、Pointcut、Advice、Aspect

 Spring AOP仅支持方法级别的Joinpoint，更确切地说，仅支持方法执行类型的Joinpoint，这对于大多数需求而言，已经足够了。如果需要更完善的AOP支持，可以考虑AspectJ。

 
##### []()2.1 Spring AOP中的Pointcut

 Spring以接口定义org.springframework.aop.Pointcut作为AOP模块中**所有**Pointcut的最顶层抽象：

 
```
public interface Pointcut {
	ClassFilter getClassFilter();
	MethodMatcher getMethodMatcher();
	Pointcut TRUE = TruePointcut.INSTANCE;
}

```
 Pointcut定义了两个方法：getClassFilter和getMethodMatcher，前者用于匹配Java中的类，后者用于匹配Java中的方法。并且还定于了一个Pointcut实现类的实例TRUE，这个常量默认匹配所有的类以及所有的方法。

 ClassFilter接口定义了匹配类的方法matches：

 
```
@FunctionalInterface public interface ClassFilter {
	boolean matches(Class<?> clazz);
	ClassFilter TRUE = TrueClassFilter.INSTANCE;
}

```
 当织入的目标对象的Class类型与Pointcut所规定的类型相符时，matches方法会返回true，否则返回false，代表不会对这个Class进行织入。

 MethodMatcher定义的匹配方法则稍显复杂些：

 
```
public interface MethodMatcher {
	boolean matches(Method method, Class<?> targetClass);
	boolean isRuntime();
	boolean matches(Method method, Class<?> targetClass, Object... args);
	MethodMatcher TRUE = TrueMethodMatcher.INSTANCE;
}

```
 MethodMatcher通过重载定义了两个matches方法，这两个方法的执行分界线为isRuntime方法。

 在对方法进行匹配检查的时候，如果不需要检查Joinpoint传入的参数，那么isRuntime方法可以恒返回false，这种类型的MethodMatcher可以称之为**StaticMethodMather**，这种MethodMather不会调用boolean matches(Method method, Class<?> targetClass, Object… args)这个方法，以提高执行效率。  
 如果需要检查Joinpoint传入的参数，那么isRuntime可以恒返回true，这种MethodMather称为**DynamicMethodMather**。

 在MethodMatcher基础上，Pointcut的类型就可以分为StaticMethodMatherPointcut和DynamicMethodMatherPointcut。

 Spring AOP提供了很多常用的Pointcut实现类，其结构图如下：  
 ![在这里插入图片描述](https://img-blog.csdn.net/2018101312410470?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FiYzEyM2x6Zg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

 （1）NameMathchMethodPointcut  
 这是最简单的Pointcut实现，属于StaticMethodMathcerPointcut的子类，可以根据自身指定的一组方法名称与Joinpoint处的方法的方法名称进行匹配，如：

 
```
NameMathchMethodPointcut pointcut = new NameMathchMethodPointcut();
pointcut.setMappedName("setId","get*"); //可传入多个字符串

```
 NameMathchMethodPointcut除了可以直接指定方法名，还可以使用’*'通配符进行简单的模糊匹配

 需要注意的是，NameMathchMethodPointcut无法对重载的方法名进行匹配。

 （2）JdkRegexpMethodPointcut  
 JdkRegexpMethodPointcut是一个基于JDK的正则表达式引擎进行匹配的MethodPointcut，基本用法如下：

 
```
JdkRegexpMethodPointcut pointcut = new JdkRegexpMethodPointcut();
pointcut.setPattern(".*get.*"); //可传入多个字符串

```
 使用正则表达式进行匹配方法时，其匹配模式必须是整个方法签名，而不能仅仅给出方法名称。比如上述Pattern可以匹配public void getMethod()或者private String getStr()之类。

 早期Spring还提供了Perl5RegxpMethodPointcut用于实现和JdkRegexpMethodPointcut类似的正则匹配功能，不过从最新的Spring 5来看，已经删去了Perl5RegxpMethodPointcut这个类。

 （3）AnnotationMatchingPointcut  
 从名字上来看，AnnotationMatchingPointcut根据目标中是否有指定的注解来匹配Joinpoint，要使用它，首先需要声明相应的注解：

 
```
@Retention(RetentionPolicy.RUNTIME) @Target(ElementType.TYPE)
public @interface MyClassPointcut {}

@Retention(RetentionPolicy.RUNTIME) @Target(ElementType.Method)
public @interface MyMethodPointcut {}

@MyClassPointcut
public class Target {
	@MyMethodPointcut
	public void method() {
		System.out.println("method Run");
	}
}

```
 MyClassPointcut为类级别的注解，MyMethodPointcut为方法级别的注解。  
 如果需要将一个标注了MyClassPointcut注解的类中所有的方法都进行匹配，那么可以：

 
```
AnnotationMatchingPointcut pointcut = new AnnotationMatchingPointcut(MyClassPointcut.class);
//或者
AnnotationMatchingPointcut pointcut = AnnotationMatchingPointcut.forClassAnnotation(MyClassPointcut.class);

```
 如果仅需要匹配标注了MyMethodPointcut注解的方法，那么可以：

 
```
AnnotationMatchingPointcut pointcut = AnnotationMatchingPointcut.forMethodAnnotation(MyMethodPointcut.class);

```
 如果需要两者同时指定：

 
```
AnnotationMatchingPointcut pointcut = new AnnotationMatchingPointcut(
		MyClassPointcut.class, MyMethodPointcut.class);

```
 （4）ComposablePointcut  
 ComposablePointcut可以为Pointcut提供逻辑运算功能，比如基本的并运算和交运算，比如：

 
```
ComposablePointcut pointcut = new ComposablePointcut(classFilter, methodMatcher);

```
 可以将ClassFilter和MethodMathcer合并在一起组成一个Pointcut，同时也可以传入一个Pointcut实现类。

 如果只需要进行Pointcut和Pointcut之间的逻辑运算，Pointcuts类就可以满足了。

 如果上面的Pointcut无法满足业务需求的话，也是可以自定义Pointcut的。

 
##### []()2.2 Spring AOP中的Advice

 Advice实现了将被织入到Pointcut规定的Joinpoint处的横切逻辑。在Spring中，Advice按照自身实例能否在目标对象类的所有实例中共享这一标准，可以划分为两类：per-class类型Advice和per-instance类型的Advice

 Spring AOP Advice结构图：  
 ![在这里插入图片描述](https://img-blog.csdn.net/20181013145208456?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FiYzEyM2x6Zg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

 []()2.2.1 per-class类型的Advice per-class类型的Advice实例可以在目标对象类的所有实例间进行共享，它只能提供方法拦截的功能，不会为目标对象类保存任何状态或者添加任何新的特性。  
 Spring AOP将org.aopalliance.aop.Advice作为Advice的根接口。

 （1）BeforeAdvice  
 前面已经提过，Before Advice是在Joinpoint指定位置之前执行的Advice类型。Before Advice执行完成后，程序执行流程将从Joinpoint处继续执行，通常而言Before Advice不会打断程序的执行流程，如果有必要可以通过抛异常来中止。

 Spring AOP提供了org.springframework.aop.MethodBeforeAdvice接口来实现BeforeAdvice：

 
```
public interface MethodBeforeAdvice extends BeforeAdvice {
	void before(Method method, Object[] args, @Nullable Object target) throws Throwable;
}

```
 其中，参数method为Joinpoint处的方法对象，args为目标方法的传入参数，target为该方法是通过哪个对象调用的。  
 MethodBeforeAdvice接口继承了BeforeAdvice接口，BeforeAdvice接口仅仅只是一个标记接口，没有定义方法。

 （2）ThrowsAdvice  
 Spring以接口定义org.springframework.aop.ThrowsAdvice对应AOP概念中的AfterThrowingAdvice。  
 该接口没有定义任何方法，但是我们在实现相应的ThrowsAdvice，我们的方法定义需要遵循以下规则：

 
```
void afterThrowing([Method, args, target], ThrowableSubclass);

```
 比如以下方法都是合法的：

 
```
public void afterThrowing(Throwable t);
public void afterThrowing(RuntimeException t);
public void afterThrowing(Method m, Object[] args, Object target, Exception t);

```
 （3）AfterReturningAdvice  
 org.springframework.aop.AfterReturningAdvice接口定义了AOP中的AfterReturningAdvice：

 
```
public interface AfterReturningAdvice extends AfterAdvice {
	void afterReturning(@Nullable Object returnValue, Method method, 
			Object[] args, @Nullable Object target) throws Throwable;
}

```
 在afterReturning方法中，我们可以访问到当前Joinpoint方法的返回值、方法对象、方法参数以及所在的对象。

 （4）AroundAdvice  
 Spring采用了AOP Alliance接口org.aopalliance.intercept.MethodInterceptor来实现AroundAdvice：

 
```
@FunctionalInterface
public interface MethodInterceptor extends Interceptor {
	Object invoke(MethodInvocation invocation) throws Throwable;
}

```
 通过MethodInterceptor的invoke方法的MethodInvocation参数，我们可以控制相应的Joinpoint的拦截行为。通过调用MethodInvocation的proceed方法，可以让程序继续沿着调用链执行。如果我们没有调用proceed方法，那么Joinpoint处的方法就不会被执行。  
 通过proceed方法，我们可以捕获Joinpoint方法抛出的异常。

 
##### []()2.2.2 per-instance类型的Advice

 per-instance类型的Advice不会在目标类所有对象实例之间进行共享，而是会为不同的实例对象保存它们各自的状态以及相关逻辑。在Spring AOP中，Introduction是唯一的一种per-instance类型的Advice。

 Introduction可以在不改动目标类定义的情况下，为目标类添加新的属性以及行为。Spring提供了org.springframework.aop.IntroductionInterceptor接口作为拦截器，通过它将新的接口定义机器实现类的逻辑附加到目标对象上：

 
```
public interface IntroductionInterceptor extends MethodInterceptor, DynamicIntroductionAdvice {
}

public interface DynamicIntroductionAdvice extends Advice {
	boolean implementsInterface(Class<?> intf);
}

```
 IntroductionInterceptor继承了MethodInterceptor接口和DynamicIntroductionAdvice接口。我们可以界定当前的IntroductionInterceptor为哪些接口类提供相应的拦截。通过MethodInterceptor，IntroductionInterceptor就可以处理新添加的接口上的方法调用了。通常，对于IntroductionInterceptor来说，如果是新添加的接口上的方法调用，不必去调用MethodInterceptor的proceed方法。

 
##### []()2.3 Spring AOP中的Aspect

 Advisor代表Spring中的Aspect，与正常的Aspect不同，Advisor通常只持有一个Pointcut和一个Advice，而Aspect要求可以定义多个Pointcut和多个Advice，可以认为Advisor是一种特殊的Aspect。

 在Spring AOP中，Advisor分为两种：PointcutAdvisor和IntroductionAdvisor，其基本结构如下：  
 ![在这里插入图片描述](https://img-blog.csdn.net/20181013171412311?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FiYzEyM2x6Zg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

 **PointcutAdvisor**  
 Spring AOP中大部分Advisor都实现了org.springframework.aop.PointcutAdvisor接口。  
 （1）DefaultPointcutAdvisor  
 DefaultPointcutAdvisor是最通用的PointcutAdvisor接口实现。除了不能指定Introduction类型的Advice，剩下的任何类型的Pointcut、任何类型的Advice都可以通过DefaultPointcutAdvisor来使用：

 
```
DefaultPointcutAdvisor advisor = new DefaultPointcutAdvisor(pointcut, advice);
//或者
DefaultPointcutAdvisor advisor = new DefaultPointcutAdvisor()
advisor.setPointcut(pointcut);
advisor.setAdvice(advice);

```
 在构造DefaultPointcutAdvisor，可以指定属于这个DefaultPointcutAdvisor的Pointcut和Advice，也可以在DefaultPointcutAdvisor实例构造完成后，再通过setPointcut以及setAdvice方法设置相应的Pointcut和Advice。当然，也可以通过IoC容器进行注册管理。

 （2）NameMatchMethodPointcutAdvisor  
 NameMatchMethodPointcutAdvisor是细化后的DefaultPointcutAdvisor，它限制了Pointcut类型只可为NameMatchMethodPointcut且外部不能更改。

 （3）RegexpMethodPointcutAdvisor  
 RegexpMethodPointcutAdvisor也限定了Pointcut，即只可通过正则表达式为其设置相应的Pointcut，它的内部已经持有一个AbstractRegexpMethodPointcut实例。

 **IntroductionAdvisor**  
 IntroductionAdvisor和PointcutAdvisor最本质上的区别就是，IntroductionAdvisor只能用于类级别的拦截，只能使用Introduction型的Advice。这个接口的实现类只有DefaultIntroductionAdvisor。

 
##### []()2.4 Ordered接口的作用

 实际项目中，只存在单一横切关注点的情况较少，大多数时候都会有多个横切点需要处理，那么就会有多个Advisor存在。当某些Advisor匹配到同一个Joinpoint的时候，就会在这同一个Joinpoint处执行多个Advice的横切逻辑。如果需要保证这些Advisor的执行顺序的话，就需要通过Ordered接口定义的优先级来排列这些Advisor了。

 Spring在处理同一Joinpoint的多个Advisor的时候，实际上会按照指定的顺序和优先级来执行它们，顺序号越小，优先级越高，我们可以从0开始指定（小于0的顺序号原则上由Spring框架内部使用）。如果没有指定这些优先级，那么Spring会按照它们声明的顺序来应用它们。

 
--------
 
### []()三、Spring AOP的织入

 了解完Joinpoint、Pointcut、Advice、Aspect后，我们接下来就是将这些模块拼接并织入。

 
##### []()3.1 ProxyFactory

 ProxyFactory是Spring AOP的一个基本的织入器。Spring AOP是基于代理模式的AOP实现，织入过程完成后，会返回织入了横切逻辑的目标对象的代理对象。使用ProxyFactory只需要指定两个东西：  
 （1）要进行织入的目标对象。可以为一个类的对象，也可以为类本身（Class对象）。  
 （2）将要应用到目标对象的Advisor。  
 对于Introduction类型之外Advice，ProxyFactory内部会为这些Advice构造相应的Advisor，只不过在为它们构造的Advisor中使用的Pointcut为Pointcut.TRUE。  
 对于Introduction类型的Advice而言，如果是IntroductionInfo的子类，它本身包含了必要的信息，框架内部会给它构造一个DefaultIntroductionAdvisor。而如果是DynamicIntroductionAdvice子类实现，将抛出异常。

 **基于接口的代理**  
 为了演示ProxyFactory的使用方法，我们定义一个类和接口：

 
```
public interface Eatable {
	void eat();
}

public class Person implements Eatable{
	public void eat() {
		System.out.println("eating");
	}
}

```
 Person类的eat方法为Joinpoint。  
 然后我们再定义一个Advice实现类：

 
```
public static class MyAdvice implements MethodBeforeAdvice {
	@Override
	public void before(Method method, Object[] args, Object target) throws Throwable {
		System.out.println("before eating");
	}
}

```
 最后是主方法：

 
```
public static void main(String[] args) {
	//构造Pointcut，定义切入点为eat方法
	NameMatchMethodPointcut pointcut = new NameMatchMethodPointcut();
	pointcut.addMethodName("eat");
	//构造MyAdvice和Advisor
	DefaultPointcutAdvisor advisor = new DefaultPointcutAdvisor(pointcut, new MyAdvice());
	Person person = new Person();
	ProxyFactory factory = new ProxyFactory(person);
	factory.setInterfaces(Eatable.class);
	factory.addAdvisor(advisor);
	Eatable eatable = (Eatable)factory.getProxy();
	eatable.eat();
}

```
 输出结果为：

 
```
before eating
eating

```
 上面的案例就是ProxyFactory的基于接口的代理。  
 ProxyFactory的setInterfaces方法可以告知ProxyFactory我们要对Eatable接口类型进行代理。如果没有其它行为属性进行干预，我们也可以不指定setInterfaces方法指定接口。  
 如果目标类实现了至少一个接口，不管我们有没有通过ProxyFactory的setInterfaces方法指明，只要不将optimize和proxyTargetClass两个属性设为true，ProxyFactory都会按照面向接口进行代理。

 **基于类的代理**  
 如果目标类没有实现任何接口，那么默认情况下会采用基于类的代理，即通过CGLIB动态生成字节码。在上面那个例子中，如果我们把Eatable接口去掉，输出结果依然不会改变。  
 如果即使是实现了接口还是一定需要使用类的代理，那么在调研getProxy方法之前，可以调用ProxyFactory的setProxyTargetClass方法。

 
```
public static void main(String[] args) {
	NameMatchMethodPointcut pointcut = new NameMatchMethodPointcut();
	pointcut.addMethodName("eat");
	DefaultPointcutAdvisor advisor = new DefaultPointcutAdvisor(pointcut, new MyAdvice());
	Person person = new Person();
	ProxyFactory factory = new ProxyFactory(person);
	factory.setProxyTargetClass(true);
	factory.addAdvisor(advisor);
	Person eatable = (Person)factory.getProxy();
	eatable.eat();
}

```
 
##### []()3.2 ProxyFactoryBean

 ProxyFactoryBean可以使Spring IoC容器和Spring AOP互相结合。通过Spring IoC容器，我们可以在容器中对Pointcut和Advice进行管理。ProxyFactoryBean和ProxyFactory使用并无太大差别。

 从名字上看，ProxyFactoryBean可以理解为生产Proxy的FactoryBean，如果容器中某个对象依赖于ProxyFactoryBean，那么它将会使用到ProxyFactoryBean的getObject方法所返回的代理对象。

 ![在这里插入图片描述](https://img-blog.csdn.net/20181013192157762?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FiYzEyM2x6Zg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

 ProxyFactoryBean继承了ProxyFactory共有的父类的ProxyCreatorSupport，ProxyCreatorSupport已经把设置目标对象、配置其它部件、生成对应的AopProxy等工作都做完了。

 如果ProxyFactoryBean的singleton属性设为true，则ProxyFactoryBean在第一次生成代理对象后，会通过内部实例变量singletonInstance缓存生成的代理对象。

 我们以上面的例子，来通过IoC容器配置ProxyFactoryBean：

 
```
<beans>
    <bean id="Person" class="com.test.Person" />
    <!--配置切入点为eat方法-->
    <bean id="PersonPointcut" class="org.springframework.aop.support.NameMatchMethodPointcut">
    	<property name="mappedName" value="eat" />
    </bean>
    <!--配置Advice，即切入逻辑-->
    <bean id="PersonAdvice" class="com.test.MyAdvice" />
    <!--配置Advisor，将Advice和Pointcut组装起来-->
    <bean id="PersonAdvisor" class="org.springframework.aop.support.DefaultPointcutAdvisor">
    	<property name="pointcut" ref="PersonPointcut"/>
    	<property name="advice" ref="PersonAdvice" />
    </bean>
    
    <bean id="PersonProxyFactory" class="org.springframework.aop.framework.ProxyFactoryBean">
    	<!--设置目标代理对象为Person-->
    	<property name="target" ref="Person" />
    	<!--配置Advisor为PersonAdvisor-->
    	<property name="interceptorNames">
    		<list>
    			<value>PersonAdvisor</value>
    		</list>
    	</property>
    </bean>
</beans>

```
 
```
public static void main(String[] args) {
	ClassPathXmlApplicationContext ctx = new ClassPathXmlApplicationContext("applicationContext.xml");
	Person person = ctx.getBean("PersonProxyFactory", Person.class);
	person.eat();
	ctx.close();
}

```
 输出结果与上述例子相同。

 基于注解的AOP使用方法，可以参考这篇博客：[https://www.cnblogs.com/liuruowang/p/5711563.html](https://www.cnblogs.com/liuruowang/p/5711563.html)

   
  