---
title: 撸了一个可调试 gRPC 的GUI客户端
date: 2021/11/15 08:10:16 
categories: 
- Go
tags: 
- grpc
---


![](https://tva1.sinaimg.cn/large/008i3skNly1gwuz3q9a2nj30rs0rs3z1.jpg)

# 前言

平时大家写完 `gRPC` 接口后是如何测试的？往往有以下几个方法：

1. 写单测代码，自己模拟客户端测试。
![](https://tva1.sinaimg.cn/large/008i3skNly1gwv0138u2ij31eq0lwn07.jpg)

2. 第二种就是可以搭一个 `gRPC-Gateway` 服务，这样就可以在 `postman` 中进行模拟。

但这两种方法都不是特别优雅；第一种方法当请求结构体嵌套特别复杂时，在代码中看起来就不是很直观。
而且代码会特别长。

第二种方法在 postman 中与请求 HTTP 接口一样，看起来非常直观；但需要额为维护一个 `gRPC-Gateway` 服务，同时接口定义发生变化时也得重新发布，使用起来稍显复杂。

于是我经过一番搜索找到了两个看起来还不错的工具：
- `BloomRPC`

# 核心功能

## 安装

# 后续计划
