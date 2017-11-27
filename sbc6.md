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

# 服务路由


## 传统路由

## 服务路由

## 请求负载

# 过滤器

# Zuul 高可用

## Eureka 高可用


## Nginx 高可用

# 总结