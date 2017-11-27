---
title: sbc(六)Hystrix-服务容错与保护
date: 2017/11/25 01:13:22    
categories: 
- sbc
tags: 
- Java
- SpringBoot
- SpringCloud
- Zuul
---

![](https://ws1.sinaimg.cn/large/006tNc79gy1flrejb4pbpj30qo0cuq5s.jpg)

# 前言

看过之前[SBC](https://crossoverjie.top/categories/sbc/)系列的小伙伴应该都可以搭建一个高可用、分布式的微服务了。 目前的结构图应该如下所示:
![](https://ws1.sinaimg.cn/large/006tKfTcly1flvyjrv2unj30dc0gwaaw.jpg)

各个微服务之间都不存在单点，并且都注册于 `Eureka` ，并基于此进行服务的注册于发现，再通过 `Ribbon` 进行服务调用，并具有客户端负载功能。

一切看起来都比较美好，整个系统的各个服务之间也能串联起来。但这里却忘了一个重要的细节。

> 当我们需要对外提供服务时怎么处理？

这当然也能实现，无非就是将我们具体的微服务地址加端口暴露出去即可。

那又如何来实现负载呢？

这也可以实现，可以通过 `Nginx F5` 之类的工具进行负载。

但是如果系统庞大，服务拆分的足够多那又有谁来维护这些路由关系呢？

当然这是运维的活，不过这时候运维可能就要发飙了！

并且还有一系列的问题:

- 服务调用之间的一些鉴权、签名校验怎么做？
- 由于服务端地址较多，客户端请求难以维护。

针对于这一些问题 `SpringCloud` 全家桶自然也有对应的解决方案: `Zuul`。
当我们系统整合 Zuul 网关之后架构图应该如下所示:

![](https://ws2.sinaimg.cn/large/006tKfTcly1flw0fbfukxj30mp0icdgk.jpg)

我们在所有的请求进来之前抽出一层网关应用，将服务提供的所有细节都进行了包装，这样所有的客户端都是和网关进行交互，简化了客户端开发。

同时具有如下功能:

- Zuul 注册于 `Eureka` 并集成了 `Ribbon` 所以自然也是可以从注册中心获取到服务列表进行客户端的负载。
- 功能丰富的路由功能，解放运维。
- 具有过滤器，所以鉴权、验签都可以集成。

基于此我们来看看之前的架构中如何集成 `Zuul` 。

# 集成 Zuul
为此我新建了一个项目 `sbc-gateway-zuul` 就是一个基础的 `SpringBoot` 结构。其中加入了 Zuul 的依赖：

```xml
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-zuul</artifactId>
</dependency>
```

由于需要将网关也注册到 `Eureka` 中，所以自然也需要:

```xml
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-eureka</artifactId>
</dependency>
```

紧接着配置一些项目基本信息:

```properties
# 项目配置
spring.application.name=sbc-gateway-zuul
server.context-path=/
server.port=8383

# eureka地址
eureka.client.serviceUrl.defaultZone=http://node1:8888/eureka/
eureka.instance.prefer-ip-address=true
```

在启动类中加入开启 `Zuul` 的注解，一个网关应用就算是搭好了。

```java
@SpringBootApplication

//开启zuul代理
@EnableZuulProxy
public class SbcGateWayZuulApplication {
}
```

启动 `Eureka` 和网关看到已经注册成功那就大功告成了:

![](https://ws4.sinaimg.cn/large/006tKfTcly1flx2fwc3v2j314y085dgp.jpg)

# 服务路由
路由是网关的核心功能之一，可以使系统有一个统一的对外接口，下面来看看具体的应用。

## 传统路由
传统路由非常简单，和 `Nginx` 类似，由开发、运维人员来收到维护请求地址和对应服务的映射关系，类似于:

```properties
zuul.routes.user-service.path=/user-service/**
zuul.routes.user-sercice.url=http://localhost:8080/
```

这样当我们访问 `http://localhost:8383/user-service/getUserInfo/1` 网关就会自动给我们路由到 `http://localhost:8080/getUserInfo/1` 上。

可见只要我们维护好找个映射关系即可自由的配置路由信息(`user-sercice 可自定义`)，但是很明显这种方式不管是对运维还是开发都不友好。由于实际这种方式用的不多就再过多展开。

## 服务路由
对此 `Zuul` 提供了一种基于服务的路由方式。我们只需要维护请求地址与服务 ID 之间的映射关系即可，并且由于集成了 `Ribbon` , Zuul 还可以在路由的时候通过 Eureka 实现负载调用。

具体配置：

```properties
zuul.routes.sbc-user.path=/api/user/**
zuul.routes.sbc-user.serviceId=sbc-user
```

这样当输入 `http://localhost:8383/api/user/getUserInfo/1` 时就会路由到注册到 `Eureka` 中服务 ID 为 `sbc-user` 的服务节点，如果有多节点就会按照 Ribbon 的负载算法路由到其中一台上。

以上配置还可以简写为:

```properties
# 服务路由 简化配置
zuul.routes.sbc-user=/api/user/**
```

这样让我们访问 `http://127.0.0.1:8383/api/user/userService/getUserByHystrix` 时候就会根据负载算法帮我们路由到 sbc-user 应用上，如下图所示:

![](https://ws1.sinaimg.cn/large/006tKfTcly1flx4pbe3nsj31ga0e5gnq.jpg)
启动了两个 sbc-user 服务。

请求结果:
![](https://ws4.sinaimg.cn/large/006tKfTcly1flx4q2zktbj30yd0ll79b.jpg)

一次路由就算完成了。

在上面的配置中有看到 `/api/user/**` 这样的通配符配置，具体有以下三种配置需要了解:

- `?` 只能匹配任意的单个字符，如 `/api/user/?` 就只能匹配 `/api/user/x  /api/user/y /api/user/z` 这样的路径。
- `*` 只能匹配任意字符，如 `/api/user/*` 就只能匹配 `/api/user/x /api/user/xy /api/user/xyz`。
- `**` 可以匹配任意字符、任意层级。结合了以上两种通配符的特点，如 `/api/user/**` 则可以匹配 `/api/user/x /api/user/x/y /api/user/x/y/zzz `这样的路径，最简单粗暴！

谈到通配符匹配就不得不提到一个问题，如上面的 `sbc-user` 服务由于后期迭代更新，将 sbc-user 中的一部分逻辑抽成了一个服务 `sbc-user-pro`。新应用的路由规则是 `/api/user/pro/**`,如果我们按照:

```properties
zuul.routes.sbc-user=/api/user/**
zuul.routes.sbc-user-pro=/api/user/pro/**
```

进行配置的话，我们想通过 `/api/user/pro/` 来访问 `sbc-user-pro` 应用，却由于满足第一个路由规则，所以会被 Zuul 路由到 `sbc-user` 这个应用上。该怎么解决这个问题呢？

翻看路由源码 `org.springframework.cloud.netflix.zuul.filters.SimpleRouteLocator` 中的 `locateRoutes()` 方法:

```java
	/**
	 * Compute a map of path pattern to route. The default is just a static map from the
	 * {@link ZuulProperties}, but subclasses can add dynamic calculations.
	 */
	protected Map<String, ZuulRoute> locateRoutes() {
		LinkedHashMap<String, ZuulRoute> routesMap = new LinkedHashMap<String, ZuulRoute>();
		for (ZuulRoute route : this.properties.getRoutes().values()) {
			routesMap.put(route.getPath(), route);
		}
		return routesMap;
	}
```

发现路由规则是遍历配置文件并放入 **`LinkedHashMap`** 中，由于 `LinkedHashMap` 是有序的，所以为了达到上文的效果，配置文件的加载顺序非常重要，因此我们只需要将优先匹配的路由规则放前即可解决。

# 过滤器

# Zuul 高可用

## Eureka 高可用


## Nginx 高可用

# 总结