---
title: Spring 5.0源码解读：Bean XML配置文件解析类XmlBeanDefinitionReader
date: 2019-04-18 20:45:49
tags: CSDN迁移
---
 版权声明：尊重原创，转载请注明出处 [ https://blog.csdn.net/abc123lzf/article/details/89354623]( https://blog.csdn.net/abc123lzf/article/details/89354623)   
  #### []()引言

 
--------
 `XmlBeanDefinitionReader` 是Spring中用来将Bean的XML配置文件转换为多个 `BeanDefinition` 对象的工具类，一个 `BeanDefinition` 对象对应一个<bean>标签中的信息。这样， `BeanFactory` 或者 `ApplicationContext` 仅需要负责根据 `BeanDefinition` 的内容来生成的bean的实例。

   
 
#### []()测试代码

 
--------
 我们以下列代码为例分析 `XmlBeanDefinitionReader` 解析XML文档的过程：

 
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
 上述代码执行步骤如下：  
 1、通过 `ClassPathResource` 对象将classpath目录下的Bean XML配置文件封装起来。  
 在Spring框架中，对其内部使用到的资源通过一个抽象接口 `Resource` 表示。这些资源可以来自于文件系统、网络、内存（字节数组）等。  
  `Resource` 类图：  
 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190417170202229.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FiYzEyM2x6Zg==,size_16,color_FFFFFF,t_70)  
 2、构造一个BeanFactory对象 `DefaultListableBeanFactory` 。  
 3、构造 `XmlBeanDefinitionReader` ，并传入目标BeanFactory对象，表示将Bean配置文件的内容写入到 `BeanDefinitionRegistry` 中（ `DefaultListableBeanFactory` 实现了 `BeanDefinitionRegistry` ）。  
 4、调用 `XmlBeanDefinitionReader` 的 `loadBeanDefinitions(Resouce)` 方法将配置文件内容写入BeanFactory。  
 5、执行获取Bean的业务逻辑…

   
 
#### []()XmlBeanDefinitionReader实例化过程

 
--------
 UML类图：  
 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190417172030623.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FiYzEyM2x6Zg==,size_16,color_FFFFFF,t_70)  
  `XmlBeanDefinitionReader` 实现了两个接口： `EnvironmentCapable` 和 `BeanDefinitionReader` 接口，实现了 `EnvironmentCapable` 接口表示这个类可以存储一些环境有关的变量（Spring会默认存储系统变量），实现了 `BeanDefinitionReader` 接口表示这是一个 `BeanDefinition` 的读取工具。

 `BeanDefinitionReader` 接口定义了一些基本的API：

 
```
public interface BeanDefinitionReader {
	//获取负责保存BeannDefinition的对象(在上述例子中也就是XmlBeanDefinitionReader)
	BeanDefinitionRegistry getRegistry();
	//获取资源加载器
	@Nullable ResourceLoader getResourceLoader();
	//获取加载Bean的类加载器，可以为null
	@Nullable ClassLoader getBeanClassLoader();
	//获取Bean名称生成器
	BeanNameGenerator getBeanNameGenerator();
	//读取目标资源文件的方法
	int loadBeanDefinitions(Resource resource) throws BeanDefinitionStoreException;
	int loadBeanDefinitions(Resource... resources) throws BeanDefinitionStoreException;
	int loadBeanDefinitions(String location) throws BeanDefinitionStoreException;
	int loadBeanDefinitions(String... locations) throws BeanDefinitionStoreException;
}

```
 了解这些接口后，我们来分析 `XmlBeanDefinitionReader` 的构造方法：  
  `XmlBeanDefinitionReader` 仅提供了一个构造方法，这个构造方法需要传入一个 `BeanDefinitionRegistry` 实例：

 
```
public XmlBeanDefinitionReader(BeanDefinitionRegistry registry) {
	super(registry);
}

```
 这里调用到了父类 `AbstractBeanDefinitionReader` 构造方法：

 
```
protected AbstractBeanDefinitionReader(BeanDefinitionRegistry registry) {
	Assert.notNull(registry, "BeanDefinitionRegistry must not be null");
	this.registry = registry;
	//如果这个registry同时也能够充当ResourceLoader的角色，那么赋给成员变量resourceLoader
	if (this.registry instanceof ResourceLoader) {
		this.resourceLoader = (ResourceLoader) this.registry;
	} else { //否则采用PathMatchingResourcePatternResolver作为ResourceLoader
		this.resourceLoader = new PathMatchingResourcePatternResolver();
	}

	// 如果这个registry能够充当EnvironmentCapable的角色，那么赋给成员变量environment
	if (this.registry instanceof EnvironmentCapable) {
		this.environment = ((EnvironmentCapable) this.registry).getEnvironment();
	} else { //否则采用StandardEnvironment作为EnvironmentCapable
		this.environment = new StandardEnvironment();
	}
}

```
 需要注意的是， `ApplicationContext` 接口继承了 `ResourceLoader` 这个接口，而对于非 `ApplicationContext` 的 `BeanFactory` 实现类（例如 `DefaultListableBeanFactory` ）则没有实现这个接口。同样对于 `EnvironmentCapable` 来说， `DefaultListableBeanFactory` 也没有实现这个接口，但是 `ApplicationContext` 接口继承了 `EnvironmentCapable` 接口。

 所以，如果传入的registry是 `DefaultListableBeanFactory` 实例，那么this.registry instanceof ResourceLoader会返回false，this.registry instanceof EnvironmentCapable同样也是返回false。

 `EnvironmentCapable` 接口的作用刚才也提到过了，这里说一下它的实现类 `StandardEnvironment` ， `StandardEnvironment` 默认存储了系统变量（内部通过 `System` 类的静态方法 `getenv()` 和 `getProperties()` 获取），有兴趣的可以自己去看看源码，比较简单，这里就不罗列了。

 这里提一下 `ResourceLoader` 这个接口。从接口名字上可以看出，这个接口应该是定义了载入资源的方法，事实上也确实如此：

 
```
public interface ResourceLoader {
	String CLASSPATH_URL_PREFIX = ResourceUtils.CLASSPATH_URL_PREFIX;
	Resource getResource(String location);
	@Nullable ClassLoader getClassLoader();
}

```
 `ResourceLoader` 接口的实现类 `PathMatchingResourcePatternResolver` 采用了装饰者设计模式，其内部是通过 `DefaultResourceLoader` 实现的，那么 `DefaultResourceLoader` 是怎样实现上面这两个方法的呢？

 
```
@Override
public Resource getResource(String location) {
	Assert.notNull(location, "Location must not be null");
	//首先根据持有的ProtocolResolver解析location
	for (ProtocolResolver protocolResolver : this.protocolResolvers) {
		Resource resource = protocolResolver.resolve(location, this);
		if (resource != null) {
			return resource;
		}
	}
	//如果location以"/"为开头，那么则认定location为文件系统的绝对路径，所以从文件系统中查找
	if (location.startsWith("/")) {
		return getResourceByPath(location);
	} else if (location.startsWith(CLASSPATH_URL_PREFIX)) { //如果以classpath:开头，那么从classpath中查找
		return new ClassPathResource(location.substring(CLASSPATH_URL_PREFIX.length()), getClassLoader());
	} else {
		try {
			// 尝试通过URL查找
			URL url = new URL(location);
			return (ResourceUtils.isFileURL(url) ? new FileUrlResource(url) : new UrlResource(url));
		}
		catch (MalformedURLException ex) { //如果URL获取失败，那么尝试从文件系统中查找
			return getResourceByPath(location);
		}
	}
}

```
 `ProtocolResolver`  接口定义了根据路径字符串获取资源的方法，若这个 `ProtocolResolver`  无法获取到资源，那么应当返回null。用户可以调用 `addProtocolResolver(ProtocolResolver resolver)` 方法来添加自定义的 `ProtocolResolver` 。  
   


 
#### []()Bean XML文档的载入

 
--------
 构造完 `XmlBeanDefinitionReader` 后，接下来的工作就是调用它的 `loadBeanDefinitions(Resource resource)` 方法进行文档的解析了。

 
```
@Override
public int loadBeanDefinitions(Resource resource) throws BeanDefinitionStoreException {
	return loadBeanDefinitions(new EncodedResource(resource));
}

public int loadBeanDefinitions(EncodedResource encodedResource) throws BeanDefinitionStoreException {
	Assert.notNull(encodedResource, "EncodedResource must not be null");
	if (logger.isTraceEnabled())
		logger.trace("Loading XML bean definitions from " + encodedResource);
	//获取当前线程正在加载的资源
	Set<EncodedResource> currentResources = this.resourcesCurrentlyBeingLoaded.get();
	//如果没有加载过的资源，那么初始化一个HashSet用来存放当前线程正在加载的资源
	if (currentResources == null) { 
		currentResources = new HashSet<>(4);
		this.resourcesCurrentlyBeingLoaded.set(currentResources);
	}
	//将这个资源添加到Set集合中，若添加失败(Set集合存在这个Resource)，则说明发生了配置文件的循环依赖，抛出异常
	if (!currentResources.add(encodedResource)) { 
		throw new BeanDefinitionStoreException("Detected cyclic loading of " + 
							encodedResource + " - check your import definitions!");
	}
	try { //将资源转换为InputSource
		InputStream inputStream = encodedResource.getResource().getInputStream();
		try {
			InputSource inputSource = new InputSource(inputStream);
			if (encodedResource.getEncoding() != null) {
				inputSource.setEncoding(encodedResource.getEncoding());
			}
			//开始进行正式的解析工作
			return doLoadBeanDefinitions(inputSource, encodedResource.getResource());
		} finally {
			inputStream.close();
		}
	} catch (IOException ex) {
		throw new BeanDefinitionStoreException(
					"IOException parsing XML document from " + encodedResource.getResource(), ex);
	} finally {
		//配置文件解析完成后，从Set集合中移除
		currentResources.remove(encodedResource);
		if (currentResources.isEmpty()) //如果当前线程没有正在加载的资源了，则移除这个Set集合
			this.resourcesCurrentlyBeingLoaded.remove();
	}
}

```
 `loadBeanDefinitions(Resource resource)` 方法执行流程如下：  
 1、将 `Resource` 转换为 `EncodedResource` ，表示这是一个需要进行解码的资源（因为配置文件属于文本文件，需要对字符进行解码来防止乱码）。  
 2、根据一个 `NamedThreadLocal` （ `ThreadLocal` 的子类，仅添加了一个name成员变量，用来表示名称）对象来获取当前线程**正在加载**的配置文件（存放在一个 `Set<EncodedResource>` 集合中）。如果没有获取到这个集合，则说明当前线程没有任何配置文件处于加载过程，此时会实例化一个HashSet并通过 `NamedThreadLocal` 绑定到当前线程，并将当前传入的配置文件 `EncodedResource` 添加到Set集合。如果获取到了Set集合并且发现这个集合已经存在这个 `EncodedResource` 对象了（ `EncodedResource` 重写了equals方法，通过资源文件路径来判定是否相等），则说明配置文件发生了**循环依赖**，抛出 `BeanDefinitionStoreException` 异常。

 关于 `ThreadLocal` 的原理，可以参考我的博客：[https://blog.csdn.net/abc123lzf/article/details/81978210。](https://blog.csdn.net/abc123lzf/article/details/81978210%E3%80%82)

 那么什么时候会发生配置文件的循环依赖？例如这里有两个Bean的XML配置文件：  
 context1.xml

 
```
<beans>
	<import resource="context2.xml" />
</beans>

```
 context2.xml

 
```
<beans>
	<import resource="context1.xml" />
</beans>

```
 这两个配置文件互相依赖对方，在加载过程中就会抛出异常。

 3、将 `EncodedResource` 转换为 `InputSource` 对象（ `InputSource` 不是Spring自带的，属于包 `org.xml.sax` ），然后，调用 `doLoadBeanDefinitions` 进行正式的配置文件解析工作

 4、解析完成后，将 `EncodedResource` 从Set集合中删除，表示这个配置文件已经解析完成。

 现在我们来看 `doLoadBeanDefinitions` 做了什么工作：

 
```
protected int doLoadBeanDefinitions(InputSource inputSource, Resource resource)
			throws BeanDefinitionStoreException {
	try {
		Document doc = doLoadDocument(inputSource, resource);
		int count = registerBeanDefinitions(doc, resource);
		if (logger.isDebugEnabled()) {
			logger.debug("Loaded " + count + " bean definitions from " + resource);
		}
		return count;
	} catch (Exception ex) {
		//省略异常处理
	}
}

```
 `doLoadBeanDefinitions` 方法有两个步骤：  
 1、调用 `doLoadDocument` 方法将配置文件内容转换为 `org.w3c.dom.Document` 对象，如果配置文件有任何XML语法错误，那么会在这里抛出异常。  
 2、调用 `registerBeanDefinitions` 方法，解析 `Document` 对象中的节点内容，将其转换为 `BeanDefinition` 对象然后注册到 `BeanDefinitionRegistry` 上（也就是测试代码中的 `DefaultListableBeanFactory` ），返回注册的 `BeanDefinition` 对象数量。

 `doLoadDocument` 方法更多的是涉及到XML语法的解析工作了，所以这里就不介绍其中的原理了。  
 来看 `registerBeanDefinitions` 方法：

 
```
public int registerBeanDefinitions(Document doc, Resource resource) throws BeanDefinitionStoreException {
	//实例化DefaultBeanDefinitionDocumentReader
	BeanDefinitionDocumentReader documentReader = createBeanDefinitionDocumentReader();
	//获取BeanDefinitionRegistry持有的BeanDefinition数量
	int countBefore = getRegistry().getBeanDefinitionCount();
	//解析配置，并将新解析到的BeanDefinition注册到BeanDefinitionRegistry
	documentReader.registerBeanDefinitions(doc, createReaderContext(resource));
	//返回本次解析的BeanDefinition数量
	return getRegistry().getBeanDefinitionCount() - countBefore;
}

```
 我们来解读一下上述方法的执行细节：  
 1、首先调用 `createBeanDefinitionDocumentReader` 方法构造一个 `BeanDefinitionDocumentReader` ：

 
```
protected BeanDefinitionDocumentReader createBeanDefinitionDocumentReader() {
	return BeanUtils.instantiateClass(this.documentReaderClass);
}

```
 这里直接通过反射构造了 `DefaultBeanDefinitionDocumentReader` 对象。  
  `BeanDefinitionDocumentReader` 接口定义了以下方法：

 
```
void registerBeanDefinitions(Document doc, XmlReaderContext readerContext) 
		throws BeanDefinitionStoreException;

```
 这个方法的职责是将Document节点的内容读入到 `XmlReaderContext` 。

 2、获取 `BeanDefinitionRegistry` （即 `DefaultListableBeanFactory` ）持有的 `BeanDefinition` 数量，这里涉及到BeanFactory源码部分，本文就不说明了。

 3、调用 `createReaderContext` 方法创建 `XmlReaderContext` :

 
```
public XmlReaderContext createReaderContext(Resource resource) {
	return new XmlReaderContext(resource, this.problemReporter, this.eventListener,
			this.sourceExtractor, this, getNamespaceHandlerResolver());
}

```
 从 `XmlReaderContext` 这个类的名字上来看，应当是在文档解析过程中提供全局信息的一个类。构造 `XmlReaderContext` 对象一共需要传入6个变量，分别是：  
  `Resource` 实例：代表Bean XML配置文件的资源对象  
  `ProblemReporter` 实例：用来记录错误信息，这里传入的是 `FailFastProblemReporter` ，它通过日志的方式记录错误信息。  
  `ReaderEventListener` 实例：提供一个读入事件监听服务，但是这里传入的是 `EmptyReaderEventListener` ，查看源码发现方法实现均为空。  
  `XmlBeanDefinitionReader` 实例：也就是this  
  `NamespaceHandlerResolver` 实例：根据XML文档的Namespace来提供不同的 `NamespaceHandler` ，用来应对XML文档中的自定义标签，例如 `<tx:annotation-driven>` 。

 创建完 `XmlReaderContext` 后，马上就会执行 `DefaultBeanDefinitionDocumentReader` 的 `registerBeanDefinitions` 方法进行正式的解析工作。

 4、解析完成后，返回本次读入的 `BeanDefinition` 数量。

   
 
#### []()Bean XML文档的解析

 
--------
 下面我们从刚才的第三步继续深究：

 
```
@Override
public void registerBeanDefinitions(Document doc, XmlReaderContext readerContext) {
	this.readerContext = readerContext; //首先绑定ReaderContext
	doRegisterBeanDefinitions(doc.getDocumentElement()); //提取出ROOT标签，即<beans>
}

```
 这里会获取XML配置根标签 `<beans>` ，然后调用 `doRegisterBeanDefinitions` 方法：

 
```
protected void doRegisterBeanDefinitions(Element root) {
	BeanDefinitionParserDelegate parent = this.delegate;
	this.delegate = createDelegate(getReaderContext(), root, parent);
	if (this.delegate.isDefaultNamespace(root)) { //如果根标签是默认标签
		//处理<beans>标签的profile属性
		String profileSpec = root.getAttribute(PROFILE_ATTRIBUTE);
		if (StringUtils.hasText(profileSpec)) {
			String[] specifiedProfiles = StringUtils.tokenizeToStringArray(
					profileSpec, BeanDefinitionParserDelegate.MULTI_VALUE_ATTRIBUTE_DELIMITERS);
			if (!getReaderContext().getEnvironment().acceptsProfiles(specifiedProfiles)) {
				if (logger.isDebugEnabled()) {
					logger.debug("Skipped XML bean definition file due to specified profiles [" + profileSpec +
							"] not matching: " + getReaderContext().getResource());
				}
				return;
			}
		}
	}
	preProcessXml(root); //解析前的处理，默认实现为空，提供给子类重写
	parseBeanDefinitions(root, this.delegate); //解析<beans>中的元素
	postProcessXml(root); //解析后的处理，默认实现为空，提供给子类重写
	this.delegate = parent;
}

```
 该方法首先会检查当前环境是否和配置文件相符，即获取 `<beans>` 标签的 `profile` 属性，然后通过调用 `Environment` 的 `acceptsProfiles` 方法进行判定。若不相符，则放弃解析当前配置文件。关于 `<beans>` 标签的 `profile` 属性，可以参考[这篇博客](https://www.cnblogs.com/adolfmc/p/4377939.html)。

 
```
protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {
	if (delegate.isDefaultNamespace(root)) { //如果根标签是默认标签
		NodeList nl = root.getChildNodes(); //获取子标签集合
		for (int i = 0; i < nl.getLength(); i++) { //遍历子标签
			Node node = nl.item(i);  //获取子标签
			if (node instanceof Element) {
				Element ele = (Element) node;
				if (delegate.isDefaultNamespace(ele)) { //解析默认标签，例如<bean>
					parseDefaultElement(ele, delegate); //解析默认标签
				} else {
					delegate.parseCustomElement(ele); //解析自定义标签，例如<tx:annotation-driven>
				}
			}
		}
	} else { //如果是自定义标签
		delegate.parseCustomElement(root);
	}
}

```
 上述代码对不同类型标签有着不同的解析方式。我们先来看根标签为默认标签、子标签也为默认标签的解析方式：

 []()1、子标签为默认标签的解析 
```
private void parseDefaultElement(Element ele, BeanDefinitionParserDelegate delegate) {
	if (delegate.nodeNameEquals(ele, IMPORT_ELEMENT)) { //解析<import>标签
		importBeanDefinitionResource(ele);
	} else if (delegate.nodeNameEquals(ele, ALIAS_ELEMENT)) { //解析<alias>标签
		processAliasRegistration(ele);
	} else if (delegate.nodeNameEquals(ele, BEAN_ELEMENT)) { //解析<bean>标签
		processBeanDefinition(ele, delegate);
	} else if (delegate.nodeNameEquals(ele, NESTED_BEANS_ELEMENT)) { //解析<beans>标签
		doRegisterBeanDefinitions(ele);
	}
}

```
 **1.1 <bean>标签解析**

 
```
protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) {
	//解析默认标签
	BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
	if (bdHolder != null) {
		//解析默认标签中的自定义标签
		bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);
		try {
			// Register the final decorated instance.
			BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, getReaderContext().getRegistry());
		}
			catch (BeanDefinitionStoreException ex) {
				getReaderContext().error("Failed to register bean definition with name '" +
						bdHolder.getBeanName() + "'", ele, ex);
			}
			// Send registration event.
			getReaderContext().fireComponentRegistered(new BeanComponentDefinition(bdHolder));
		}
	}

```
 `BeanDefinitionParserDelegate` 是XML配置文件中标签解析的核心类， `processBeanDefinition` 调用了它的 `parseBeanDefinitionElement` 方法并传入 `Element` 元素，目的是将这个元素转换为 `BeanDefinitionHolder` 对象， `BeanDefinitionHolder` 并非 `BeanDefinition` 接口的实现类，它的作用是作为一个元素解析过程中的临时信息承载体，它的成员变量包含了Bean的ID、name和 `BeanDefinition` 对象：

 
```
@Nullable
public BeanDefinitionHolder parseBeanDefinitionElement(Element ele) {
	return parseBeanDefinitionElement(ele, null);
}

@Nullable
public BeanDefinitionHolder parseBeanDefinitionElement(Element ele, @Nullable BeanDefinition containingBean) {
	String id = ele.getAttribute(ID_ATTRIBUTE);			//获取bean的id属性
	String nameAttr = ele.getAttribute(NAME_ATTRIBUTE);	//获取bean的name属性
	//建立一个List集合，用于存放bean的名称
	List<String> aliases = new ArrayList<>();
	if (StringUtils.hasLength(nameAttr)) { //
		//将name属性内容根据分隔符','或';'将其分成多个字符串，并添加到name集合中
		String[] nameArr = StringUtils.tokenizeToStringArray(nameAttr, MULTI_VALUE_ATTRIBUTE_DELIMITERS);
		aliases.addAll(Arrays.asList(nameArr));
	}
	String beanName = id;
	//如果id为空且name不为空
	if (!StringUtils.hasText(beanName) && !aliases.isEmpty()) {
		beanName = aliases.remove(0); //将第一个name作为beanName
		if (logger.isTraceEnabled()) {
			logger.trace("No XML 'id' specified - using '" + beanName +
					"' as bean name and " + aliases + " as aliases");
		}
	}

	if (containingBean == null) { 
		//检查这个beanName是否在配置中是独一无二的，如果不是就会发生错误
		checkNameUniqueness(beanName, aliases, ele);
	}
	//创建Bean信息的承载实例AbstractBeanDefinition(BeanDefinition的抽象子类)
	AbstractBeanDefinition beanDefinition = parseBeanDefinitionElement(ele, beanName, containingBean);
	if (beanDefinition != null) {
		if (!StringUtils.hasText(beanName)) { //如果beanName为空
			try { 
				if (containingBean != null) { //生成一个内部使用的beanName
					beanName = BeanDefinitionReaderUtils.generateBeanName( 
							beanDefinition, this.readerContext.getRegistry(), true);
				} else {
					beanName = this.readerContext.generateBeanName(beanDefinition);
					String beanClassName = beanDefinition.getBeanClassName();
					if (beanClassName != null && beanName.startsWith(beanClassName) && 
							beanName.length() > beanClassName.length() &&
							!this.readerContext.getRegistry().isBeanNameInUse(beanClassName)) {
						aliases.add(beanClassName);
					}
				}
				if (logger.isTraceEnabled()) {
					logger.trace("Neither XML 'id' nor 'name' specified - " +
							"using generated bean name [" + beanName + "]");
				}
			} catch (Exception ex) {
				error(ex.getMessage(), ele);
				return null;
			}
		}
		String[] aliasesArray = StringUtils.toStringArray(aliases);
		//将解析后的信息封装到BeanDefinitionHolder
		return new BeanDefinitionHolder(beanDefinition, beanName, aliasesArray);
	}
	return null;
}

```
 现在来分析一下 `parseBeanDefinitionElement` 方法的执行流程：  
 1、提取Bean的id、name属性，如果没有设置id，就取name作为它的id。  
 2、解析文档内容并封装为一个 `AbstractBeanDefinition` 对象  
 3、如果id和name属性都为空，并且containingBean也为null，那么就调用 `BeanDefinitionReaderUtils` 工具类的静态方法 `generateBeanName` 生成一个内部使用的id，这个id一般为Bean的全限定类名，如果这个bean是一个内部Bean，那么则名称为父Bean的名称+"$child"+"#"+这个 `BeanDefinition` 的内存地址（通过 `System` 类的 `identityHashCode` 获得）。  
 4、最后将Bean的解析结果封装为 `BeanDefinitionHolder` 返回。

 下面是第2步 `AbstractBeanDefinition` 的生成过程。

 
```
/* File: org.springframework.beans.factory.xml.BeanDefinitionParserDelegate */
@Nullable
public AbstractBeanDefinition parseBeanDefinitionElement(Element ele, String beanName,
		@Nullable BeanDefinition containingBean) {
	this.parseState.push(new BeanEntry(beanName));

	String className = null;
	if (ele.hasAttribute(CLASS_ATTRIBUTE)) { //解析class属性
		className = ele.getAttribute(CLASS_ATTRIBUTE).trim();
	}
	String parent = null;
	if (ele.hasAttribute(PARENT_ATTRIBUTE)) { //解析parent属性
		parent = ele.getAttribute(PARENT_ATTRIBUTE);
	}
	try {
		//根据class名称和父Bean构造BeanDefinition
		AbstractBeanDefinition bd = createBeanDefinition(className, parent);
		//解析bean标签下的属性(非子标签)
		parseBeanDefinitionAttributes(ele, beanName, containingBean, bd);
		//解析<description>标签
		bd.setDescription(DomUtils.getChildElementValueByTagName(ele, DESCRIPTION_ELEMENT));
		//解析<meta>标签
		parseMetaElements(ele, bd);
		//解析<lookup-method>标签
		parseLookupOverrideSubElements(ele, bd.getMethodOverrides());	
		//解析<replaced-method>标签
		parseReplacedMethodSubElements(ele, bd.getMethodOverrides());
		//解析<constructor-arg>标签
		parseConstructorArgElements(ele, bd);
		//解析<property>标签
		parsePropertyElements(ele, bd);
		//解析<qualifier>标签
		parseQualifierElements(ele, bd);
		//将XML配置文件的Resource对象赋给BeanDefinition
		bd.setResource(this.readerContext.getResource());
		bd.setSource(extractSource(ele));

		return bd;
	} catch (ClassNotFoundException ex) {
		error("Bean class [" + className + "] not found", ele, ex);
	} catch (NoClassDefFoundError err) {
		error("Class that bean class [" + className + "] depends on not found", ele, err);
	} catch (Throwable ex) {
		error("Unexpected failure during bean definition parsing", ele, ex);
	} finally {
		this.parseState.pop();
	}
	return null;
}

```
 上述方法主要是用来处理 `<bean>` 标签的属性及其包含的子标签。  
 我们先来简单回顾一下 `<bean>` 有哪些属性和子标签：  
 **1、 `<bean>` 标签的属性**  
  `parent` ：用来指定父Bean，这里应当写入父Bean的名称，可以为抽象Bean  
  `scope` ：Bean的作用域，Spring内置的有singleton（单例）、prototype（多例）、request（请求范围）、session（会话）、global-session（全局）  
  `abstract` ：是否为抽象Bean，抽象Bean一般是提供一个Bean的模板，不包含class信息。其值只可为true或者false。  
  `lazy-init` ：是否采用延迟初始化策略，对于非 `ApplicationContext` 的 `BeanFactory` 实现类来说，是默认开启延迟初始化策略的，只有在需要获取Bean的实例时候才会加载这个类，而 `ApplicationContext` 是默认关闭延迟初始化的，即在容器加载过程中完成类的加载操作，后续仅需要进行实例化即可。其值只可为true、false或者default。  
  `autowire` ：即自动装配机制，当别的Bean需要依赖这个Bean的时候，是通过什么样的策略来识别这个Bean并注入到别的Bean。其值只可为 `byName` 、 `byType` 、 `constructor` 、 `default` 和 `no` 。具体可参照这篇博客：[https://www.cnblogs.com/ViviChan/p/4981539.html](https://www.cnblogs.com/ViviChan/p/4981539.html)  
  `depends-on` ：表示依赖于指定的Bean，在当前Bean初始化的时候，会优先初始化 `depends-on` 中指定的Bean  
  `autowire-candidate` ：Bean的候选属性，同样可以参照上面的博客。其值只可为true、false或者default。  
  `primary` ：表示是否是优先的Bean，如果容器中有多个同一类型的Bean，如果不指定一个 `primary` 为true的Bean，那么在注入时就会因为存在多个类型匹配的Bean而抛出异常。当指定了一个 `primary` 为true的Bean后，就不会因此而抛出异常。其值只可为true、false  
  `init-method` ：Bean的默认初始化方法  
  `destory-method` ：Bean的默认销毁方法  
  `factory-bean` ：指向实例工厂方法的bean  
  `factory-method` ：指向实例工厂方法的名字

 **2、子标签**  
  `<description>` ：Bean的描述信息  
  `<meta>` ：用来存储一些键值型的参数，通过调用 `BeanDefinition` 的 `getAttribute` 方法获得  
  `<lookup-method>` ：用来通过方法来实现依赖注入，可以参考博客：[https://www.cnblogs.com/ViviChan/p/4981619.html](https://www.cnblogs.com/ViviChan/p/4981619.html)  
  `<replaced-method>` ：用来替换该Bean存在的方法，其属性有 `name` 和 `replacer` 。可以参考[这篇博客](https://blog.csdn.net/qq_22912803/article/details/52503914)  
  `<property>` ：通过成员变量注入  
  `<constructor-arg>` ：通过构造方法注入  
  `<qualifier>` ：指定Bean的一个特殊名称，防止在注入时其它Bean时因为有多个匹配的Bean而发生异常。

 了解完了后，我们来看看 `AbstractBeanDefinition` 是如何构造的：

 
```
protected AbstractBeanDefinition createBeanDefinition(@Nullable String className, @Nullable String parentName)
		throws ClassNotFoundException {
	return BeanDefinitionReaderUtils.createBeanDefinition(parentName, className, 
				this.readerContext.getBeanClassLoader());
}

public static AbstractBeanDefinition createBeanDefinition(@Nullable String parentName, 
		@Nullable String className, @Nullable ClassLoader classLoader) throws ClassNotFoundException {
	GenericBeanDefinition bd = new GenericBeanDefinition();
	bd.setParentName(parentName); //设置父Bean的名称
	if (className != null) {
		if (classLoader != null) { //如果设置了类加载器，那么直接初始化该类
			bd.setBeanClass(ClassUtils.forName(className, classLoader));
		} else { //否则，只是记录这个类的类名
			bd.setBeanClassName(className);
		}
	}
	return bd;
}

```
 这里的类加载器默认从 `ReaderContext` 获取，而 `ReaderContext` 内部则默认通过 `XmlBeanDefinitionReader` 获得。 `XmlBeanDefinitionReader` 默认的Bean类加载器为null，表示进行延迟初始化。所以在这个地方构造了 `GenericBeanDefinition` 后，仅仅只是记录了类名而不进行类的初始化。除非调用了 `XmlBeanDefinitionReader` 的 `setBeanClassLoader` 方法强制指定了类加载器。

 `GenericBeanDefinition` 继承了 `AbstractBeanDefinition` ，仅添加了一个成员变量 `parentName` ，表示父Bean的名称。  
 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190418153412707.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FiYzEyM2x6Zg==,size_16,color_FFFFFF,t_70)

 构造完 `GenericBeanDefinition` 后，就会开始进行 `<bean>` 元素的解析工作，即 `parseBeanDefinitionAttributes` 方法，这段代码限于篇幅就不贴了，其实很简单，就是调用了 `AbstractBeanDefinition` 一些set方法设置每个属性的值，感兴趣的朋友可以自己去看一看。

 随后的操作步骤：  
 1、根据 `<description>` 内容将描述信息设置到 `GenericBeanDefinition`   
 2、根据 `<meta>` 内容设置参数信息到 `GenericBeanDefinition` ：

 
```
public void parseMetaElements(Element ele, BeanMetadataAttributeAccessor attributeAccessor) {
	NodeList nl = ele.getChildNodes();
	for (int i = 0; i < nl.getLength(); i++) {
		Node node = nl.item(i);
		if (isCandidateElement(node) && nodeNameEquals(node, META_ELEMENT)) {
			Element metaElement = (Element) node;
			String key = metaElement.getAttribute(KEY_ATTRIBUTE);
			String value = metaElement.getAttribute(VALUE_ATTRIBUTE);
			BeanMetadataAttribute attribute = new BeanMetadataAttribute(key, value);
			attribute.setSource(extractSource(metaElement));
			attributeAccessor.addMetadataAttribute(attribute);
		}
	}
}

```
 `parseMetaElements` 会将每个 `<meta>` 元素中的key和value的值封装为一个 `BeanMetadataAttribute` 对象然后添加到 `GenericBeanDefinition` 中。

 3、根据 `<lookup-method>` 标签将其中的name、bean属性封装为一个 `LookupOverride` 对象并添加到 `GenericBeanDefinition` 持有的 `MethodOverrides` 对象中。逻辑和上述代码类似。  
 4、根据 `<replaced-method>` 标签将其中的内容封装为 `ReplaceOverride` 对象并添加到 `GenericBeanDefinition` 持有的 `MethodOverrides` 对象中。  
 5、将每个 `<constructor-arg>` 内容封装为 `ConstructorArgumentValues.ValueHolder` 对象并添加到 `GenericBeanDefinition` 持有的 `ConstructorArgumentValues` 中。  
 6、将每个 `<property>` 标签封装为一个 `PropertyValue` 对象然后添加到 `GenericBeanDefinition` 。  
 7、将 `<qualifier>` 标签封装为一个 `BeanMetadataAttribute` 对象然后添加到一个新构造的 `AutowireCandidateQualifier` 中，然后将 `AutowireCandidateQualifier` 添加到 `GenericBeanDefinition` 。

 回过头来继续分析 `processBeanDefinition` 方法，当 `parseBeanDefinitionElement` 返回一个 `BeanDefinitionHolder` 实例后，会继续调用 `BeanDefinitionParserDelegate` 的 `decorateBeanDefinitionIfRequired` 方法，这个方法是用来解析自定义标签的：

 
```
public BeanDefinitionHolder decorateBeanDefinitionIfRequired(Element ele, BeanDefinitionHolder definitionHolder) {
	return decorateBeanDefinitionIfRequired(ele, definitionHolder, null);
}

public BeanDefinitionHolder decorateBeanDefinitionIfRequired(Element ele, 
		BeanDefinitionHolder definitionHolder, @Nullable BeanDefinition containingBd) {
	BeanDefinitionHolder finalDefinition = definitionHolder;
	//获取这个<bean>标签下所有的自定义属性
	NamedNodeMap attributes = ele.getAttributes();
	for (int i = 0; i < attributes.getLength(); i++) {
		Node node = attributes.item(i);
		finalDefinition = decorateIfRequired(node, finalDefinition, containingBd);
	}
	//获取这个<bean>标签下所有的自定义标签
	NodeList children = ele.getChildNodes();
	for (int i = 0; i < children.getLength(); i++) {
		Node node = children.item(i);
		if (node.getNodeType() == Node.ELEMENT_NODE) {
			finalDefinition = decorateIfRequired(node, finalDefinition, containingBd);
		}
	}
	return finalDefinition;
}

```
 上述方法的主要功能就是获取这个 `<bean>` 标签下所有的自定义标签及其属性，获取到 `Node` 对象后，就会调用 `decorateIfRequired` 方法进行进一步的解析工作：

 
```
public BeanDefinitionHolder decorateIfRequired(Node node, BeanDefinitionHolder originalDef, 
		@Nullable BeanDefinition containingBd) {
	String namespaceUri = getNamespaceURI(node); //获取这个Node的Namespace
	//如果Namespace不为null且不是默认的namespace
	if (namespaceUri != null && !isDefaultNamespace(namespaceUri)) {
		//获取对应的NamespaceHandler进行处理
		NamespaceHandler handler = this.readerContext.getNamespaceHandlerResolver().resolve(namespaceUri);
		if (handler != null) { //如果找到了对应的NamespaceHandler，那么就调用它的decorate方法解析
			BeanDefinitionHolder decorated = handler.decorate(node, originalDef, 
				new ParserContext(this.readerContext, this, containingBd));
			if (decorated != null) { //如果解析成功，就返回
				return decorated;
			}
		//如果是Spring的Namespace却没有找到对应的NamespaceHandler，那么抛出异常
		} else if (namespaceUri.startsWith("http://www.springframework.org/")) { 
			error("Unable to locate Spring NamespaceHandler for XML schema namespace [" + namespaceUri + "]", node);
		} else { //如果没有找到NamespaceHandler
			if (logger.isDebugEnabled()) {
				logger.debug("No Spring NamespaceHandler found for XML schema namespace [" + namespaceUri + "]");
			}
		}
	}
	//如果执行到这里，则表示并没有对这个Node进行任何的解析工作
	return originalDef;
}

```
 `NamespaceHandler` 接口定义了解析自定义标签的方法，用户如果想实现自己的自定义标签并成功地载入容器，那么就需要实现这个接口。

 现在，这个 `<bean>` 标签中所有的元素和标签都已经解析完成，那么剩下的工作就是将这个 `BeanDefinitionHolder` 中的 `BeanDefinition` 注册到 `BeanDefinitionRegistry` 上（也就是测试代码中的 `DefaultListableBeanFactory` ），这里是通过工具类 `BeanDefinitionReaderUtils` 的静态方法 `registerBeanDefinition` 完成的：

 
```
public static void registerBeanDefinition(BaeanDefinitionHolder definitionHolder,
		 BeanDefinitionRegistry registry) throws BeanDefinitionStoreException {
	String beanName = definitionHolder.getBeanName(); //获取bean的名称
	//注册到BeanDefinitionRegistry
	registry.registerBeanDefinition(beanName, definitionHolder.getBeanDefinition());
	//注册id和name的对应关系
	String[] aliases = definitionHolder.getAliases();
	if (aliases != null) {
		for (String alias : aliases) {
			registry.registerAlias(beanName, alias);
		}
	}
}

```
 **1.2 <import>标签解析**  
 在当前配置文件中， `<import>` 标签用来引入别的配置文件，其属性resource制定了其路径。

 
```
protected void importBeanDefinitionResource(Element ele) {
	//获取resource属性
	String location = ele.getAttribute(RESOURCE_ATTRIBUTE);
	if (!StringUtils.hasText(location)) {
		getReaderContext().error("Resource location must not be empty", ele);
		return;
	}
	//解析路径中的系统属性
	location = getReaderContext().getEnvironment().resolveRequiredPlaceholders(location);
	Set<Resource> actualResources = new LinkedHashSet<>(4);
	boolean absoluteLocation = false;
	//判断路径是绝对路径还是相对路径
	try {
		absoluteLocation = ResourcePatternUtils.isUrl(location) || ResourceUtils.toURI(location).isAbsolute();
	} catch (URISyntaxException ex) {
	}
	if (absoluteLocation) {	//如果是绝对路径
		try { //加载这个资源
			int importCount = getReaderContext().getReader().loadBeanDefinitions(location, actualResources);
			if (logger.isTraceEnabled()) {
				logger.trace("Imported " + importCount + " bean definitions from URL location [" + location + "]");
			}
		} catch (BeanDefinitionStoreException ex) {
			getReaderContext().error("Failed to import bean definitions from URL location [" + location + "]", ele, ex);
		}
	} else { //如果是相对路径
		try {
			int importCount;
			Resource relativeResource = getReaderContext().getResource().createRelative(location);
			if (relativeResource.exists()) { //如果资源存在，则加载这个资源
				importCount = getReaderContext().getReader().loadBeanDefinitions(relativeResource);
				actualResources.add(relativeResource);
			} else { //如果不存在，则转换为URL处理
				String baseLocation = getReaderContext().getResource().getURL().toString();
				importCount = getReaderContext().getReader().loadBeanDefinitions(
						StringUtils.applyRelativePath(baseLocation, location), actualResources);
			}
			if (logger.isTraceEnabled()) {
				logger.trace("Imported " + importCount + " bean definitions from relative location [" + location + "]");
			}
		} catch (IOException ex) {
			getReaderContext().error("Failed to resolve current resource location", ele, ex);
		} catch (BeanDefinitionStoreException ex) {
			getReaderContext().error(
						"Failed to import bean definitions from relative location [" + location + "]", ele, ex);
		}
	}
	Resource[] actResArray = actualResources.toArray(new Resource[0]);
	//通知事件监听器，在本文案例中什么也不会做
	getReaderContext().fireImportProcessed(location, actResArray, extractSource(ele));
}

```
 上述方法的大致流程为：  
 1、获取resource属性的值  
 2、转换为路径信息  
 3、递归调用 `XmlBeanDefinitionReader` 的解析方法  
 4、通知 `ReaderContext` 绑定的事件监听器

 **1.3 <alias>标签解析**  
  `<alias>` 标签的作用的是给Bean指定别名，和 `<bean>` 标签中的name属性类似。

 
```
protected void processAliasRegistration(Element ele) {
	String name = ele.getAttribute(NAME_ATTRIBUTE); //获取name属性
	String alias = ele.getAttribute(ALIAS_ATTRIBUTE); //获取alias属性
	boolean valid = true;
	if (!StringUtils.hasText(name)) {
		getReaderContext().error("Name must not be empty", ele);
		valid = false;
	}
	if (!StringUtils.hasText(alias)) {
		getReaderContext().error("Alias must not be empty", ele);
		valid = false;
	}
	if (valid) {
		try {
			//向BeanDefinitionRegistry注册name与别名的映射关系
			getReaderContext().getRegistry().registerAlias(name, alias);
		} catch (Exception ex) {
			getReaderContext().error("Failed to register alias '" + alias +
				"' for bean with name '" + name + "'", ele, ex);
		}
		//通知事件监听器
		getReaderContext().fireAliasRegistered(name, alias, extractSource(ele));
	}
}

```
 **1.4 <beans>标签解析**  
 XML配置文件支持 `<beans>` 标签的双重嵌套，其作用和 `<import>` 类似，其解析过程和顶层的 `<beans>` 标签没有区别，都是调用了 `DefaultBeanDefinitionDocumentReader` 的 `doRegisterBeanDefinitions` 方法。

 **2、 子标签为自定义标签的解析**  
 若在遍历标签中发现了这个标签为自定义标签时，便会调用 `BeanDefinitionParserDelegate` 的 `parseCustomElement` 方法：

 
```
@Nullable
public BeanDefinition parseCustomElement(Element ele) {
	return parseCustomElement(ele, null);
}

@Nullable
public BeanDefinition parseCustomElement(Element ele, @Nullable BeanDefinition containingBd) {
	String namespaceUri = getNamespaceURI(ele);
	if (namespaceUri == null)
		return null;
	NamespaceHandler handler = this.readerContext.getNamespaceHandlerResolver().resolve(namespaceUri);
	if (handler == null) {
		error("Unable to locate Spring NamespaceHandler for XML schema namespace [" + namespaceUri + "]", ele);
		return null;
	}
	return handler.parse(ele, new ParserContext(this.readerContext, this, containingBd));
}

```
 和上面解析 `<bean>` 的自定义属性和自定义标签类似，都是先获取这个自定义标签的Namespace然后找到其对应的 `NamespaceHandler` 进行处理。

   
  