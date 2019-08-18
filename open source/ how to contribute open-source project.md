---
title: 如何参与一个顶级开源项目
date: 2019/08/19 08:13:00
categories: 
- open source
tags: 
- dubbo
- thread
- 多线程
---

![](http://ww4.sinaimg.cn/large/006tNc79ly1g642hvvv5yj31de0rs76o.jpg)

# 前言

最近个人事情比较多（搬家、换工作、短暂休息）所以一直也没有顾得上博客更新，恰好最近收到一封邮件提醒了我。

![](http://ww1.sinaimg.cn/large/006tNc79ly1g642lwk57mj30k50j1adp.jpg)

是时候写一篇文章来聊聊参与开源项目的事（最近也确实进入了笔荒期）。

ps:第一次收到这样的中秋节礼物，加上 `Dubbo` 社区的活跃及阿里的重视度，还在做 `PRC` 或微服务技术选型的朋友可以考虑 `Dubbo`。


# 参与开源

现在具体来聊聊参与开源的事；

日常开发中几乎所有的开发者都会享受到开源项目所带来的便利甚至是收入，早在十几年前甚至几年前开源活动一直都是有国外开发者主导，受限于环境这也是没办法的事。

但这几年国内互联网公司逐渐国际化扩大影响力也很大程度的提高了我们的开发水平，以 `BAT` 为首出现了许多优秀的开源项目。

现在甚至参与开源项目还能另辟蹊径的拿到大厂 `offer`，所以不少朋友其实都想参与其中，可能这事给人的第一感觉就不太容易，所以现在还卡在第一步。


## 具体步骤

以下是以我个人经验总结的几大步骤：

- 发现问题或自荐 `feature` 。
- fork 源码。
- 本地开发、自测。
- 发起 `pull request` 。
- 等待社区 `Code Review` 。
- 跟进社区意见调整代码。
- 审核通过，合并进 `master` 分支，完成本次贡献。


下面我会结合最近一次参与 `Dubbo` 的流程来具体聊聊。


### 发现问题或自荐 feature

首先第一步自然要搞清楚自己本次贡献的内容是什么？通常都是解决某个问题或者是提交一个新的 `feature` ；前者相对起来更加容易一些。

当然这个问题可以是自己使用过程中发现的，也可以是 Issues 列表中待解决的问题。


以本次为例，就是我在使用过程中所发现的问题，也提交了相关 [Issue](https://github.com/apache/dubbo/issues/4556) 并写了一篇文章记录并解决了该问题：[What？一个 Dubbo 服务启动要两个小时！](https://crossoverjie.top/2019/07/05/troubleshoot/dubbo-start-slow/)

值得注意的是在提交 Issue 之前最好是先在 Issue 列表中通过关键字检索下是否已经有相关问题，避免重复。

同时提交之后也许社区会进行跟进，被打上 `invalid` 标签认为不是问题，或者是使用姿势不对也是有可能的。

### fork 源码，本地开发

当确定这是一个待修复的问题时就可以着手开发了。

首先第一步自然是将源码拷贝一份到自己仓库中。

![](http://ww3.sinaimg.cn/large/006tNc79ly1g6486zo8umj30se03wjs1.jpg)

接着只需要 clone 自己仓库中的源码到本地进行开发了。

先回顾下我遇到的这个问题。

![](http://ww2.sinaimg.cn/large/006tNc79ly1g648ajiq2fj30rt0q27ha.jpg)

简单来说就是启动 `Dubbo` 服务非常缓慢，经过定位是 `main` 线程阻塞在了获取本机 ip 处。

所以当时我提出的方案是：在获取本机 ip 时加上超时时间，一旦超时便抛出异常或者是再次重试，但起码得有日志方便用户定位问题。


主要主线程会一直阻塞在此处 `InetAddress.getLocalHost().getHostAddress()`，但又需要知道它阻塞了多久才好判断是否超时。

所以只能再额外开启一个线程，定时去检测 `main` 线程是否已经完成任务了。


![](http://ww4.sinaimg.cn/large/006tNc79ly1g649tp7um7j31f80llaei.jpg)
![](http://ww1.sinaimg.cn/large/006tNc79ly1g649wtwsykj31fo0i6jty.jpg)

这次的重点不是讨论这里具体的技术细节，所以简单说下步骤吧：

- 额为声明了大小为 1 的线程池。
- 再声明了一个 `volatile` 标志用于判断主线程是否有完成任务。
- 声明了一个 condition 用于新线程做等待。
- 最后只需要运行这个线程用于判断这个标志即可。

### 如何自测

开发完成后下一步就是自测，由于这类项目是作为一个基础包依赖于额外的项目才能运行的，所以通常我们还得新建一个项目来配合做全流程测试（不算基本的单测）。

这里我觉得还是有几个小技巧需要注意的。

第一个是版本号；因为在本地测试，所以需要使用 `mvn clean install` 将包安装到本地才能在其他项目中依赖进去进行测试。

但由于我们从官方拉出来的代码版本都已经发布到了 maven 中央仓库中（不管是 release 还是 snapshot），所以我们本地仓库中肯定已经存在这几个版本的 jar 包。

一旦我们执行 `mvn clean install` 将自己新增的代码安装到本地时，大概率是会出问题的（也可能是我姿势不对），这样就会导致新建的项目中依赖不了自己新增的代码。

所以我通常的做法是修改版本号，这个版本号是从来没有被官方发布到中央仓库中的，可以确保自己新增的代码会以一个版本安装到本地，这样我们再依赖这个版本进行测试即可。

> 不过再提交时得注意不要把这个版本号提交上去了。


### 发起 pull request

自测完成后便可发起 `pull request` 了，不要大意，这里还得有一个地方需要注意，那就是代码换行符的问题。

一旦换行符与源仓库的不一致时，`git` 会认为这次修改是删除后重来的，这样会给 `code review` 带来巨大的麻烦。

![](http://ww1.sinaimg.cn/large/006tNc79ly1g64amfnceqj31g40brtcz.jpg)

就像这样，明明我改动的行数并不多，但 `git` 确认为你是推翻了重来，导致审核起来根本不知道你改了哪些地方。

最简单的方法就是设置自己 `git` 的全局配置，可以参考[这里](http://kuanghy.github.io/2017/03/19/git-lf-or-crlf)。

```shell
# 提交时转换为LF，检出时转换为CRLF
git config --global core.autocrlf true

# 提交时转换为LF，检出时不转换
git config --global core.autocrlf input

# 提交检出均不转换
git config --global core.autocrlf false
```

![](http://ww3.sinaimg.cn/large/006tNc79ly1g64aouribqj30rg086taa.jpg)

都没问题后便可点击这里发起 pull request，后面按照引导走即可。

当然各个项目之间还会有自己定制的贡献流程，最好就是查看官方的贡献指南。

[http://dubbo.apache.org/en-us/docs/developers/contributor-guide/new-contributor-guide_dev.html](http://dubbo.apache.org/en-us/docs/developers/contributor-guide/new-contributor-guide_dev.html)

### Code Review

`pr` 发起后便可等待社区审核了。

在这过程中要充分和社区进行交流，有可能你的方案和社区的想法并不一致。

比如像我这次：

![](http://ww3.sinaimg.cn/large/006tNc79ly1g64b5j0dqvj30la0m3n0x.jpg)
![](http://ww4.sinaimg.cn/large/006tNc79ly1g64b6roglyj30kz0fugnf.jpg)
最终通过沟通加上自己后面的思考觉得还是社区的方案更加轻便合理一些，达成一致之后社区便将这次 pr 合并进 master 中。

其实整个过程我觉得最有意义的便是 `code review` 的过程，所有人都可以参与其中头脑风暴，其中也不乏需要大牛，不知不觉便能学到不少东西。

## 类似案例

虽然我之前的方案没有被采纳，但类似的用法（一个线程监控其他线程）还是不少，正好在 `Dubbo` 中也有用到。


便是其中核心的服务调用，默认情况下对使用者来说这看起来是一个同步调用，也就是说消费方会等待 PRC 执行完毕后才会执行后续逻辑。

但其实在底层这就是一个 `TCP` 网络包的发送过程，**本身就是异步的**。

只是 `Dubbo` 默认情况下做了异步转同步。

![](http://ww1.sinaimg.cn/large/006tNc79ly1g64bqqen92j31bj0hejwg.jpg)

如图中的红框部分，`Dubbo` 自身调用了 `get()` 方法用于同步获取服务提供者的返回结果。

![](http://ww4.sinaimg.cn/large/006tNc79ly1g64bs9zoxsj30sq0eomzt.jpg)

逻辑其实也挺简单，和我上文的方案类似，只是这里的 `isDone()` 函数返回的是是否已经拿到了服务提供者的返回值而已。


# 总结

本次总结了参与开源的具体步骤，其实也挺简单；就如官方所说哪怕是提个 Issue，修改一个错别字都算是参与，所以不要想的太难。

最后还简单分析了 Dubbo 调用过程中的异步转同步的过程，掌握这些操作对自己平时开发也是很有帮助的。

**你的点赞与分享是对我最大的支持**
