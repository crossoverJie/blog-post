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
- [https://github.com/fullstorydev/grpcui](https://github.com/fullstorydev/grpcui)

![](https://tva1.sinaimg.cn/large/008i3skNly1gwv5txp2evj30qo0erjrx.jpg)

首先看 `BloomRPC` 页面美观，功能也很完善；但却有个非常难受的地方，那就是不支持 `int64`, 会有精度问题。

![](https://tva1.sinaimg.cn/large/008i3skNly1gwv6c1oy8gj31360u0acg.jpg)
> 直接将请求的 `int64` 返回回来。

```go
func (o *Order) Create(ctx context.Context, in *v1.OrderApiCreate) (*v1.Order, error) {
	fmt.Println(in.OrderId)
	return &v1.Order{
		OrderId: in.OrderId,
		Reason:  nil,
	}, nil
}
```
会发现服务端收到的数据精度已经丢失了。

这个在我们大量使用 `int64` 的业务中非常难受，大部分接口都没法用了。

---
![](https://tva1.sinaimg.cn/large/008i3skNly1gwv6w1nvpkj310q0iewfr.jpg)
`grpcui` 是我再使用了 `BloomRPC` 一段时间之后才发现的工具，功能也比较完善；


# 核心功能

## 安装

# 后续计划
