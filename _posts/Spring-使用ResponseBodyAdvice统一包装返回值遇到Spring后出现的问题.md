---
title: Spring 使用ResponseBodyAdvice统一包装返回值遇到Spring后出现的问题
tags: 编程基础
categories: 编程基础
abbrlink: f4c7ba3e
date: 2021-10-14 16:40:26
---

考虑到多人协作与代码封装性，我将Controller方法的返回值和HTTP请求最终的返回值做了隔离，也就是在开发过程中在Controller的方法中返回都是具备业务含义的POJO，但是在最终通过HTTP返回的时候都会被包装成如下格式：

```java
public class Response<T> {
    private int code;
    private String msg;
    private T data;
}
```
其中的data就是实际的业务内容，要实现这个效果很简单，就是Spring的APO，通过自定义ResponseBodyAdvice，在beforeBodyWrite方法中包装内容，可以在supports中加一些过滤条件，例如如果有某个注解就不包装等，提示灵活性，统一的错误处理在handleTException方法中。

```java
@RestControllerAdvice
public class APIResponseAdvice implements ResponseBodyAdvice<Object> {

    private int codeOk = 0;
    private int codeErr = 1;
   
    @ExceptionHandler(Exception.class)
    public MetadataResponse handleTException(HttpServletRequest request, TException ex) {
        log.error("process url {} failed", request.getRequestURL().toString(), ex);
        return Response.builder().code(codeErr).msg(ex.getMessage()).build();
    }

    @Override
    public boolean supports(MethodParameter returnType, @NotNull Class converterType) {
        return returnType.getParameterType() != Response.class
                && AnnotationUtils.findAnnotation(Objects.requireNonNull(returnType.getMethod()), NotWrapperResponse.class) == null
                && AnnotationUtils.findAnnotation(returnType.getDeclaringClass(), NotWrapperResponse.class) == null;
    }

       @Override
    public Object beforeBodyWrite(Object body,
                                  @NotNull MethodParameter returnType,
                                  @NotNull MediaType selectedContentType,
                                  @NotNull Class<? extends HttpMessageConverter<?>> selectedConverterType,
                                  @NotNull ServerHttpRequest request, @NotNull ServerHttpResponse response) {
        return Response.builder()
                .code(codeOk).data(body).build();
    }
}

```

NotWrapperResponse就是一个标记，如果Controller的方法上出现这个注解则不进行包装。

```java
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
public @interface NotWrapperResponse {

}
```
看起来没有什么问题，不过实际在运行的时候出现一个问题，就是如果返回值是String会出现这样的问题：

```java
java.lang.ClassCastException: com.xxx.Response cannot be cast to java.lang.String
	at org.springframework.http.converter.StringHttpMessageConverter.addDefaultHeaders(StringHttpMessageConverter.java:44) ~[spring-web-5.2.5.RELEASE.jar:5.2.5.RELEASE]

```

先说原因，在beforeBodyWrite方法中有一个参数是selectedConverterType，这个参数就是对内容解析的对象，正常来说如果是Pojo的话，他的解析对象是MappingJackson2HttpMessageConverter，但是恰巧如果是String返回值的话，默认生效的是org.springframework.http.converter.StringHttpMessageConverter，也就是返回值类型为字符串的时候，mvc框架就会将其跳转到路径为返回内容的地址。

如果将返回值类型改为int后，可以发现mvc框架选中的转换器也是org.springframework.http.converter.json.MappingJackson2HttpMessageConverter。
那么问题是找到了，返回字符串的时候采用的是特殊的StringHttpMessageConverter转换器，而其他格式则是采用MappingJackson2HttpMessageConverter转换器来解析的，StringHttpMessageConverter转换器将内容转为String字符串，而当前我们返回的则是一个具体的对象，这就导致了报错的根本原因也就是类型转换异常。

要解决这个问题可以直接把转换器的级别提升，也就是在创建WebMvcConfigurer对象的时候配置转换器，如下：
```java
 	@Bean
    public WebMvcConfigurer createWebMvcConfigurer(@Autowired HandlerInterceptor[] interceptors) {
        return new WebMvcConfigurer() {
            public void addInterceptors(InterceptorRegistry registry) {
                for (HandlerInterceptor interceptor : interceptors) {
                    registry.addInterceptor(interceptor);
                }
            }

            @Override
            public void configureMessageConverters(List<HttpMessageConverter<?>> converters) {
                converters.add(0, new MappingJackson2HttpMessageConverter());
            }
        };
    }
```
将MappingJackson2HttpMessageConverter的优先级提高，当然也可以在AOP的时候重新选择，而不是直接这样写，不过都能搞定，重要的是理解为什么会出现这个问题就行。

但是，因为我是在一个庞大的遗留系统上开发，这样修改后，解决了返回值的问题，随之而以来又引入另一个问题，那就是如果Controller里面出现：

```java
 	public XXXX cancelQuery(@RequestBody String id) {
        
    }
```
会导致出现Json解析错误，因为前面的修改方案会让所有字符串的解析都走org.springframework.http.converter.json.MappingJackson2HttpMessageConverter，则在RequestBody的时候也会使用Json的converter，而这里的id是个纯字符串，会出现json的解析错误。

所以最后是在beforeBodyWrite中，使用：
```java
		if (body instanceof String) {
            return JSON.toJSONString(Response.builder()
                    .code(codeOk).data(body).build());
        }
```
来判断，然后转成Json，这里也有一个潜在的不完美就是这里虽然toJson了，但是走的还是String的Convert，则会导致整个Json内容以字符串的形式返回给前端。
