---
layout:     post
title:      JAVA Spring拦截器实现及源码分析
subtitle:   
date:       2018-07-07
author:     BY
header-img: img/post-bg-unix-linux.jpg
catalog: true
tags:
    - JAVA 
    - Spring
---

> 最近在重构的项目当中，有一些用户身份验证的东西，刚开始不太了解Spring框架，都是在Controller中每次拿到http request然后调用一个公共方法解析用户token，发现这样异常的麻烦冗余不优雅，在以前的Go，PHP当中，都是在url上做一个中间件解析token，也想在JAVA中实现这样的方式。然后发现一个比较好的东西--Spring拦截器

#### 拦截器的使用

> 首先我们先来看下Spring中拦截器的使用，我用一个用户token的校验作为例子

- 自定义实现HandlerInterceptor接口

```java
public class AuthInterceptor implements HandlerInterceptor {
 }

```

- 重写preHandle方法

```java
@Override
public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
     if ("OPTIONS".equalsIgnoreCase(request.getMethod())) {
         return true;
     }
        
     String token = request.getHeader("Authorization");
     if (token == null) {
         returnError(response, ErrorCode.ERROR_PERMISSION_DENIED);
         return false;
     }

     AccountUser accountUser = authService.decodeWwwToken(token);
     if (accountUser == null) {
         returnError(response, ErrorCode.ERROR_PERMISSION_DENIED);
         return false;
     }

     request.setAttribute("accountUser", accountUser);
     return true;
}
```
- 将实现Interceptor接口的通过@Bean注册到IOC

```java

@Configuration
public class BeanConfig {

    @Bean
    public AuthInterceptor authInterceptor() {
        return new AuthInterceptor();
    }
}


```

-我这边这么创建Interceptor，而不是在 addInterceptors 中直接new出interceptor
因为 interceptor 中需要autowire使用一些service。 必须由spring来管理，否则无法autowire。

#### 拦截器执行流程

我们一步步的断点源码来看，从一个URL的请求开始

首先我们经过DispatcherServlet.java 的doDispatch方法做请求的解析分发

![](http://silenblog.oss-cn-beijing.aliyuncs.com/doDispatch.png)

经过观察，interceptors经过getHandler 方法获取
![](http://silenblog.oss-cn-beijing.aliyuncs.com/interceptors.png)

我们再来看下getHandler方法
![](http://silenblog.oss-cn-beijing.aliyuncs.com/getHandler.png)

我们来看下返回HandlerExecutionChain对象类型的HandlerMapping中getHandler的方法

回到抽象类AbstractHandlerMapping中的HandlerExecutionChain方法

```java
HandlerExecutionChain executionChain = getHandlerExecutionChain(handler, request);

```


>**在getHandlerExecutionChain方法中，我发现拦截器是从List<HandlerInterceptor> adaptedInterceptors获取的，也就是说拦截器在请求到来之前已经加载进去了，是通过@bean的编译时候就注入了，还是动态的加载进去，这我一下子也没整明白，等有时间翻看源码再来确认下这个问题**

![](http://silenblog.oss-cn-beijing.aliyuncs.com/adapted.png)

我们先接着往下看，回到DispatcherServlet文件，我们拿到了拦截器对象，我们会走applyPreHandle方法，这个方法会接收servlet解析好的request对象，去让拦截器判断，我们看下代码

![](http://silenblog.oss-cn-beijing.aliyuncs.com/prefer.png)

这我们看到了interceptor.preHandle方法，这不就是我们在实现拦截器时候重写的方法吗

下面我们自然而然的走到了我们实现HandlerInterceptor 接口的方法中。如果校验不通过，那么返回false

返回的false再经过servlet的封装，返回response

----

> 以上就是spring 拦截器的实现及源码分析，有一个未明白的点 ，等再阅读源码后再分析









