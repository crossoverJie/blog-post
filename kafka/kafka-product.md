---
title: 从源码分析如何优雅的向 Kafka 发送消息
date: 2018/10/10 00:01:14       
categories: 
- Kafka
- Java 进阶
tags: 
- Kafka
---

![](https://ws2.sinaimg.cn/large/006tNbRwgy1fw2g4pw7ooj31kw11xwjh.jpg)

# 前言

在上文 [设计一个百万级的消息推送系统](https://crossoverjie.top/2018/09/25/netty/million-sms-push/) 中提到消息使用的是 `Kafka` 作为中间。

其中有朋友咨询在大量消息的情况下 `Kakfa` 是如何保证消息的高效及一致性呢？

正好以这个问题结合 `Kakfa` 的源码讨论下如何正确、高效的发送消息。

> 内容较多，对源码感兴趣的朋友请系好安全带(源码基于 `v0.10.0.0` 版本分析)。

<!--more-->

# 简单的消息发送

在分析之前先看看一个简单的消息发送是怎么样的。

> 以下代码基于 SpringBoot 构建。

首先创建一个 `org.apache.kafka.clients.producer.Producer` 的 bean。

![](https://ws1.sinaimg.cn/large/006tNbRwgy1fw2hc2t8oij30n507g0u6.jpg)

首先关注 `bootstrap.servers`，它是必填参数。指的是 Kafka 集群中的 broker 地址，例如 `127.0.0.1:9094`。

> 其余几个参数暂时不做讨论，后文会有详细介绍。

接着注入这个 bean 即可调用它的发送函数发送消息。

![](https://ws4.sinaimg.cn/large/006tNbRwgy1fw2he841x7j30ou054751.jpg)

这里我给某一个 Topic 发送了 10W 条数据，运行程序消息正常发送。

但这仅仅只是做到了消息发送，对消息是否成功发送完全没管，等于是纯`异步`的方式。

## 同步

那么我想知道消息到底发送成功没有改怎么办呢？

其实 Producer 的 API 已经帮我们考虑到了，发送之后只需要调用它的 get() 方法即可同步获取发送结果。

![](https://ws4.sinaimg.cn/large/006tNbRwly1fw3fsyrkpbj3103065mya.jpg)

发送结果：

![](https://ws2.sinaimg.cn/large/006tNbRwly1fw3ftq0w5lj312g053770.jpg)

这样的发送效率其实是比较低下的，每次都需要同步等待消息发送的结果。 

## 异步

为此我们应当采取异步的方式发送，其实 `send()` 方法默认则是异步的，只要不手动调用  `get()` 方法。

但这样就没法获知发送结果。

所以查看 `send()` 的 API 可以发现还有一个参数。

```java
Future<RecordMetadata> send(ProducerRecord<K, V> producer, Callback callback);
```

`Callback` 其实是一个回调接口，在消息发送完成之后可以回调我们自定义的实现。

![](https://ws3.sinaimg.cn/large/006tNbRwly1fw3g4hce6aj30zv0b0dhp.jpg)

执行之后的结果：

![](https://ws2.sinaimg.cn/large/006tNbRwly1fw3g54ne3oj31do06t0wl.jpg)

同样的也能获取结果，同时发现回调的线程并不是上文同步时的`主线程`，这样也能证明是异步回调的。

同时回调的时候会传递两个参数：

- `RecordMetadata` 和上文一致的消息发送成功后的元数据。
- `Exception` 消息发送过程中的异常信息。

但是这两个参数并不会同时都有数据，只有发送失败才会有异常信息，同时发送元数据为空。

所以正确的写法应当是：

![](https://ws4.sinaimg.cn/large/006tNbRwly1fw3g9fst9kj30zy07jab0.jpg)

> 至于为什么会只有参数一个有值，在下文的源码分析中会一一解释。


# 源码分析

现在只掌握了基本的消息发送，想要深刻的理解发送中的一些参数配置还是得源码说了算。

首先还是来谈谈消息发送时的整个流程是怎么样的，`Kafka` 并不是简单的把消息通过网络发送到了 `broker` 中，在 Java 内部还是经过了许多优化和设计。
I
## 发送流程

为了直观的了解发送的流程，简单的画了几个在发送过程中重要的步骤。

![](https://ws3.sinaimg.cn/large/006tNbRwly1fw3j5x05izj30a40btmxt.jpg)

- 初始化以及真正发送消息的 `kafka-producer-network-thread` IO 线程。
- 将消息序列化。
- 得到需要发送的分区。
- 写入内部的一个缓存区中。
- 初始化的 IO 线程不断的消费这个缓存来发送消息。

## 步骤解析

接下来详解每个步骤。

### 初始化


![](https://ws1.sinaimg.cn/large/006tNbRwly1fw3jc9hvwbj30rc0273yn.jpg)

调用该构造方法进行初始化时，不止是简单的将基本数据写入 `KafkaProducer`。比较麻烦的是初始化 `Sender` 线程进行缓冲区消费。

初始化 IO 线程处：

![](https://ws2.sinaimg.cn/large/006tNbRwly1fw3jh4xtt2j31fo02pgms.jpg)

可以看到 Sender 线程有需要成员变量，比如：

```
acks,retries,requestTimeout
```

等，这些参数会在后文分析。


### 序列化消息


### 路由分区

### 写入内部缓存

### 消费缓存

# Producer 参数解析

## acks

## bitchsize

## retries

# 高效的发送方式

# 关闭 Producer

**欢迎关注公众号一起交流：**
