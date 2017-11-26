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

# 服务路由

## 传统路由

## 服务路由

## 请求负载

# 过滤器

# Zuul 高可用

## Eureka 高可用


## Nginx 高可用

# 总结