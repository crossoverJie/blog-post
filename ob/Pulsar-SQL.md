---
title: 使用 SQL 的方式查询消息队列数据以及踩坑指南
date: 2023/08/30 09:31:53
categories: 
- Pulsar
tags: 
- SQL
---
![Pulsar-sql.png](https://s2.loli.net/2023/08/30/3iz9yqfuSCn18xk.png)
# 背景

为了让业务团队可以更好的跟踪自己消息的生产和消费状态，需要一个类似于表格视图的消息列表，用户可以直观的看到发送的消息；同时点击详情后也能查到消息的整个轨迹。

>  消息列表
![20230802234405.png](https://s2.loli.net/2023/08/02/l9uvSnqAOxfPer7.png)

<!--more-->

> 点击详情后查看轨迹
![20230802234058.png](https://s2.loli.net/2023/08/02/H7pjinzQ5EWR2tF.png)


# 原理介绍


由于 `Pulsar` 并没有关系型数据库中表的概念，所有的数据都是存储在 `Bookkeeper` 中，为了模拟使用 SQL 查询的效果 Pulsar 提供了 `Presto` (现在已经更名为 `Trino`)的插件。
> Trino 是一个分布式的 SQL 查询引擎，它也提供了插件能力，如果我们想通过 SQL 从自定义数据源查询数据时，基于它的 SPI 编写一个插件是很方便的。

这样便可以类似于查询数据库一样查询 `Pulsar` 数据：
![image.png](https://s2.loli.net/2023/08/30/1YEtorbwaZAXylL.png)

---
![image.png](https://s2.loli.net/2023/08/30/u6gc3YxvH94ZDPG.png)
Pulsar 插件的运行流程如上图所示：
- 启动的时候通过 `Pulsar-Admin` 接口获取一些元数据，比如 Scheme，topic 分区信息等。
- 然后会创建一个只读的 Bookkeeper 客户端，用于获取数据。
- 之后根据 SQL 条件过滤数据即可。

相关代码：
![image.png](https://s2.loli.net/2023/08/30/vr7ED6BYgOsoqxz.png)
![image.png](https://s2.loli.net/2023/08/30/Np2XD7T9cJAwxQC.png)

# 使用 Pulsar-SQL
![image.png](https://s2.loli.net/2023/08/30/UBqocsjFvC2yXEe.png)

使用起来也很简单，官方提供了两个命令：
- sql-worker: 会启动一个 trino 服务端同时运行了 Pulsar 插件
- sql: 就是一个 SQL 命令行终端。


## 遇到的问题

自己在本地运行的时候自然是没问题，可是一旦想在生产运行，同时如果你的 `Pulsar` 集群是运行再 `k8s` 环境中时就会碰到一些问题。

### 无法使用现有 Trino 集群

首先第一个问题是如果生产环境已经有了一个 `Trino` 集群想要复用的时候就会碰到问题，常规流程是将 `Pulsar` 的插件复制到 `Trino` 的 `Plugin` 目录，然后重启 `Trino` 后就能使用该插件。

当然社区也是支持这么做的：
![image.png](https://s2.loli.net/2023/08/30/RqtIvwy5HNsr27M.png)
但是当我将 Pulsar-plugin 复制到 Trino 中运行的时候却失败了，整体的流程可以参考这个 issue：
[https://github.com/apache/pulsar/discussions/20941](https://github.com/apache/pulsar/discussions/20941)

简单来说 `Trino` 的官方镜像和 `pulsar-plugin` 并不能兼容，这个问题直接影响到我们是否可以在生产环境使用它。

但是手动编译出来的 `Trino` 服务和插件是兼容的，可以直接运行。

![image.png](https://s2.loli.net/2023/08/30/MswBlVXi12DICr9.png)

因此我只能在本地编译出 Trino 服务端和 `pulsar-plugin` 然后打包成一个镜像来运行了，当然这样的坏处就是无法利用到我们现有的 `Trino` 集群，又得重新部署一个了。

![image.png](https://s2.loli.net/2023/08/30/vG83bleTf1EcCPp.png)

流程也比较麻烦：
- 首先是本地编译 `Pulsar-SQL` 模块
- 将生成物复制到当前目录
- 执行 `make docker` 打出 docker 镜像并上传到私服
- 再执行 `kubectl` 将 trino 部署到 `k8s` 环境中

整个流程做下来加上和社区的沟通，更加确定这个功能应该是很少有人在生产环境使用的，毕竟第一个坑就很麻烦，更别提后续的问题了😂。

### Presto 插件不支持 AuthToken

第二个问题也是个深坑，当我把 Trino 部署好查询数据的时候直接抛了一个调用 `pulsar-admin`  接口连接超时的异常。

结果排查了半天发现原来是 `pulsar-plugin` 里没有提供 `JWT` 的验证方式，而我们的 Pulsar 集群恰好是打开了 `JWT` 验证的。

为此我只能先在本地修复了这个问题，同时也提交了 PR，预计会在下一个大版本合并吧：
[https://github.com/apache/pulsar/pull/20860](https://github.com/apache/pulsar/pull/20860)

### 新创建的 topic 查询失败
第二个问题是当查询一个新创建的 topic 时，客户端会直接 block，相关的复现流程在这里：
[https://github.com/apache/pulsar/issues/20910](https://github.com/apache/pulsar/issues/20910)

![image.png](https://s2.loli.net/2023/08/30/nYestcQqRax1NVv.png)

这个问题还好，不是很致命，是我在本地测试的时候无意间发现的。

本地我已经修复了，后面也提交了一个 PR，目前还在讨论中：
[https://github.com/apache/pulsar/pull/20911](https://github.com/apache/pulsar/pull/20911)


### 查询消息会丢失最后一条

这个问题也不是很严重，数据量少的时候会发现，就是在指定了消息发送时间的查询条件时，最后一条消息会被过滤掉，相关 issue 在这里：
[https://github.com/apache/pulsar/issues/20919](https://github.com/apache/pulsar/issues/20919)
![image.png](https://s2.loli.net/2023/08/30/MPamvyduxrTZRkY.png)
这个我只是定位到了原因，但不太清楚 为什么要这么做(-1)，影响也不是很大，就放在这里搁置了。
### Schema 不兼容

最后发现的一个问题是我们线上某些 topic 查询数据的时候会抛出 `Not a record: "string"`的异常，但只是部分 topic，也排查了很久，整个源码中没有任何一个地方有这个异常。

[https://github.com/apache/pulsar/issues/20945](![](https://s2.loli.net/2023/08/30/UBl6OPGzASnfqT2.png))


![image.png](https://s2.loli.net/2023/08/30/UBl6OPGzASnfqT2.png)


根本原因是生产者生成的 schema 有问题，类型已经是 JSON 了，但是 schema 却是 string，这样导致 `pulsar-plugin`  在反序列化 schema 的时候抛出了异常，由于是 pb 反序列化抛出的异常，所以源码中都搜索不到。

> 没有问题的 topic 使用了正确的 schema

后续我也在本地修复了这个问题，当抛出异常后就将 schema 降级为基本类型进行解析。
![image.png](https://s2.loli.net/2023/08/30/XZfWG2EYHpj5QJb.png)

不过本质问题还是客户端使用有误，如果对 `schema` 理解不准确的话还是建议使用 `byte[]` 吧，这样至少兼容性不会有问题。
相关 PR：
[https://github.com/apache/pulsar/pull/20955](https://github.com/apache/pulsar/pull/20955)

# 总结
`Pulsar-SQL` 是一个非常有用的功能，只是我们使用过程中确实发现了一些问题，大部分都已经修复了；
希望对后续使用该功能的朋友有所帮助。
#Pulsar 