---
title: 动态代理与RPC
date: 2020/04/28 09:10:36 
categories: 
- cim
- rpc
- 动态代理
tags: 
- Java
- Netty
---

![](https://tva1.sinaimg.cn/large/007S8ZIlly1ge6hhdwokqj311j0u0e0a.jpg)

# 前言

![](https://tva1.sinaimg.cn/large/007S8ZIlly1ge6hoauyzej31j60ei41y.jpg)
随着最近关注 [cim](https://github.com/crossoverJie/cim) 项目的人越发增多，导致提的问题以及 Bug 也在增加，在修复问题的过程中难免代码洁癖又上来了。

看着一两年前写的东西总是怀疑这真的是出自自己手里嘛？有些地方实在忍不住了便开始了漫漫重构之路。


# 前后对比

在开始之前想简单介绍一下 `cim` 这个项目，下面是它的架构图：
![](https://tva1.sinaimg.cn/large/007S8ZIlly1ge7onwt13dj315o0r4n1g.jpg)

简单来说就是一个 IM 即时通讯，主要有以下部分组成：

- IM-server 自然就是服务端了，用于和客户端保持长连接。
- IM-client 客户端，可以简单认为类似于的 QQ 可以的客户端工具；当然功能肯定没那么丰富，只提供了一些简单消息发送、接收的功能。
- Route 路由服务，主要用于客户端鉴权、消息的转发等；提供一些 http 接口，可以用于查看系统状态、在线人数等功能。

当然服务端、路由都可以水平扩展。

---

![](https://tva1.sinaimg.cn/large/007S8ZIlly1ge7ofwaw13j317i0u0q7g.jpg)

这是一个消息发送的流程图，假设现在部署了两个服务端 A、B 和一个路由服务；其中 ClientA 和 ClientB 分别和服务端 A、B 保持上连接。

当 ClientA 向 ClientB 发送一个 `hello world` 时，整个的消息流转如图所示：

1. 先通过 http 将消息发送到 Route 服务。
2. 路由服务得知 ClientB 是连接在 ServerB 上；于是再通过 http 将消息发送给 ServerB。
3. 最终 ServerB 将消息通过与 ClientB 的长连接通道 push 下去，至此消息发送成功。





# 绕不过去的动态代理

# RPC

# 总结

**你的点赞与分享是对我最大的支持**
