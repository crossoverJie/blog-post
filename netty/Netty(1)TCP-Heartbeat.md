---
title: Netty(一) SpringBoot 整合长连接心跳机制
date: 2018/05/24 01:02:54       
categories: 
- Netty
tags: 
- TCP
- Heartbeat
- SpringBoot
---


![photo-1522204657746-fccce0824cfd.jpeg](https://i.loli.net/2018/05/25/5b0774828db53.jpeg)

# 前言

Netty 是一个高性能的 NIO 网络框架，本文基于 SpringBoot 以常见的心跳机制来认识 Netty。

最终能达到的效果：

- 客户端每隔 N 秒检测是否需要发送心跳。
- 服务端也每隔 N 秒检测是否需要发送心跳。
- 服务端可以主动 push 消息到客户端。
- 基于 SpringBoot 监控，可以查看实时连接以及各种应用信息。

![show](https://github.com/crossoverJie/netty-action/blob/master/pic/show.gif)


# IdleStateHandler

## 实现原理


# SpringBoot 监控

## actuator 监控

## 整合 SBA

# 总结



