---
title: 「造个轮子」——cicada 设计全局上下文
date: 2018/10/09 01:19:36 
categories: 
- cicada
- 轮子
tags: 
- Java
- HTTP
- Netty
---

![](https://ws4.sinaimg.cn/large/006tNbRwgy1fvyf0zkwadj31g80teqh0.jpg)

# 前言

本次 [Cicada](https://github.com/TogetherOS/cicada) 已经更新到了 [![](https://maven-badges.herokuapp.com/maven-central/top.crossoverjie.opensource/cicada-core/badge.svg)](https://maven-badges.herokuapp.com/maven-central/top.crossoverjie.opensource/cicada-core/) `v1.0.3`。

主要是解决了两个 issue，[#9](https://github.com/TogetherOS/cicada/issues/9) [#8](https://github.com/TogetherOS/cicada/issues/8)。

所以本次的主要更新为：

- [Cicada](https://github.com/TogetherOS/cicada) 采用合理的线程分配来处理接入请求线程以及 IO 线程。
- 支持多种响应方式（以前只有 json，现在支持 text）。
- 为了满足上者引入了 `context`。
- 优雅停机。

其中我觉得最核心也最有用的就是这个 Context，并为此重构了大部分代码。

# 多种响应方式

在起初 `Cicada` 默认只能响应 `json`，这一点确实不够灵活。加上后续也打算支持模板解析，所以不如直接在 API 中加入可让用户自行选择不同的响应方式。

因此调整后的 API 如下。

> 想要输出 `text/plain` 时。

```java
@CicadaAction("textAction")
public class TextAction implements WorkAction {
    @Override
    public void execute(CicadaContext context, Param param) throws Exception {
        String url = context.request().getUrl();
        String method = context.request().getMethod();
        context.text("hello world url=" + url + " method=" + method);
    }
}
```

> 而响应输出 `application/json` 时只需要把需要响应的对象写入到 `json()` 方法中.

![](https://ws1.sinaimg.cn/large/006tNbRwgy1fvzq2nideej30ou0dlwh2.jpg)

因此原有的业务 action 中也加入了一个上下文的参数：

```java
/**
 * abstract execute method
 * @param context current context
 * @param param request params
 * @throws Exception throw exception
 */
void execute(CicadaContext context ,Param param) throws Exception;
```

下面就来看看这个 Context 是如何完成的。

# Cicada Context

先看看有了这个上下文之后可以做什么。

比如有些场景下我们需要拿到本次请求中的头信息，这时就可以通过这个 `Context` 对象直接获取。

当然不止是头信息：

- 获取请求头。
- 设置响应头。
- 设置 cookie。
- 获取请求 URL。
- 获取请求的 method（get/post）等。

其实通过这些特点可以看出这些信息其实都和一次 `请求、响应` 密切相关，并且各个请求之间的信息应当互不影响。


这样的特性是不是非常熟悉，没错那就是 `ThreadLocal`，它可以将每个线程的信息存储起来互不影响。

> ThreadLocal 的原理本次不做过多分析，只谈它在 cicada 中的应用。

## CicadaContext.class

先来看看 `CicadaContext` 这个类的主要成员变量以及方法。

![](https://ws2.sinaimg.cn/large/006tNbRwgy1fvzqkl5ct6j30nl0kb0w7.jpg)

成员变量是两个接口 `CicadaRequest、CicadaResponse`，看名称就能看出肯定是存放请求和响应数据的。


## HttpDispatcher.class

想要存放本次请求的上下文自然是在真正请求分发的地方 `HttpDispatcher`。

![](https://ws3.sinaimg.cn/large/006tNbRwgy1fvzqt0dlwhj30ne0kudjn.jpg)

这里改的较大的就是两个红框处，第一部分是做山下文初始化及赋值。

第二部分自然就是卸载山下文。


> 先看初始化。

`CicadaRequest cicadaRequest = CicadaHttpRequest.init(defaultHttpRequest) ;`

首先是讲 request 初始化：

`CicadaHttpRequest` 自然是实现了 `CicadaRequest` 接口：

![](https://ws3.sinaimg.cn/large/006tNbRwgy1fvzqx6hvpcj30gt0c70u7.jpg)

这里只保存了请求的 URL、method 等信息，后续要加的请求头也存放在此处即可。

`Response` 也是同理的。

![](https://ws2.sinaimg.cn/large/006tNbRwgy1fvzqyz3bfjj30l20gigod.jpg)

> 这两个具体的实现类都私有化了构造函数，防止外部破坏了整体性。

接着将当前请求的上下文保存到了 `CicadaContext` 中。

```java
CicadaContext.setContext(new CicadaContext(cicadaRequest,cicadaResponse));
```

而这个函数本质使用的则是 `ThreadLocal` 来存放 `CicadaContext`。

```java
    public static void setContext(CicadaContext context){
        ThreadLocalHolder.setCicadaContext(context) ;
    }
    
    private static final ThreadLocal<CicadaContext> CICADA_CONTEXT= new ThreadLocal() ;
    
    /**
     * set cicada context
     * @param context current context
     */
    public static void setCicadaContext(CicadaContext context){
        CICADA_CONTEXT.set(context) ;
    }
```

## 处理业务及响应

接着就是处理业务，调用不同的 API 做不同响应。

拿 `context.text()` 来说：

![](https://ws4.sinaimg.cn/large/006tNbRwgy1fvzr73fkrsj30o404ygm8.jpg)

其实就是设置了对应想响应方式、以及把响应内容写入了 `CicadaResponse` 的 `httpContent` 中。

业务处理完后调用 `responseContent()` 进行响应：

```java
responseContent(ctx,CicadaContext.getResponse().getHttpContent());
```

其实就是在上下文中拿到的响应方式及响应内容返回给客户端。

## 卸载上下文

最后有点非常重要，那就是 **卸载上下文**。

如果这里不做处理，之后随着请求的增多，ThreadLocal 里存放的数据也越来越多，最后肯定会导致内存溢出。

所以 `CicadaContext.removeContext()` 就是为了及时删除当前上下文的。


# 优雅停机

最后还新增了一个停机的方法。

![](https://ws3.sinaimg.cn/large/006tNbRwgy1fvzrfl6dsnj30j60bi75u.jpg)

其实也就是利用 Hook 函数实现的。

由于目前 Cicada 开的线程，占用的资源都不是特别多，所以只是关闭了 Netty 所使用的线程。

如果后续新增了自身的线程等资源，那也可以全部放到这里来进行释放。

# 总结

`Cicada` 已经更新了 4 个版本，雏形都有了。

后续会重点实现模板解析和注解请求路由完成，把 `MVC` 中的 `view` 完成就差不多了。

还没有了解的朋友可以点击下面链接进入主页了解下😋。

[https://github.com/TogetherOS/cicada](https://github.com/TogetherOS/cicada)