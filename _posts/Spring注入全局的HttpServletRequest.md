---
title: Spring注入全局的HttpServletRequest
tags: 编程基础
categories: 编程基础
abbrlink: 2f3880eb
date: 2021-09-24 14:14:31
---

首先要说一下是Spring的Controller默认是单例的，也就是说，如果Controller中有个field使用@Autowired自动注入，那么注入后这个field的值在该Controller中是全局的，不会再改变的(手动修改除外)。
而在Controller中，很多方法都会用到HttpServletRequest，一般会在方法的参数中写上HttpServletRequest，例如这样：

```java
@Controller
@RequestMapping("/")
public class HomeAction {
    @RequestMapping(value = {"", "/"})
    public String index(HttpServletRequest request, HttpServletResponse       response) {
		//do something
    }
    @RequestMapping(value = {"list"})
    public String list(HttpServletRequest request, HttpServletResponse response) {
		//do something
    }
}
```

Spring会自动将这个参数传进来，但是如果大量的方法都需要这个参数，可以将这个参数定义在Controller的一个field中，就像这样：

```java
@Controller
@RequestMapping("/")
public class HomeAction {
    @Autowired
    private HttpServletRequest request;
    @RequestMapping(value = {"", "/"})
    public String index(HttpServletResponse  response) {
		//do something
    }
    @RequestMapping(value = "list")
    public String list(HttpServletResponse response) {
		//do something
    }
}
```

这样所有的方法中都可以直接使用HttpServletRequest，这里第一感觉会有点奇怪，因为在HomeAction实例化的时候，HttpServletRequest就已经设置进去了，而且在该Controller实例的整个生命周期内，HttpServletRequest的值都不会变化，也就是注入的默认是单例，那么在多线程的时候，怎么能保证每次使用的HttpServletRequest都能正确对应到当前的http请求呢？所以很多人看到这里可能会觉得这个的代码有问题，会造成线程不安全，参数覆盖等等一系列。要彻底清除这个问题，得从以下几个角度来看：

### HttpServletRequest 是怎么注入进去的
所有的@Autowired进来的JDK动态代理对象的InvocationHandler处理器均为AutowireUtils.ObjectFactoryDelegatingInvocationHandler。也就是代码：
```java
	private static class ObjectFactoryDelegatingInvocationHandler implements InvocationHandler, Serializable {

		private final ObjectFactory<?> objectFactory;
		public ObjectFactoryDelegatingInvocationHandler(ObjectFactory<?> objectFactory) {
			this.objectFactory = objectFactory;
		}

		@Override
		public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
			String methodName = method.getName();
			if (methodName.equals("equals")) {
				return (proxy == args[0]);
			} else if (methodName.equals("hashCode")) {
				return System.identityHashCode(proxy);
			} else if (methodName.equals("toString")) {
				return this.objectFactory.toString();
			}
		
			// 执行目标方法。注意：目标实例对象是objectFactory.getObject()
			try {
				return method.invoke(this.objectFactory.getObject(), args);
			} catch (InvocationTargetException ex) {
				throw ex.getTargetException();
			}
		}
	}

```

该InvocationHandler处理器实现其实很“简陋”，最关键的点在于：最终invoke调用的实例是来自于objectFactory.getObject()，而这里使用的ObjectFactory是：WebApplicationContextUtils.RequestObjectFactory。至于为何使用的是这个Factory来处理，请参考web容器初始化时的这块代码：

```java
	public static void registerWebApplicationScopes(ConfigurableListableBeanFactory beanFactory, @Nullable ServletContext sc) {
		
		// web容器下新增支持了三种scope
		// 非web容器（默认）只有单例和多例两种嘛
		beanFactory.registerScope(WebApplicationContext.SCOPE_REQUEST, new RequestScope());
		beanFactory.registerScope(WebApplicationContext.SCOPE_SESSION, new SessionScope());
		if (sc != null) {
			ServletContextScope appScope = new ServletContextScope(sc);
			beanFactory.registerScope(WebApplicationContext.SCOPE_APPLICATION, appScope);
			sc.setAttribute(ServletContextScope.class.getName(), appScope);


			// ==================依赖注入=================
			// 这里决定了，若你依赖注入ServletRequest的话，就使用RequestObjectFactory来处理你
			beanFactory.registerResolvableDependency(ServletRequest.class, new RequestObjectFactory());
			beanFactory.registerResolvableDependency(ServletResponse.class, new ResponseObjectFactory());
			beanFactory.registerResolvableDependency(HttpSession.class, new SessionObjectFactory());
			beanFactory.registerResolvableDependency(WebRequest.class, new WebRequestObjectFactory());
		}

	}
```
经过这一波分析，通过@Autowired方式依赖注入得到HttpServletRequest是线程安全的结论是显而易见的了：通过JDK动态代理，每次方法调用实际调用的是实际请求对象HttpServletRequest。先对它的关键流程步骤总结如下：

* 在Spring解析HttpServletRequest类型的@Autowired依赖注入时，实际注入的是个JDK动态代理对象
* 该代理对象的处理器是：ObjectFactoryDelegatingInvocationHandler，内部实际实例由ObjectFactory动态提供，数据由RequestContextHolder请求上下文提供，请求上下文的数据在请求达到时被赋值，参照下面步骤
* 该ObjectFactory是一个RequestObjectFactory（这是由web上下文初始化时决定的）
* 请求进入时，单反只要经过了FrameworkServlet处理，便会在处理时（调用Controller目标方法前）把Request相关对象设置到RequestContextHolder的ThreadLocal中去
* 这样便完成了：调用Controller目标方法前完成了Request对象和线程的绑定，所以在目标方法里，自然就可以通过当前线程把它拿出来喽，这一切都拜托的是ThreadLocal去完成的



### @Autowired是怎么找到的

我们知道 AutowiredAnnotationBeanPostProcessor 是用来处理bean里的属性的注入的. 是众多的BeanPostProcessor里的一种，从AutowiredAnnotationBeanPostProcessor最后到DefaultListableBeanFactory.findAutowireCandidates，也就是：

* beanFactory.registerResolvableDependency(ServletRequest.class, new RequestObjectFactory()); 实际上就是往resolvableDependencies这个map里扔
* AutowiredAnnotationBeanPostProcessor 处理的时候要去找一个ServletRequest，会在resolvableDependencies找，找到一个key为ServletRequest.class的类就是RequestObjectFactory

至此怎么找到 RequestObjectFactory 也明白了。


实际上，除了HttpServletRequest外，还有很多其他内容可以通过Spring自动注入，例如：

* ServletRequest
* ServletResponse
* HttpSession
* Principal
* Locale
* InputStream
* Reader
* OutputStream
* Writer

搞明白这个流程，就不会觉得有什么问题，但是不得不说一下这个写法的形式，结合正常对Spring的bean的管理机制，确实会造成一些肉眼可见的误解。

