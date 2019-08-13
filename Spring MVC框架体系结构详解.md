---
title: Spring MVC框架体系结构详解
date: 2019-06-14 01:22:23
tags: CSDN迁移
---
  ### []()概述

 无论是基于Spring Boot还是基于SSM框架的Web开发，都涉及到了Spring MVC框架的使用。Spring MVC是Spring Framework的其中一个产品，是一个强大灵活的基于MVC的Web框架，它包含以下优点：

  
  * 请求统一通过内置的 `DispatcherServlet` 处理，开发者无需编写难以管理、维护的 `Filter` 、 `Servlet` 类。 
  * 分工明确，包括控制器、验证器、命令对象、模型对象、处理程序映射视图解析器等等，每一个功能实现由一个专门的对象负责完成 
  * 自带 `Spring` 框架的IoC和AOP功能，有效降低了代码的耦合度 
  * 配置相对于传统的 `web.xml` 更加简单方便，维护起来更加容易 
  * 支持国际化 
  * 支持多种视图技术  本文假定读者有一定的Spring MVC开发经验。

 
### []()DispatcherServlet

 `DispatcherServlet` 是整个Spring MVC的核心，它的继承体系如下：  
 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190613212909604.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FiYzEyM2x6Zg==,size_16,color_FFFFFF,t_70)  
 可以看出它实现了 `javax.servlet.Servlet` 接口，所以它本质上是一个 `Servlet` ，并可通过 `web.xml` 文件注册到Servlet容器上。  
 并且，它也实现了 `ApplicationContextAware` 和 `EnvironmentAware` 接口，所以在 `Spring` 容器在创建 `DispatcherServlet` 实例的时候，也会将IoC容器 `ApplicationContext` 和环境参数容器 `Environment` 注入到 `DispatcherServlet` ，使其可以利用到Spring的IoC容器。

 一般在开发基于Spring MVC的Web应用的时候，我们一般会在 `web.xml` 文件中这么定义这个 `DispatcherServlet` ：

 
```
<servlet>
	<servlet-name>DispatcherServlet</servlet-name>
	<servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
	<load-on-startup>1</load-on-startup>
	<init-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>classpath:applicationContext.xml</param-value>
    </init-param>
</servlet>
<servlet-mapping>
    <servlet-name>DispatcherServlet</servlet-name>
    <url-pattern>/</url-pattern>
</servlet-mapping>

```
 可以看出和一般的 `Servlet` 配置没什么区别，我们只需要将所有的URL请求产生的 `HttpServletRequest` 都由这个 `DispatcherServlet` 进行处理就可以了。同样和其它 `Servlet` 类似， `DispatcherServlet` 是单例的，多个请求由同一个 `DispatcherServlet` 处理。

 `DispatcherServlet` 处理请求的流程可以用以下图片概括：  
 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190614010513145.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FiYzEyM2x6Zg==,size_16,color_FFFFFF,t_70)  
 其处理流程可以概括为：

  
  2. 客户端发起请求后，由 `Servlet` 容器将请求包装为 `HttpServletRequest` 对象，然后 `Servlet` 容器通过反射创建 `DispatcherServlet` 对象，调用其 `init` 方法并传入 `ServletConfig` 对象（存储了 `web.xml` 文件定义的 `init-param` 参数）初始化 `DispatcherServlet` ，然后调用其 `service` 方法并传入 `HttpServletRequest` 和 `HttpServletResponse` 进行服务。 
  4. `DispatcherServlet` 接收到 `Servlet` 容器传来的 `HttpServletRequest` 和 `HttpServletResponse` 后，会通过其URL，找到其对应的 `HandlerMapping` 。 
  6. 调用 `HandlerMapping` 实例的 `getHandler` 方法并传入 `HttpServletRequest` ，获取 `HandlerExecutionChain` （包含一个 `Handler` 对象和多个 `HandlerInterceptor` ） 
  8. 执行 `HandlerExecutionChain` 包含的 `HandlerInterceptor` 中的前置处理方法 
  10. 根据 `HandlerExecutionChain` 包含的 `Handler` 找到其对应的 `HandlerAdaptor` 。 
  12. 调用该 `HandlerAdaptor` 的 `handler` 方法并传入 `HttpServletRequest` 、 `HttpServletResponse` 和 `Handler` 对象，返回 `ModelAndView` 对象。 
  14. 如果 `ModelAndView` 指定的是视图名称而不是 `View` 实例（一般情况），那么调用 `ViewResolve` 的方法得到 `View` 对象。 
  16. 调用 `View` 对象的 `render` 并传入模型数据，渲染视图然后写入到 `HttpServletResponse` 。请求处理完成。  我们来分别讲解上述元件的基本构造。

 
--------
 
### []()HandlerMapping

 `HandlerMapping` 负责帮助 `DispatcherServlet` 进行HTTP请求的URL到具体 `Handler` 的**匹配**，也就是Web请求到 `Handler` （我们熟知的 `Controller` 其实就是 `Handler` 的一种）的**映射关系**。

 `HandlerMapping` 有以下几种类型：

  
  * `SimpleUrlHandlerMapping` ，可以用来处理具体的URL与Handler之间的映射 
  * `RequestMappingHandlerMapping` ，用来处理具体的URL与基于注解的 `Controller` 之间的映射关系， `@RequestMapping` 就是通过这个 `HandlerMapping` 实现的。  其它类型相对来说用的比较少，这里就不再阐述了。

 一个 `DispatcherServlet` 是可以拥有多个 `HandlerMapping` 实例的，那这么多 `HandlerMapping` 实例怎么才能找到对应的 `HandlerMapping` 呢？  
 答案很简单，只需要遍历 `HandlerMapping` 集合就可以了：  
  `DispatcherServlet` 通过 `List` 来保存 `HandlerMapping` ：

 
```
@Nullable private List<HandlerMapping> handlerMappings;

```
 HandlerMapping本质上是一个接口，它只定义了一个抽象方法：

 
```
@Nullable HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception;

```
 注意其返回值是 `@Nullable` 注解修饰的，也就是说，只需要根据返回值是否为 `null` 来判断有没有找到合适的 `HandlerMapping` 就可以了。 `DispatcherServlet` 也的确是这么实现的：

 
```
@Nullable
protected HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
	if (this.handlerMappings != null) {
		for (HandlerMapping mapping : this.handlerMappings) {
			HandlerExecutionChain handler = mapping.getHandler(request);
			if (handler != null) {
				return handler;
			}
		}
	}
	return null;
}

```
 如果要是没有找到合适的 `HandlerMapping` ，那么就会向客户端返回一个**404错误**。  
 虽然 `HandlerMapping` 接口没有实现 `Ordered` 接口，但是其官方实现类都实现了 `Ordered` 接口，在 `HandlerMapping` 对象添加到 `List` 集合时，会根据 `Ordered` 接口定义的方法的返回值来进行排序，优先级高的 `HandlerMapping` 会排在前面。

 `HandlerMapping` 在找到合适的 `Handler` 和 `HandlerInterceptor` 后，会将其包装为一个 `HandlerExecutorChain` 对象并返回。  
  `HandlerExecutionChain` 可以理解为一个HTTP请求处理链，它包含一个 `Handler` 实例和多个 `HandlerInterceptor` 实例：

 
```
private final Object handler;
@Nullable private HandlerInterceptor[] interceptors;
@Nullable private List<HandlerInterceptor> interceptorList;
private int interceptorIndex = -1;

```
 可以发现 `Handler` 是 `Object` 类型的，之所以这么设计是为了**可扩展性**考虑的，通过这种机制不仅仅可以实现Spring MVC自带的 `Controller` ，还可以通过扩展实现 `Struts` 的 `Action` 。

 
--------
 
### []()HandlerAdaptor

 刚才已经提到 `Handler` 是 `Object` 类型的，那么如何执行 `Handler` 处理我们编写的业务逻辑呢？ `HandlerAdaptor` 就是为此诞生的，它作为一个适配器，屏蔽了不同 `Handler` 类型给 `DispatcherServlet` 带来的“麻烦”，它可以看成是一个 `DispatcherServlet` 和 `Handler` 之间的“中间人”。

 `HandlerAdaptor` 本身是一个接口，它定义了以下几个抽象方法：

 
```
public interface HandlerAdapter {
	boolean supports(Object handler);
	@Nullable ModelAndView handle(HttpServletRequest request, HttpServletResponse response, 
					              Object handler) throws Exception;
	long getLastModified(HttpServletRequest request, Object handler);
}

```
 其中 `supports` 方法就是用来判断这个 `HandlerAdaptor` 支不支持处理这个 `Handler` 。  
  `DispatcherServlet` 在遍历 `HandlerAdaptor` 集合的时候，会首先调用 `supports` 方法并传入 `HandlerExecutorChain` 中的 `Handler` 获取其判定结果，如果为 `false` ，那么继续遍历下一个 `HandlerAdaptor` 。如果返回结果为 `true` ，那么就会调用其 `handle` 方法并获取 `ModelAndView` 对象。

 至于 `getLastModified` 方法，它的主要目的只是为返回给客户端的 `Last-Modified` 这个HTTP响应头提供其相应的时间戳，如果不想支持此功能直接返回-1即可。

 我们以处理 `Controller` 类型的 `HandlerAdaptor` ： `SimpleControllerHandlerAdapter` 为例看看它是如何处理的：

 
```
public class SimpleControllerHandlerAdapter implements HandlerAdapter {

	@Override
	public boolean supports(Object handler) {
		return (handler instanceof Controller);
	}

	@Override
	@Nullable
	public ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler)
			throws Exception {
		return ((Controller) handler).handleRequest(request, response);
	}

	@Override
	public long getLastModified(HttpServletRequest request, Object handler) {
		if (handler instanceof LastModified) {
			return ((LastModified) handler).getLastModified(request);
		}
		return -1L;
	}
}

```
 很简单， `supports` 方法只需要判断这个 `Handler` 是不是 `Controller` 类型的就好了。至于其 `handler` 方法，就是进行一个强制类型转换然后调用 `Controller` 的 `handlerRequest` 方法。

 
--------
 
### []()HandlerInterceptor

 `HandlerInterceptor` 可以看成是一种 `DispatcherServlet` 内部的 `Filter` ，它可以在 `Handler` 执行前后对处理流程进行拦截。

 `HandlerInterceptor` 本质上同样是一个接口，它定义了三种方法：

 
```
public interface HandlerInterceptor {
	default boolean preHandle(HttpServletRequest request, HttpServletResponse response, 
			Object handler) throws Exception {
		return true;
	}
	
	default void postHandle(HttpServletRequest request, HttpServletResponse response, 
			Object handler, @Nullable ModelAndView modelAndView) throws Exception {
	}
	
	default void afterCompletion(HttpServletRequest request, HttpServletResponse response, 
			Object handler, @Nullable Exception ex) throws Exception {
	}
}

```
 这三种方法均是拦截方法：

  
  * preHandler：这个拦截方法会**在调用 `HandlerAdaptor` 之前执行**， `DispatcherServlet` 会根据该方法的 `boolean` 返回值决定是否继续执行后续流程。如果返回值为 `true` ，那么后继的 `HandlerInterceptor` 的 `preHandler` 会继续调用，如果不存在后继 `HandlerInterceptor` 那么会直接执行对应的 `HandlerAdaptor` 。如果返回值为 `false` 或是**抛出了异常**，那么后继 `HandlerInterceptor` 和 `HandlerAdaptor` 就不会继续执行了。 
  * postHandler：**在 `Handler` 执行完毕，视图的解析和渲染之前执行**。在这个时候我们就可以处理 `HandlerAdaptor` 返回的 `ModelAndView` 了，例如可以修改模型数据或是视图。但是**不可阻断后续处理流程**。 
  * afterCompletion： `Dispatcher` 已经完成了视图渲染后，不管是否发生了异常该方法都会得到执行。如果上述处理流程是以异常结束的，那么在这里我们就可以**得到这个异常的引用**并处理这个异常了。  在Spring 5.0以后的Web开发中，我们可以通过实现 `WebMvcConfigurer` 接口来添加 `HandlerInterceptor` 。

 
--------
 
### []()HandlerExceptionResolver

 `HandlerExceptionResolver` 用于在 `Handler` 查找和 `Handler` 执行期间提供异常处理机制。如果在Handler执行期间没有抛出任何异常，就会返回 `ModelAndView` ，而一旦抛出异常， `HandlerExceptionResolver` 就立刻接手异常的处理，并同样返回一个 `ModelAndView` 。

 `HandlerExceptionResolver` 接口源代码如下：

 
```
public interface HandlerExceptionResolver {
@Nullable
ModelAndView resolveException(HttpServletRequest request, HttpServletResponse response, 
					          @Nullable Object handler, Exception ex);
}

```
 该接口只定义了一个方法 `resolveException` ，在 `Dispatcher` 调用时会传入 `Handler` 抛出的异常的引用和 `Handler` 实例。

 
--------
 
### []()ViewResolver

 `ViewResolver` 即视图定位器，它的主要职责是根据 `HandlerAdaptor` 返回的 `ModelAndView` 中的**逻辑视图名**，为 `DispatcherServlet` 提供一个 `View` 实例。当然如果 `Handler` 在 `ModelAndView` 直接指定了 `View` 实例，那么就不会调用 `ViewResolver` 。

 `ViewResolver` 接口源代码如下：

 
```
public interface ViewResolver {
	@Nullable
	View resolveViewName(String viewName, Locale locale) throws Exception;
}

```
 `ViewResolver` 实现类只需要根据逻辑视图名和 `Locale` （用于实现国际化）返回相应的 `View` 实例即可。

 大部分 `ViewResolve` 实现类都继承了 `AbstractCachingViewResolver` ，也就是说大部分 `ViewResolve` 实现类都加入了 `View` 缓存的功能，因为每次实例化 `View` 是一个开销比较大的操作。

 常用的 `ViewResolve` 实现类有：

  
  * `InternalResourceViewResolver` ：它对应 `InternalResourceView` ，用于处理JSP模板。 
  * `FreeMarkerViewResolver` ：用于定位 `Freemarker` 类型的视图。 
  * `ResourceBundleViewResolver` ：该实现类构建于 `ResourceBoundle` ，支持国际化。  
--------
 
### []()View

 `View` 即视图，负责通过模型数据向 `HttpServletResponse` 写入用户可见的界面， `Spring MVC` 提供了基于JSP、Velocity、Freemarker、Excel、PDF等视图技术。

 `View` 接口源代码如下：

 
```
public interface View {
	String RESPONSE_STATUS_ATTRIBUTE = View.class.getName() + ".responseStatus";
	String PATH_VARIABLES = View.class.getName() + ".pathVariables";
	String SELECTED_CONTENT_TYPE = View.class.getName() + ".selectedContentType";

	@Nullable
	default String getContentType() {
		return null;
	}

	void render(@Nullable Map<String, ?> model, HttpServletRequest request, HttpServletResponse response)
			throws Exception;
}

```
 其中， `getContentType` 用于向HTTP响应头写入 `content-type` 数据，好让浏览器知道视图是什么类型，例如HTTP页面返回的就应当是 `text/html` 。  
  `render` 方法用于根据模型数据渲染视图，并写入到响应体。

   
  