---
title: Spring 5.0源码解读：Bean容器DefaultListableBeanFactory
date: 2019-04-21 19:22:08
tags: CSDN迁移
---
 版权声明：尊重原创，转载请注明出处 [ https://blog.csdn.net/abc123lzf/article/details/89388481]( https://blog.csdn.net/abc123lzf/article/details/89388481)   
  #### []()引言

 
--------
 在Spring中， `BeanFactory` 是IoC容器的核心接口，它负责管理所有的Bean，解决各种Bean与Bean之间的依赖关系。 `BeanFactory` 接口有一个经典的实现类就是 `DefaultListableBeanFactory` ：  
 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190418211756953.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FiYzEyM2x6Zg==,size_16,color_FFFFFF,t_70)  
   


 
#### []()测试代码

 
--------
 
```
@Test
public void beanFactoryTest() {
	Resource xmlSrc = new ClassPathResource("test/context.xml");
	BeanFactory bf = new DefaultListableBeanFactory();
	BeanDefinitionReader bdr = new XmlBeanDefinitionReader((BeanDefinitionRegistry) bf);
	bdr.loadBeanDefinitions(xmlSrc);
	TestBean tb = bf.getBean("testBean", TestBean.class);
	System.out.println(tb);
}

```
 在上篇文章中我们已经学习到了 `XmlBeanDefinitionReader` 是如何加载Bean XML配置文件并写入到 `BeanDefinitionRegistry` ，那么本篇文章我们来分析 `DefaultListableBeanFactory` 是如何注册Bean、加载Bean的。

   
 
#### []()BeanDefinition的注册

 
--------
 在 `XmlBeanDefinitionReader` 解析XML文档的过程中需要将解析后的产物 `BeanDefinition` 写入到 `BeanDefinitionRegistry` ，那么 `DefaultListableBeanFactory` 是如何实现这个功能的呢？答案就在 `registerBeanDefinition` 这个方法中：

 
```
@Override
public void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)
		throws BeanDefinitionStoreException {
	Assert.hasText(beanName, "Bean name must not be empty");
	Assert.notNull(beanDefinition, "BeanDefinition must not be null");

	if (beanDefinition instanceof AbstractBeanDefinition) { 
		try {
			//进行Bean的验证操作，判断其属性是否合法
			((AbstractBeanDefinition) beanDefinition).validate();
		} catch (BeanDefinitionValidationException ex) {
			throw new BeanDefinitionStoreException(beanDefinition.getResourceDescription(), beanName,
						"Validation of bean definition failed", ex);
		}
	}
	BeanDefinition existingDefinition = this.beanDefinitionMap.get(beanName);
	if (existingDefinition != null) {	//如果有相同Bean
		if (!isAllowBeanDefinitionOverriding()) {  //如果不允许被覆盖，那么抛出异常
			throw new BeanDefinitionOverrideException(beanName, beanDefinition, existingDefinition);
		} else if (existingDefinition.getRole() < beanDefinition.getRole()) {
			if (logger.isInfoEnabled()) {
				logger.info("Overriding user-defined bean definition for bean '" + beanName +
						"' with a framework-generated bean definition: replacing [" +
						existingDefinition + "] with [" + beanDefinition + "]");
			}
		} else if (!beanDefinition.equals(existingDefinition)) {
			if (logger.isDebugEnabled()) {
				logger.debug("Overriding bean definition for bean '" + beanName +
						"' with a different definition: replacing [" + existingDefinition +
						"] with [" + beanDefinition + "]");
			}
		} else {
			if (logger.isTraceEnabled()) {
				logger.trace("Overriding bean definition for bean '" + beanName +
						"' with an equivalent definition: replacing [" + existingDefinition +
						"] with [" + beanDefinition + "]");
			}
		}
		//将bean id和BeanDefinition关系存入Map中
		this.beanDefinitionMap.put(beanName, beanDefinition);
	} else {
		if (hasBeanCreationStarted()) { //如果BeanFactory已经开始加载Bean
			synchronized (this.beanDefinitionMap) { //给Map加锁
				this.beanDefinitionMap.put(beanName, beanDefinition); //将bean id和BeanDefinition关系存入Map中
				List<String> updatedDefinitions = new ArrayList<>(this.beanDefinitionNames.size() + 1);
				//将这个Bean的ID、所有的别名添加到List
				updatedDefinitions.addAll(this.beanDefinitionNames);
				updatedDefinitions.add(beanName);
				this.beanDefinitionNames = updatedDefinitions;
				if (this.manualSingletonNames.contains(beanName)) {
					Set<String> updatedSingletons = new LinkedHashSet<>(this.manualSingletonNames);
					updatedSingletons.remove(beanName);
					this.manualSingletonNames = updatedSingletons;
				}
			}
		} else { //如果BeanFactory未处于初始化/运行状态，那么就无需保证其线程安全性
			this.beanDefinitionMap.put(beanName, beanDefinition);
			this.beanDefinitionNames.add(beanName);
			this.manualSingletonNames.remove(beanName);
		}
		this.frozenBeanDefinitionNames = null;
	}

	if (existingDefinition != null || containsSingleton(beanName)) {
		resetBeanDefinition(beanName);
	}
}

```
   
 
#### []()BeanFactory加载Bean的过程

 
--------
 当所有的 `BeanDefinition` 成功注册到 `DefaultListableBeanFactory` 后，接下来便可以调用它的相关方法获取Bean了。  
 当我们调用 `BeanFactory` 的 `getBean` 等方法时， `BeanFactory` 就会加载用户所需要的Bean。  
 首先说明下， `DefaultListableBeanFactory` 是一个线程安全的类，可以支持多个线程同时操作 `DefaultListableBeanFactory` ，所以其内部代码采用了很多多线程设计的思想。

 我们从 `getBean(String name, Class<T> requiredType)` 方法开始解析：

 
```
@Override
public <T> T getBean(String name, Class<T> requiredType) throws BeansException {
	return doGetBean(name, requiredType, null, false);
}

protected <T> T doGetBean(final String name, @Nullable final Class<T> requiredType,
			@Nullable final Object[] args, boolean typeCheckOnly) throws BeansException {
	//...
}

```
 `doGetBean` 方法代码量比较大，可以看出Bean的加载还是一个比较复杂的一个过程，我们来分步解析这个方法：

 []()1、转换name为beanName 
```
final String beanName = transformedBeanName(name);

```
 首先 `doGetBean` 会通过调用 `transformedBeanName` 方法处理传入的name参数。

 
```
protected String transformedBeanName(String name) {
	return canonicalName(BeanFactoryUtils.transformedBeanName(name));
}

public static String transformedBeanName(String name) {
	Assert.notNull(name, "'name' must not be null");
	if (!name.startsWith(BeanFactory.FACTORY_BEAN_PREFIX)) {
		return name;
	}
	//将name属性的前面“&”字符裁剪掉
	return transformedBeanNameCache.computeIfAbsent(name, beanName -> {
		do {
			beanName = beanName.substring(BeanFactory.FACTORY_BEAN_PREFIX.length());
		} while (beanName.startsWith(BeanFactory.FACTORY_BEAN_PREFIX));
			return beanName;
	});
}

public String canonicalName(String name) {
	String canonicalName = name;
	String resolvedName;
	do { //循环查找Bean的唯一ID(beanName)
		resolvedName = this.aliasMap.get(canonicalName);
		if (resolvedName != null) {
			canonicalName = resolvedName;
		}
	}
	while (resolvedName != null);
	return canonicalName;
}

```
 回顾一下Spring的基础知识，如果一个Bean属于 `FactoryBean` 类型，那么在传入这个Bean的name时，获取的是 `FactoryBean` 调用 `getObject` 方法的对象，如果需要获取 `FactoryBean` 本身，那么需要在name前加入字符’&’。

 getBean方法除了可以传入Bean的ID来获取Bean，还可以传入Bean的别名来获取。在Spring中，Bean可以取别名，别名同样可以再取一个别名，这种关系通过成员变量 `aliasMap` 保存。所以可以看到上述 `canonicalName` 方法通过一个循环来层层递进，找到这个Bean的唯一标识符ID。

 []()2、尝试从当前BeanFactory的缓存中找到单例 
```
Object bean;
Object sharedInstance = getSingleton(beanName);	//检查缓存或者实例工厂是否有对应的实例
if (sharedInstance != null && args == null) {
	if (logger.isTraceEnabled()) {
		if (isSingletonCurrentlyInCreation(beanName)) {
			logger.trace("Returning eagerly cached instance of singleton bean '" + beanName +
				"' that is not fully initialized yet - a consequence of a circular reference");
		} else {
			logger.trace("Returning cached instance of singleton bean '" + beanName + "'");
		}
	}
	bean = getObjectForBeanInstance(sharedInstance, name, beanName, null); //返回对应的实例
}

```
 首先会调用 `getSingleton` 尝试从缓存中寻找是否有对应的实例：

 
```
@Override @Nullable
public Object getSingleton(String beanName) {
	return getSingleton(beanName, true);
}

@Nullable
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
	//从缓存中找到实例
	Object singletonObject = this.singletonObjects.get(beanName);
	//如果没有在缓存中找到实例并且当前实例处于正在创建的状态
	if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
		synchronized (this.singletonObjects) {
			singletonObject = this.earlySingletonObjects.get(beanName);
			if (singletonObject == null && allowEarlyReference) {
				ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
				if (singletonFactory != null) {
					singletonObject = singletonFactory.getObject();
					this.earlySingletonObjects.put(beanName, singletonObject);
					this.singletonFactories.remove(beanName);
				}
			}
		}
	}
	return singletonObject;
}

```
 这里为什么会出现没有在缓存中找到实例并且该实例处于正在创建的状态呢？因为这是Spring解决Singleton Bean的循环依赖的机制产生的，Spring解决循环依赖的方式是通过提前曝光的方式（具体是将可以创建这个单例的 `ObjectFactory` 实例添加到 `singletonFactories` ），当 `BeanFactory` 检测到循环依赖后，就会将依赖的对象提前创建好，并添加到 `earlySingletonObjects` 中。需要注意的是， `singletonObjects` 缓存只负责存放**已经完全生成好**的单例Bean实例。

 如果 `getSingleton` 成功获取到了单例Bean对象，那么接下来就会调用 `getObjectForBeanInstance` 对这个对象进行进一步的处理：

 
```
protected Object getObjectForBeanInstance(Object beanInstance, String name, 
		String beanName, @Nullable RootBeanDefinition mbd) {
	//如果name以'&'开头
	if (BeanFactoryUtils.isFactoryDereference(name)) {
		if (beanInstance instanceof NullBean) {
			return beanInstance;
		}
		if (!(beanInstance instanceof FactoryBean)) {
			throw new BeanIsNotAFactoryException(transformedBeanName(name), beanInstance.getClass());
		}
	}
	//如果实例不属于FactoryBean类型或者name以'&'开头，那么直接返回实例
	if (!(beanInstance instanceof FactoryBean) || BeanFactoryUtils.isFactoryDereference(name)) {
		return beanInstance;
	}
	//如果是属于FactoryBean类型的Bean
	Object object = null;
	if (mbd == null) { //尝试从缓存中获取FactoryBean生产的实例
		object = getCachedObjectForFactoryBean(beanName);
	}
	if (object == null) { //如果没有从缓存中拿到FactoryBean实例
		FactoryBean<?> factory = (FactoryBean<?>) beanInstance;
		//是否指定了FactoryBean的ID(beanName)，如果指定了就获取它的RootBeanDefinition
		if (mbd == null && containsBeanDefinition(beanName)) {
			mbd = getMergedLocalBeanDefinition(beanName);
		}
		//检查是否是用户定义的而不是应用程序定义的
		boolean synthetic = (mbd != null && mbd.isSynthetic());
		object = getObjectFromFactoryBean(factory, beanName, !synthetic);
	}
	return object;
}

```
 在上述方法中，如果检测到name以’&'开头并且这个Bean不属于 `FactoryBean` ，那么就会抛出异常，否则，直接返回这个Bean的实例。  
 接下来，便是对 `FactoryBean` 的处理了。首先会尝试获取这个Bean的 `RootBeanDefinition` 对象，并判断它是否是“合成”的（合成的Bean即不是由应用程序本身定义的Bean，例如通过AOP代理后生成的Bean）。然后会尝试调用 `getCachedObjectForFactoryBean` 从缓存中拿到 `FactoryBean` 生产的实例：

 
```
@Nullable
protected Object getCachedObjectForFactoryBean(String beanName) {
	return this.factoryBeanObjectCache.get(beanName);
}

```
 如果没有成功从缓存中拿到实例，那么接下来就会调用 `getObjectFromFactoryBean` 方法获取实例：

 
```
protected Object getObjectFromFactoryBean(FactoryBean<?> factory, String beanName, boolean shouldPostProcess) {
	//如果生产的实例的是单例并且单例缓存中包含这个FactoryBean的实例
	if (factory.isSingleton() && containsSingleton(beanName)) {
		synchronized (getSingletonMutex()) {
			Object object = this.factoryBeanObjectCache.get(beanName);
			if (object == null) {
				//该方法会调用FactoryBean的getObject方法生产实例
				object = doGetObjectFromFactoryBean(factory, beanName);
				//确保只有一个实例
				Object alreadyThere = this.factoryBeanObjectCache.get(beanName);
				if (alreadyThere != null) {
					object = alreadyThere;
				} else {
					if (shouldPostProcess) {
						if (isSingletonCurrentlyInCreation(beanName)) { //解决循环依赖
							return object;
						}
						//将BeanName放入singletonsCurrentlyInCreation集合，表示正在处理这个Bean
						beforeSingletonCreation(beanName);
						try {
							//获取当前BeanFactory持有的BeanPostProcessor进行处理
							object = postProcessObjectFromFactoryBean(object, beanName);
						} catch (Throwable ex) {
							throw new BeanCreationException(beanName,
									"Post-processing of FactoryBean's singleton object failed", ex);
						} finally {
							//将BeanName从singletonsCurrentlyInCreation集合移除，表示创建完成
							afterSingletonCreation(beanName);
						}
					}
					////将这个生产的实例放入FactoryBean单例缓存
					if (containsSingleton(beanName)) { 
						this.factoryBeanObjectCache.put(beanName, object);
					}
				}
			}
			return object;
		}
	} else {
		//同样是调用FactoryBean的getObject方法生产实例
		Object object = doGetObjectFromFactoryBean(factory, beanName);
		if (shouldPostProcess) { //如果需要调用BeanPostProcessor进行处理
			try {
				//获取当前BeanFactory持有的BeanPostProcessor进行处理
				object = postProcessObjectFromFactoryBean(object, beanName);
			} catch (Throwable ex) {
				throw new BeanCreationException(beanName, "Post-processing of FactoryBean's object failed", ex);
			}
		}
		return object;
	}
}

```
 执行步骤为：  
 1、如果 `FactoryBean` 生产的Bean是单例Bean（即 `getObject` 方法始终返回同一个对象），并且单例缓存中持有这个 `FactoryBean` 对象，那么首先尝试从 `FactoryBean` 单例缓存（负责存储 `FactoryBean` 生产的单例Bean）中获取实例。如果成功获取则直接返回，如果获取不成功，则直接调用 `doGetObjectFromFactoryBean` 方法通过 `FactoryBean` 的 `getObject` 方法生产实例。生产实例后，会再次检查FactoryBean单例缓存有没有这个Bean（这里主要是为了防止 `FactoryBean` 内部的 `getObject` 方法调用了当前 `BeanFactory` 的 `getBean` 方法产生了循环依赖问题后造成的循环依赖问题）。如果检测到已经产生了这个Bean，那么直接返回。否则将根据参数 `shouldPostProcess` 决定是否进行Bean的后处理。  
 后处理会遍历该容器持有的所有 `BeanPostProcessor` 并通过它们依次对Bean进行处理（例如处理 `@Autowired` 注解修饰的变量）

 2、如果不满足上述条件，则直接调用 `doGetObjectFromFactoryBean` 生产Bean然后通过所有的 `BeanPostProcessor` 进行处理。

 []()3、如果缓存中没有对应的实例。那么会检查是否发生了循环依赖： 
```
if (isPrototypeCurrentlyInCreation(beanName)) { //如果发生了循环依赖
	throw new BeanCurrentlyInCreationException(beanName);
}

protected boolean isPrototypeCurrentlyInCreation(String beanName) {
	Object curVal = this.prototypesCurrentlyInCreation.get();
	return (curVal != null && (curVal.equals(beanName) || 
		   (curVal instanceof Set && ((Set<?>) curVal).contains(beanName))));
}

```
 Spring会主动解决 `Singleton` 作用域的循环依赖问题。对于 `Prototype` 作用域的Bean所产生的循环依赖，Spring会直接抛出 `BeanCurrentlyInCreationException` 异常。

 上述方法只是检测非 `Singleton` 作用域的Bean有没有发生循环依赖。那么Spring是如何检查到非 `Singleton` 作用域的Bean发生了循环依赖的呢？  
 我们假设Bean A和B互相依赖。Spring会在构造非 `Singleton` 作用域的Bean的时候，在当前线程构造一个缓存（通过 `ThreadLocal` ），并将正在构造的对象A加入这个缓存。在后续方法中，发现这个对象A依赖于对象B，于是开始了对象B的加载，在加载对象B的过程中，同样将对象B丢入这个缓存，接着对象B发现又依赖于对象A，此时会尝试实例化A，这个时候就发现缓存中已经有A的实例了，也就是 `isPrototypeCurrentlyInCreation` 返回true。

 []()4、首先从父 `BeanFactory` 中查找是否有用户需要的Bean： 
```
BeanFactory parentBeanFactory = getParentBeanFactory();
if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
	String nameToLookup = originalBeanName(name);
	if (parentBeanFactory instanceof AbstractBeanFactory) {
		return ((AbstractBeanFactory) parentBeanFactory).doGetBean(
			nameToLookup, requiredType, args, typeCheckOnly);
	} else if (args != null) {
		return (T) parentBeanFactory.getBean(nameToLookup, args);
	} else if (requiredType != null) {
		return parentBeanFactory.getBean(nameToLookup, requiredType);
	} else {
		return (T) parentBeanFactory.getBean(nameToLookup);
	}
}

```
 如果满足当前 `BeanFactory` 持有父 `BeanFactory` 并且当前 `BeanFactory` 没有这个Bean，那么就会调用它的 `getBean` 方法查找Bean并返回。

 []()5、如果不是仅仅对这个Bean进行类型检查而是要创建这个Bean的实例，那么记录下这个Bean的ID（ `beanName` ）。 
```
if (!typeCheckOnly) {
	markBeanAsCreated(beanName);
}

```
 
```
protected void markBeanAsCreated(String beanName) {
	//检查正在创建的Bean清单中有没有这个Bean
	if (!this.alreadyCreated.contains(beanName)) {
		synchronized (this.mergedBeanDefinitions) {
			//类似于单例模式的双重检查
			if (!this.alreadyCreated.contains(beanName)) { 
				clearMergedBeanDefinition(beanName);
				this.alreadyCreated.add(beanName);
			}
		}
	}
}

```
 上述方法会检查正在创建的Bean清单中有没有这个Bean，如果存在则跳出方法。如果不存在，那么 `clearMergedBeanDefinition` 将这个Bean的 `RootBeanDefinition` 从缓存中移除，以防止Bean在创建过程中 `RootBeanDefinition` 一些元数据发生变化。

 []()6、将 `GenericBeanDefinition` （只负责存储XML数据）转换为 `RootBeanDefinition` ： 
```
protected RootBeanDefinition getMergedLocalBeanDefinition(String beanName) throws BeansException {
	RootBeanDefinition mbd = this.mergedBeanDefinitions.get(beanName);
	if (mbd != null) {
		return mbd;
	}
	return getMergedBeanDefinition(beanName, getBeanDefinition(beanName));
}

```
 首先会从缓存 `mergedBeanDefinitions` 中查找是否已经有处理好的 `RootBeanDefinition` 。如果没有，则进行 `GenericBeanDefinition` 的合并处理：

 
```
protected RootBeanDefinition getMergedBeanDefinition(String beanName, BeanDefinition bd)
		throws BeanDefinitionStoreException {
	return getMergedBeanDefinition(beanName, bd, null);
}

protected RootBeanDefinition getMergedBeanDefinition(
			String beanName, BeanDefinition bd, @Nullable BeanDefinition containingBd)
			throws BeanDefinitionStoreException {
	synchronized (this.mergedBeanDefinitions) {
		RootBeanDefinition mbd = null;
		if (containingBd == null) {
			//再次从缓存中获取RootBeanDefinition
			mbd = this.mergedBeanDefinitions.get(beanName);
		}
		if (mbd == null) {
			if (bd.getParentName() == null) { //如果没有父Bean
				if (bd instanceof RootBeanDefinition) {
					mbd = ((RootBeanDefinition) bd).cloneBeanDefinition();
				} else { //构造一个RootBeanDefinition，并将GenericBeanDefinition内容复制进去
					mbd = new RootBeanDefinition(bd);
				}
			} else {
				BeanDefinition pbd;
				try {
					//获取父Bean的名称
					String parentBeanName = transformedBeanName(bd.getParentName());
					if (!beanName.equals(parentBeanName)) { //如果有父Bean名称和当前Bean名称不同
						//获取父Bean的RootBeanDefinition(这里会递归合并父Bean属性)
						pbd = getMergedBeanDefinition(parentBeanName);
					} else { //如果相同，那么尝试从父BeanFactory查找
						BeanFactory parent = getParentBeanFactory();
						if (parent instanceof ConfigurableBeanFactory) {
							pbd = ((ConfigurableBeanFactory) parent).getMergedBeanDefinition(parentBeanName);
						} else { //如果没有父BeanFactory或者不属于ConfigurableBeanFactory，抛出异常
							throw new NoSuchBeanDefinitionException(parentBeanName,
									"Parent name '" + parentBeanName + "' is equal to bean name '" + beanName +
									"': cannot be resolved without an AbstractBeanFactory parent");
						}
					}
				} catch (NoSuchBeanDefinitionException ex) {
					throw new BeanDefinitionStoreException(bd.getResourceDescription(), beanName,
							"Could not resolve parent bean definition '" + bd.getParentName() + "'", ex);
				}
				//构造父Bean的RootBeanDefinition对象
				mbd = new RootBeanDefinition(pbd);
				//将父Bean属性与当前Bean合并
				mbd.overrideFrom(bd);
			}

			// 如果没有设置scope属性，那么默认设置为单例
			if (!StringUtils.hasLength(mbd.getScope())) {
				mbd.setScope(RootBeanDefinition.SCOPE_SINGLETON);
			}
			if (containingBd != null && !containingBd.isSingleton() && mbd.isSingleton()) {
				mbd.setScope(containingBd.getScope());
			}
			//将RootBeanDefinition放入缓存
			if (containingBd == null && isCacheBeanMetadata()) {
				this.mergedBeanDefinitions.put(beanName, mbd);
			}
		}
		return mbd;
	}
}

```
 如果 `GenericBeanDefinition` 没有指定父Bean，那么只需要将 `GenericBeanDefinition` 中的信息拷贝到一个新的 `RootBeanDefinition` 对象中并返回。如果指定了父Bean，那么会首先递归合并父Bean的 `GenericBeanDefinition` ，然后这个Bean才会和父Bean的 `RootBeanDefinition` 合并为一个 `RootBeanDefinition` 。合并完成后，将 `RootBeanDefinition` 放入 `mergedBeanDefinitions` 缓存。

 []()7、抽象Bean的检测： 
```
protected void checkMergedBeanDefinition(RootBeanDefinition mbd, String beanName, @Nullable Object[] args)
		throws BeanDefinitionStoreException {
	if (mbd.isAbstract()) {
		throw new BeanIsAbstractException(beanName);
	}
}

```
 这里会检测是不是抽象Bean，如果是抽象Bean则抛出异常。

 []()8、初始化依赖的Bean（ `depends-on` 指定的Bean） 
```
String[] dependsOn = mbd.getDependsOn();	//获取这个Bean的依赖Bean
if (dependsOn != null) {
	for (String dep : dependsOn) {	//初始化这个Bean的依赖Bean
		if (isDependent(beanName, dep)) { //如果依赖的Bean也依赖于这个Bean，那么抛出异常
			throw new BeanCreationException(mbd.getResourceDescription(), beanName,
				"Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
		}
		registerDependentBean(dep, beanName); //缓存这个Bean所依赖的所有Bean的名称
		try {
			getBean(dep);//递归获取这个依赖Bean
		} catch (NoSuchBeanDefinitionException ex) {
			throw new BeanCreationException(mbd.getResourceDescription(), beanName,
				"'" + beanName + "' depends on missing bean '" + dep + "'", ex);
		}
	}
}

```
 这段代码通过调用 `RootBeanDefinition` 的 `getDependsOn` 方法获取所有依赖的Bean，并遍历这些Bean来进行初始化操作。

 []()9、对Bean的作用域进行处理 （1）如果是 `Singleton` 作用域

 
```
sharedInstance = getSingleton(beanName, () -> {
	try {
		return createBean(beanName, mbd, args);
	} catch (BeansException ex) {
		destroySingleton(beanName);
		throw ex;
	}
});
bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);

```
 首先调用 `getSingleton` 方法通过传入的 `ObjectFactory` 构造Bean：

 
```
public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
	Assert.notNull(beanName, "Bean name must not be null");
	synchronized (this.singletonObjects) { //加锁，防止有其他线程构造实例
		//检查缓存中Bean有没有被创建，确保这个Bean的仅有一个实例
		Object singletonObject = this.singletonObjects.get(beanName);
		if (singletonObject == null) {
			//检查是否在Bean的destory-method中尝试调用getBean方法
			if (this.singletonsCurrentlyInDestruction) { 
				throw new BeanCreationNotAllowedException(beanName,
						"Singleton bean creation not allowed while singletons of this factory are in destruction " +
						"(Do not request a bean from a BeanFactory in a destroy method implementation!)");
			}
			//将这个beanName添加到一个Set集合中，表示这个单例正在构造
			beforeSingletonCreation(beanName); 
			boolean newSingleton = false;
			boolean recordSuppressedExceptions = (this.suppressedExceptions == null);
			if (recordSuppressedExceptions) {
				this.suppressedExceptions = new LinkedHashSet<>();
			}
			try {
				//尝试构造单例，如果在此期间有其它线程抢先构造了Bean那么抛出IllegalStateException
				singletonObject = singletonFactory.getObject();
				newSingleton = true;
			} catch (IllegalStateException ex) {
				//如果有其它线程构造了这个Bean的对象，那么直接获取这个对象
				singletonObject = this.singletonObjects.get(beanName);
				if (singletonObject == null) {
					throw ex;
				}
			} catch (BeanCreationException ex) {
				if (recordSuppressedExceptions) {
					for (Exception suppressedException : this.suppressedExceptions) {
						ex.addRelatedCause(suppressedException);
					}
				}
				throw ex;
			} finally {
				if (recordSuppressedExceptions) {
					this.suppressedExceptions = null;
				}
				afterSingletonCreation(beanName); //将这个beanName从Set集合中移除，表示构造完成
			}
			if (newSingleton) { //将这个单例Bean添加到缓存中
				addSingleton(beanName, singletonObject);
			}
		}
		return singletonObject;
	}
}

```
 简单分析一下 `getSingleton` 方法的执行步骤：  
 1、首先对单例缓存加锁，然后判断缓存中有没有构造好的单例对象，确保只有一个Bean被构造。如果从缓存中找到了这个Bean的对象，则说明有另外一个线程抢先实例化了这个Bean，那么退出这个方法。  
 2、将Bean的名称加入到一个全局 `Set` 集合中，表示这个单例正处于被构造的阶段。  
 3、调用传入的 `ObjectFactory` 的 `getObject` 方法构造对象， `getObject` 方法会调用 `BeanFactory` 的 `createBean` 方法生成对象。 `createBean` 方法的代码比较复杂，所以我们稍作讨论。  
 4、如果发现此时有其他线程抢先构造了对象，那么则采用这个对象作为返回值。  
 5、将这个实例从 `Set` 集合中移除，表示构造完成。  
 6、将这个构造好的实例添加到缓存 `singletonObjects` 中。

 （2）如果是 `prototype` 作用域

 
```
else if (mbd.isPrototype()) {	//如果不是单例
	Object prototypeInstance = null;
	try {
		beforePrototypeCreation(beanName);
		prototypeInstance = createBean(beanName, mbd, args);
	} finally {
		afterPrototypeCreation(beanName);
	}
	//检查这个Bean是不是FactoryBean，并进行相应处理
	bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
}

```
 首先会调用 `beforePrototypeCreation` 方法将当前线程正在创建的Bean的名称添加到线程私有的集合 `prototypesCurrentlyInCreation` 中，以便于检测到循环依赖。随后，同样会调用 `createBean` 创建实例，然后检查这个Bean实例是不是属于 `FactoryBean` 并进行相应处理，实例创建完成后会将 `prototypesCurrentlyInCreation` 中的变量删除，表示创建完成。

 （3）其他作用域

 
```
else {
	//寻找其定义的作用域Scope接口
	String scopeName = mbd.getScope();
	final Scope scope = this.scopes.get(scopeName);
	if (scope == null) {
		throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
	}
	try {
		Object scopedInstance = scope.get(beanName, () -> {
			//和Prototype作用域的处理类似
			beforePrototypeCreation(beanName);
			try {
				return createBean(beanName, mbd, args);
			} finally {
				afterPrototypeCreation(beanName);
			}
		});
		//检查这个Bean是不是FactoryBean，并进行相应处理
		bean = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
	} catch (IllegalStateException ex) {
		throw new BeanCreationException(beanName,
			"Scope '" + scopeName + "' is not active for the current thread; consider " +
			"defining a scoped proxy for this bean if you intend to refer to it from a singleton",
			ex);
	}
}

```
 对于其它作用域的Bean而言， `BeanFactory` 首先会尝试从 `scopes` 找到对应的 `Scope` 对象（ `Scope` 接口定义了作用域的具体表现），如果没有找到则抛出 `IllegalStateException` 异常。找到 `Scope` 对象后会调用其 `get` 方法传入Bean的名称和 `ObjectFactory` 对象构造其对应的Bean实例，其余过程和 `Prototype` 作用域处理过程类似。

 []()10、检查requiredType 
```
if (requiredType != null && !requiredType.isInstance(bean)) {	//检查实际Bean的类型是否就是requiredType
	try {
		T convertedBean = getTypeConverter().convertIfNecessary(bean, requiredType);
		if (convertedBean == null) {
			throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
		}
		return convertedBean;
	} catch (TypeMismatchException ex) {
		if (logger.isTraceEnabled()) {
			logger.trace("Failed to convert bean '" + name + "' to required type '" +
				ClassUtils.getQualifiedName(requiredType) + "'", ex);
		}
		throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
	}
}

```
 这一步骤比较简单，如果用户传入了 `requiredType` 的话，那么会检查Bean的类型是否符合 `requiredType` ，如果符合，则进行强制转换后返回给用户。如果不符合，则尝试通过 `TypeConverter` 进行类型的转换，如果无法转换那么会发生异常。

 经过上述十个步骤，我们需要Bean就正式创建好了。  
   


 
#### []()创建Bean的实例

 
--------
 回到刚才的第九个步骤，在传入 `ObjectFactory` 参数时，都会调用到一个方法 `createBean` ，该方法会将 `RootBeanDefinition` 转换为我们需要的实例。

 
```
@Override
protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
		throws BeanCreationException {
	if (logger.isTraceEnabled()) {
		logger.trace("Creating instance of bean '" + beanName + "'");
	}
	RootBeanDefinition mbdToUse = mbd;
	//正式加载Bean的Class
	Class<?> resolvedClass = resolveBeanClass(mbd, beanName);
	if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {
		//克隆一个RootBeanDefinition，防止在此期间有其它线程进行动态解析
		mbdToUse = new RootBeanDefinition(mbd);
		mbdToUse.setBeanClass(resolvedClass);
	}

	try {
		//验证覆盖的方法(lookup-method和replace-method指定的方法)
		mbdToUse.prepareMethodOverrides();
	} catch (BeanDefinitionValidationException ex) {
		throw new BeanDefinitionStoreException(mbdToUse.getResourceDescription(),
				beanName, "Validation of method overrides failed", ex);
	}

	try {
		//判断需不需要产生代理对象，也就是AOP。如果仅仅只是生成这个Class的对象，那么无需生成动态代理对象，此时就会返回null
		Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
		if (bean != null) {
			return bean;
		}
	} catch (Throwable ex) {
		throw new BeanCreationException(mbdToUse.getResourceDescription(), beanName,
				"BeanPostProcessor before instantiation of bean failed", ex);
	}

	try {
		//正式构造Bean的对象
		Object beanInstance = doCreateBean(beanName, mbdToUse, args);
		if (logger.isTraceEnabled()) {
			logger.trace("Finished creating instance of bean '" + beanName + "'");
		}
		return beanInstance;
	} catch (BeanCreationException | ImplicitlyAppearedSingletonException ex) {
		throw ex;
	} catch (Throwable ex) {
		throw new BeanCreationException(
			mbdToUse.getResourceDescription(), beanName, "Unexpected exception during bean creation", ex);
	}
}

```
 在构造对象前，会首先对这个Bean的进行类加载操作。随后，如果这个Bean定义了方法替换、方法拦截等属性，会启用AOP进行动态代理生成一个代理Bean然后返回。如果没有定义这些属性，则会调用 `doCreateBean` 生成对象：

 
```
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args)
			throws BeanCreationException {
	BeanWrapper instanceWrapper = null;
	if (mbd.isSingleton()) { //如果是单例，那么就尝试清空缓存
		instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
	}
	if (instanceWrapper == null) {
		//构造BeanWrapper对象
		instanceWrapper = createBeanInstance(beanName, mbd, args);
	}
	final Object bean = instanceWrapper.getWrappedInstance();
	Class<?> beanType = instanceWrapper.getWrappedClass();
	if (beanType != NullBean.class) {
		mbd.resolvedTargetType = beanType;
	}

	synchronized (mbd.postProcessingLock) {
		if (!mbd.postProcessed) {
			try {
				applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
			} catch (Throwable ex) {
				throw new BeanCreationException(mbd.getResourceDescription(), beanName,
						"Post-processing of merged bean definition failed", ex);
			}
			mbd.postProcessed = true;
		}
	}
	//是否需要提早曝光
	boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences && 
			isSingletonCurrentlyInCreation(beanName));
	if (earlySingletonExposure) {
		if (logger.isTraceEnabled()) {
			logger.trace("Eagerly caching bean '" + beanName +
					"' to allow for resolving potential circular references");
		}
		//添加到singletonFactories缓存
		addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
	}

	Object exposedObject = bean;
	try {
		//对bean进行填充，注入属性值，如果依赖于其它bean，则会递归初始依赖bean
		populateBean(beanName, mbd, instanceWrapper); 
		//调用初始化方法，例如init-method等
		exposedObject = initializeBean(beanName, exposedObject, mbd);
	} catch (Throwable ex) {
		if (ex instanceof BeanCreationException && beanName.equals(((BeanCreationException) ex).getBeanName())) {
			throw (BeanCreationException) ex;
		} else {
			throw new BeanCreationException(
				mbd.getResourceDescription(), beanName, "Initialization of bean failed", ex);
		}
	}
	
	if (earlySingletonExposure) {
		Object earlySingletonReference = getSingleton(beanName, false);
		if (earlySingletonReference != null) {	//只有当检测到循环依赖的时候才不会为null
			if (exposedObject == bean) {	//如果exposedObject没有在初始化方法中被改变
				exposedObject = earlySingletonReference;
			} else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
				String[] dependentBeans = getDependentBeans(beanName);
				Set<String> actualDependentBeans = new LinkedHashSet<>(dependentBeans.length);
				for (String dependentBean : dependentBeans) {
					if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
						actualDependentBeans.add(dependentBean);
					}
				}
				if (!actualDependentBeans.isEmpty()) {
					throw new BeanCurrentlyInCreationException(beanName,
							"Bean with name '" + beanName + "' has been injected into other beans [" +
							StringUtils.collectionToCommaDelimitedString(actualDependentBeans) +
							"] in its raw version as part of a circular reference, but has eventually been " +
							"wrapped. This means that said other beans do not use the final version of the " +
							"bean. This is often the result of over-eager type matching - consider using " +
							"'getBeanNamesOfType' with the 'allowEagerInit' flag turned off, for example.");
				}
			}
		}
	}
	try {
		//根据scope注册bean
		registerDisposableBeanIfNecessary(beanName, bean, mbd); 
	} catch (BeanDefinitionValidationException ex) {
		throw new BeanCreationException(
				mbd.getResourceDescription(), beanName, "Invalid destruction signature", ex);
	}
	return exposedObject;
}

```
 上述方法的实现思路如下：  
 1、如果Bean为单例首先要清除 `BeanWrapper` 缓存。  
 2、实例化Bean，并将对象和 `BeanDefinition` 信息合并到一个 `BeanWrapper` 实例。  
 3、获取 `BeanFactory` 所有的 `BeanPostProcessor` 并依次对这个Bean实例进行处理。诸如 `@Autowired` 注解的实现就是在这里进行的。  
 4、对该Bean进行依赖处理。  
 5、将该Bean所有的属性填充到实例中，并初始化实例。  
 这里Bean的初始化操作会调用Bean指定的 `init-method` ，如果Bean还实现了 `Aware` 接口也会在这一步骤进行操作。  
 6、循环依赖的检查。Spring只能够解决 `Singleton` 作用域的Bean，对于其它作用域的Bean一旦检测到循环依赖便会抛出异常。  
 7、 `DisposableBean` 的注册。  
 例如Bean指定了 `destory-method` 方法后这里会进行注册以便在 `BeanFactory` 被销毁后能够被调用。

   
  