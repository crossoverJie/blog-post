---
title: 如何给开源项目发起提案
date: 2023/12/21 11:09:52
categories:
  - OB
tags:
- Pulsar
---

# 背景

前段时间在使用 Pulsar 的 admin API 时，发现其中的一个接口响应非常慢：
```java
admin.topics().getPartitionedStats(topic);
```
使用 curl 拿到的响应结果非常大，同时也非常耗时：
![image.png](https://s2.loli.net/2023/12/21/dMsUq1eFNz9IoYC.png)

具体的 issue 在这里：[https://github.com/apache/pulsar/issues/21200](https://github.com/apache/pulsar/issues/21200)
<!--more-->

后面经过分析，是因为某些 topic 的生产者和消费者非常多，导致这个查询 topic 统计的接口数据量非常大。
![image.png](https://s2.loli.net/2023/12/21/CIwr3qivSQyPnEx.png)

但在我这个场景其实是不需要这些生产者和消费者信息的，现在就导致这个 topic 无法查看状态，所以就建议新增两个参数可以过滤这两个字段。
# 流程
因为涉及到新增 API 了，所以社区维护者就建议我起草一个提案试试：
![image.png](https://s2.loli.net/2023/12/21/qyDmVHRsFBoewiQ.png)

## 什么时候需要提案

此时就涉及到什么情况下需要给社区发起一个提案的问题了。
![image.png](https://s2.loli.net/2023/12/21/VH8NqwgWcROLXhP.png)
在官方的提案指南中有着详细的说明，简单来说就是：
- 对任何模块新增了 API、或者是重大改动的新特性、监控指标、配置参数时都需要发起提案
- 对应的如果只是对现有 bug 的修复、文档等一些可控的变更时，是不需要发起提案的，直接提交 PR 即可。

### 提案步骤

#### 起草
首先第一步就是根据官方模版起草一个提案：
重点描述背景、目的、详细设计等。
![image.png](https://s2.loli.net/2023/12/21/eT7xQEk3li6Rdyp.png)
并发起一个 PR，如果不确定怎么写的话可以参考已经合并了的提案。

#### 邮件讨论
之后则是将这个 PR 发送到开发组邮箱中，让社区成员参与讨论。

![image.png](https://s2.loli.net/2023/12/21/cAlZLYGEOqiyMoh.png)
这一步可能会比较耗时，提案内容可能会被反复修改。

发起提案的一个重要目的是可以让社区成员进行讨论，评估是否需要这个提案或者是否
有其他解决方法。
#### 发起投票
经过讨论，如果提案获得通过后就可以发起投票了，至少需要有三个 binding 通过的投票后这个提案就通过了。

> 虽然任何人都可以参与投票，但社区只会考虑 PMC 的投票建议；投票的时效性也只有 48h。

![image.png](https://s2.loli.net/2023/12/21/SiHbGyRzX1kBTjt.png)

48 小时候便可以发一个投票结果的邮件，如果达到通过条件便可以通知参与投票的 PMC 合并这个 PR 了。
![image.png](https://s2.loli.net/2023/12/21/r7tm8pyVER6QvCf.png)


#### 实现提案
之后就是没啥好说的实现过程，因为通常我们是需要在提案里详细描述实现过程以及涉及到修改的地方。

# 总结

只要提案被 review 通过后实现起来就非常简单了，跟着提案里的流程实现就好了。

> 这点非常类似于我们在企业中对某个业务做技术方案，如果大家都按照类似的流程严格审核方案，那实现起来是非常快的，而且可以尽量的减少事后扯皮。

所以最后我的实现 PR 提交之后，都没有任何的修改意见，直接就合并了；也大大降低了审核人员的负担，提高整体效率。

以上就是我第一次参与 Pulsar 社区的提案过程，我猜测其他社区的流程也是大差不差；其中重点就是异步沟通；大家都认可之后真的会比实时通信的效率高很多。

具体的提案细节可以阅读官方指南 [https://github.com/apache/pulsar/blob/master/pip/README.md](https://github.com/apache/pulsar/blob/master/pip/README.md)

#Blog #Pulsar 