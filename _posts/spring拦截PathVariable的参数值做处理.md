---
title: spring拦截PathVariable的参数值做处理
tags: 编程基础
categories: 编程基础
abbrlink: ea905952
date: 2021-11-24 11:55:22
---

工程中有一个模块是个server模块，使用了spring boot进行构建，目前大部分的参数是通过@PathVariable进行传递的，也就是在controller中是：

```java
@PostMapping("/{dbName}/{tableName}")
public xxx xxx(@PathVariable String dbName, @PathVariable String tableName) {
    return xxx;
}
```
类似上面这样的写法，当传入dbName的时候，通过hive的java sdk去访问metastore，获取db的信息，hive metastore的api如下：

```java
@Override
public Database getDatabase(String name) throws TException {
   return getDatabase(getDefaultCatalog(conf), name);
}
```
然后碰到一个问题是controller的参数是调用方传递过来的，因为是db name，所以很多调用方习惯添加 \`\` 或者添加双引号来标记d，而添加了这样的符号在传入到hive metastore api的时候是无法被识别的。

解决办法是可以在controller中做一个replace all，像这样：
```java
source.replaceAll("`", "").replaceAll("\"", "")
```
但是我不想让这类filter的东西侵入到业务逻辑，再说在每一个controller中调用一个replace 也很丑，维护也不方便，所以想着能不能使用一个Interceptor统一拦截过滤。

最开始想的是自定义PathVariable的参数解析器，也就是自己实现PathVariableMethodArgumentResolver，重写supportsParameter和resolveArgument，也就是这样：

```java
@Override
public boolean supportsParameter(MethodParameter methodParameter) {
    return true;
}

@Override 
public Object resolveArgument(MethodParameter methodParameter, 
                              ModelAndViewContainer modelAndViewContainer, 
                              NativeWebRequest nativeWebRequest, 
                              WebDataBinderFactory webDataBinderFactory ) throws Exception {
    //在这里进行替换
    return xxxx;
}
```
同时需要把这个自定义的处理器添加到WebMvcConfigurer中：

```java
@Override
public void addArgumentResolvers(List<HandlerMethodArgumentResolver> resolvers) {
      resolvers.add(new 自己定义的类());
}
```

但是在实际运行的时候我发现自定义的参数处理器不生效，这个就比较迷惑了，打算研究一下发生了什么，首先跟踪了一下WebMvcConfigurer的addArgumentResolvers，看到创建RequestMappingHadnlerAdapter时会调用所有的WebMvcConfigurer的实现类的addArgumentResolvers方法，将对应的argumentResolver添加
RequestMappingHadnlerAdapter的customArgumentResolvers属性中。

也就是我自己定义的ArgumentResolver是被添加了进去的，在使用的时候通过getArgumentResolver获取要用到的参数处理器，代码在org.springframework.web.method.support.HandlerMethodArgumentResolverComposite#getArgumentResolver ：

```java
@Nullable
private HandlerMethodArgumentResolver getArgumentResolver(MethodParameter parameter) {
	HandlerMethodArgumentResolver result = this.argumentResolverCache.get(parameter);
	if (result == null) {
		for (HandlerMethodArgumentResolver resolver : this.argumentResolvers) {
			if (resolver.supportsParameter(parameter)) {
				result = resolver;
				this.argumentResolverCache.put(parameter, result);
				break;
			}
		}
	}
	return result;
}
```

如果自己实现的ArgumentResolver中的supportsParameter返回是true，则会把这对象放入argumentResolverCache缓存中，并且break打断循环。这就意味着所有的参数处理器中只会调用其中的一个处理器，无论有多少个，始终只有第一个会生效。

这也是为什么我自己定义的没有生效的原因，看到这里了我就在想既然如此我就把自己定义的resolver放到第一个，那么问题就解决了，也就是需要找到 RequestMappingHandlerAdapter是在什么时候将customArgumentRsolvers加入到argumentResolvers里面的，再次跟踪下RequestMappingHandlerAdapter实例化bean过程的源码。
发现getDefaultArgumentResolvers()方法后argumentResolvers里面放入了30个argumentResolver，那么由此可知，关键代码就在getDefaultArgumentResolvers方法上，以下是方法源码：

```java
protected List<HandlerMethodArgumentResolver> getDefaultArgumentResolvers() {
    List<HandlerMethodArgumentResolver> resolvers = new ArrayList();
    resolvers.add(new SessionAttributeMethodArgumentResolver());
    resolvers.add(new RequestAttributeMethodArgumentResolver());
    resolvers.add(new ServletRequestMethodArgumentResolver());
    resolvers.add(new ServletResponseMethodArgumentResolver());
    resolvers.add(new RedirectAttributesMethodArgumentResolver());
    resolvers.add(new ModelMethodProcessor());
    if (this.getCustomArgumentResolvers() != null) {
        resolvers.addAll(this.getCustomArgumentResolvers());
    }
    return resolvers;
}
```

在这里可以看到这个顺序，是hard code写死了的，也就是意味着没有预留按需调整顺序的接口，无法调整，于是思路只能从IOC容器预留的扩展接口来对RequestMappingHandlerAdapter实例进行修改了。

不需要借助mvc框架的WebMvcConfigurer接口,直接使用IOC容器的扩展接口BeanPostProcessor,实现BeanPostProcessor的postProcessAfterInitialization方法,直接将自定义的处理器添加到到RequestMappingHandlerAdapter的argumentResolvers属性里面即可轻易实现：

```java
public class ResolverBeanPostProcessor implements BeanPostProcessor {
   
    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        if(beanName.equals("requestMappingHandlerAdapter")){
            //requestMappingHandlerAdapter进行修改
            RequestMappingHandlerAdapter adapter = (RequestMappingHandlerAdapter)bean;
            List<HandlerMethodArgumentResolver> argumentResolvers = adapter.getArgumentResolvers();
            //添加自定义参数处理器
            argumentResolvers = addArgumentResolvers(argumentResolvers);
            adapter.setArgumentResolvers(argumentResolvers);
        }
        return bean;
    }

    private List<HandlerMethodArgumentResolver> addArgumentResolvers(List<HandlerMethodArgumentResolver> argumentResolvers) {
        List<HandlerMethodArgumentResolver> resolvers = new ArrayList<>();
        //将自定的添加到最前面
        resolvers.add(你自己定义的resolver);
        //将原本的添加后面
        resolvers.addAll(argumentResolvers);
        return  resolvers;
    }
}
```

后来我在想既然spring hard code把这个顺序写死了，也没有预留接口去修改，那么按理说应该是不建议调整这个顺序，也就是自己定义的resolver可能会破坏spring自身的一些处理逻辑，那么应该有其他办法可以完成相同的功能。

去看了下源码发现spring里面有很多Converter，用来在spring内部实现数据转换，原生的Java是有一个可以提供数据转换功能的工具——PropertyEditor。但是它的功能有限，它只能将字符串转换为一个Java对象。在web项目中，如果只看与前端交互的那一部分，这个功能的确已经足够了。但是在后台项目内部可就得重新想办法了。

Spring针对这个问题设计了Converter模块，它位于org.springframework.core.converter包中。该模块足以替代原生的PropertyEditor，但是spring选择了同时支持两者，在Spring MVC处理参数绑定时就用到了。

于是我在想是否可以自己定一个转换器，在参数进入controller之前进行转换，这样的话整个值就被处理掉了，也就是这样：

```java
@Component
public class XXXXConverter implements Converter<String, String> {
    @Override
    public String convert(String source) {
        return source.replaceAll("`", "").replaceAll("\"", "");
    }
}
```

发现这方式果然是可行的，于是这个问题算是比较优雅的解决了。




