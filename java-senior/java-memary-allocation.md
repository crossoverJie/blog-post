---
title: 老板让我写个 Bug
date: 2018/12/12 08:01:36 
categories: 
- Java 进阶
tags: 
- Java
- JVM
---

![](https://ws3.sinaimg.cn/large/006tNbRwly1fy25iirb5tj31hc0u0e81.jpg)

# 前言

标题没有看错，真的是让我写个 `bug`！

刚接到这个需求时我内心没有丝毫波澜，甚至还有点激动。这可是我特长啊；终于可以光明正大的写 `bug` 了🙄。

先来看看具体是要干啥吧，主要就是要让一些负载很低的服务器额外消耗一些内存、CPU 等资源（至于背景就不多说了），让它的负载可以提高一些。


# JVM 内存分配回顾

于是我刷刷一把梭的把代码写好了，大概如下：

![](https://ws2.sinaimg.cn/large/006tNbRwly1fy2t4bjv5bj318s0hgjv4.jpg)



## 优先在 Eden 区分配对象

## 大对象直接进入老年代

# Linux 内存查看

# 总结

**你的点赞与分享是对我最大的支持**
