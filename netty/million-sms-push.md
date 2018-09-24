---
title: 设计一个百万级的消息推送系统
date: 2018/09/25 00:01:14       
categories: 
- Netty
tags: 
- 推送
- 路由策略
- 注册发现
- Redis
- Zookeeper
- Kafka
---

![business-communication-computer-261706.jpg](https://i.loli.net/2018/09/23/5ba7ae180e8eb.jpg)

# 前言

首先迟到的祝大家中秋快乐。

最近一周多没有更新了。其实我一直想憋一个大招，分享一些大家可能会感兴趣的干货。

鉴于最近我个人的工作内容也就这点还拿得出手，于是利用这三天小长假憋了一个出来（其实是玩了两天🤣）。


---

先简单说下本次的主题，由于我最近做的是物联网相关的开发工作，其中就不免会遇到和设备的交互。

最主要的工作就是要有一个系统来支持设备的接入、向设备推送消息；同时还得满足大量设备接入的需求。

所以本次分享的内容不但可以满足物联网领域同时还支持以下场景：

- 基于 `WEB` 的聊天系统（点对点、群聊）。
- `WEB` 应用中需求服务端推送的场景。
- 基于 SDK 的消息推送平台。

# 技术选型

要满足大量的连接数、同时支持双全工通信，并且性能也得有保障。

在 Java 技术栈中进行选型首先自然是排除掉了传统 `IO`，性能低下。

那就只有选 NIO 了，在这个层面其实选择也不多，考虑到社区、资料维护等方面最终选择了 Netty。

最终的架构图如下：

![](https://ws1.sinaimg.cn/mw690/72fbb941gy1fvjz1teappj20rg0humy1.jpg)


现在看着蒙没关系，下文一一介绍。

# 协议解析

既然是一个消息系统，那自然得和客户端定义好双方的协议格式。

常见和简单的是 HTTP 协议，但我们的需求中有一项需要是双全工的交互方式，同时 HTTP 更多的是服务于浏览器。我们需要的是一个更加精简的协议，减少许多不必要的数据传输。

因此我觉得最好是在满足业务需求的情况下定制自己的私有协议，在我这个场景下其实有标准的物联网协议。

如果是其他场景可以借鉴现在流行的 `RPC` 框架定制私有协议，使得双方通信更加高效。

不过根据这段时间的经验来看，不管是哪种方式都得在协议中预留安全相关的位置。

协议相关的内容就不过讨论了，更多介绍具体的应用。

# 简单实现

首先考虑如何实现功能，再来思考百万连接的情况。

## 注册鉴权

再做真正的消息上、下行之前首先要考虑的就是鉴权问题。

就像你使用微信一样，第一步怎么也得是登录吧，不能无论是谁都可以直接连接到平台。

所以第一步得是注册才行。

如上面架构图中的 `注册/鉴权` 模块。通常来说都需要客户端通过 `HTTP` 请求传递一个唯一标识，后台鉴权通过之后会响应一个 `token`，并将这个 `token` 和客户端的关系维护到 `Redis` 或者是 DB 中。

客户端将这个 token 也保存到本地，今后的每一次请求都得带上这个 token。一旦这个 token 过期，客户端需要再次请求获取 token。

鉴权通过之后客户端会直接通过`TCP 长连接`到图中的 `push-server` 模块。

这个模块就是真正处理消息的上、下行。

## 保存通道关系

在连接接入之后，真正处理业务之前需要将当前的客户端和 Channel 的关系维护起来。

假设客户端的唯一标识是手机号码，那就需要把手机和当前的 Channel 维护到一个 Map 中。

这点和之前 [Netty(一) SpringBoot 整合长连接心跳机制](https://crossoverjie.top/2018/05/24/netty/Netty(1)TCP-Heartbeat/#%E6%9C%8D%E5%8A%A1%E7%AB%AF%E5%BF%83%E8%B7%B3) 类似。

![](https://ws2.sinaimg.cn/large/006tNbRwgy1fvkj6oe4rej30k104c0tg.jpg)

同时为了可以通过 Channel 获取到客户端唯一标识（手机号码），还需要在 Channel 中设置对应的熟悉：

```java
    public static void putClientId(Channel channel, String clientId) {
        channel.attr(CLIENT_ID).set(clientId);
    }
```

获取时手机号码时：

```java
    public static String getClientId(Channel channel) {
        return (String)getAttribute(channel, CLIENT_ID);
    }
```

这样当我们客户端下线的时便可以记录相关日志：

```java
String telNo = NettyAttrUtil.getClientId(ctx.channel());
NettySocketHolder.remove(telNo);
log.info("客户端下线，TelNo=" +  telNo);
```

> 这里有一点需要注意：存放客户端与 Channel 关系的 Map 最好是预设好大小（避免经常扩容），因为它将是使用最为频繁同时也是占用内存最大的一个对象。

## 消息上行

接下来则是真正的业务数据上传，通常来说第一步是需要判断上传消息输入什么业务类型。

在聊天场景中，有可能上传的是文本、图片、视频等内容。

所以我们的进行区分，来做不同的处理；这就和客户端协商的协议有关了。

- 可以利用消息头中的某个字段进行区分。
- 更简单的就是一个 `JSON` 消息，拿出一个字段用于区分不同消息。

不管是哪种只有可以区分出来即可。

### 消息解析与业务解耦

消息可以解析之后便是处理业务，比如可以是写入数据库、调用其他接口等。

我们都知道在 Netty 中处理消息一般是在 `channelRead()` 方法中。

![](https://ws2.sinaimg.cn/large/006tNbRwgy1fvkkawymbkj30o6027mxf.jpg)

在这里可以解析消息，区分类型。

但如果我们的业务逻辑也写在里面，那这里的内容将是巨多无比。

甚至我们分为好几个开发来处理不同的业务，这样将会出现许多冲突、难以维护等问题。

所以非常有必要将消息解析与业务处理完全分离开来。


> 这时面向接口编程就发挥作用了。

这里的核心代码和 [「造个轮子」——cicada(轻量级 WEB 框架)](https://crossoverjie.top/2018/09/03/wheel/cicada1/#%E9%85%8D%E7%BD%AE%E4%B8%9A%E5%8A%A1-Action) 是一致的。

都是先定义一个接口用于处理业务逻辑，然后在解析消息之后通过反射创建具体的对象执行其中的`处理函数`即可。

这样不同的业务、不同的开发人员只需要实现这个接口同时实现自己的业务逻辑即可。

伪代码如下：

![](https://ws1.sinaimg.cn/large/006tNbRwgy1fvkkhd8961j30n602kglr.jpg)

![](https://ws2.sinaimg.cn/large/006tNbRwgy1fvkkhwsgkqj30nh0m0gpt.jpg)

想要了解 cicada 的具体实现请点击这里：

[https://github.com/TogetherOS/cicada](https://github.com/TogetherOS/cicada)


## 消息下行

有了上行自然也有下行。比如在聊天的场景中，有两个客户端连上了 `push-server`,他们直接需要点对点通信。

这时的流程是：

- A 将消息发送给服务器。
- 服务器收到消息之后，得知消息是要发送给 B，需要在内存中找到 B 的 Channel。
- 通过 B 的 Channel 将 A 的消息转发下去。

这就是一个下行的流程。

甚至管理员需要给所有在线用户发送系统通知也是类似：

遍历保存通道关系的 Map，挨个发送消息即可。这也是之前需要存放到 Map 中的主要原因。

伪代码如下：

![](https://ws3.sinaimg.cn/large/006tNbRwgy1fvkkpefci7j30w408h768.jpg)

具体可以参考：

[https://github.com/crossoverJie/netty-action/](https://github.com/crossoverJie/netty-action/)



# 分布式方案

## 注册发现

## 有状态连接

### 路由策略

## 消息流转

## 推送路由


# 分布式问题

## 应用监控

## 日志处理

# 总结


**欢迎关注公众号一起交流：**
