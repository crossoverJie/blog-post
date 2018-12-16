---
title: 一次生产 CPU 100% 排查优化实践
date: 2018/12/17 08:15:36 
categories: 
- 问题排查
- Java 进阶
tags: 
- Java
- Thread
- concurrent
- JVM
- disruptor
---

![](https://ws3.sinaimg.cn/large/006tNbRwly1fy67gauqxyj31eg0u0gun.jpg)

# 前言

到了年底果然都不太平，最近又收到了运维报警：表示有些服务器负载非常高，让我们定位问题。

还真是想什么来什么，前些天还故意把某些服务器的负载提高（[没错，老板让我写个 BUG！](https://crossoverjie.top/2018/12/12/java-senior/java-memary-allocation/)），不过还好是不同的环境没有互相影响。

# 定位问题

拿到问题首先去服务器上看了看，发现运行的只有我们的 Java 应用。于是先用 `ps` 命令拿到了应用的 PID。

接着使用 `ps -Hp pid` 将这个进程的线程显示出来。输入大写的 P 可以将线程按照 CPU 使用比例排序，于是得到以下结果。

![](https://ws3.sinaimg.cn/large/006tNbRwly1fy7z1kg8s3j30s40ncn9w.jpg)

果然某些线程的 CPU 使用率非常高。


为了方便定位问题我立马使用 `jstack pid > pid.log` 将线程栈 `dump` 到日志文件中。

我在上面 100% 的线程中随机选了一个 `pid=194283` 转换为 16 进制（2f6eb）后在线程快照中查询：

> 因为线程快照中线程 ID 都是16进制存放。

![](https://ws1.sinaimg.cn/large/006tNbRwly1fy7z7vtcruj30q5056tar.jpg)

发现这是 `Disruptor` 的一个堆栈，前段时间正好解决过一个由于 Disruptor 队列引起的一次 OOM：[强如 Disruptor 也发生内存溢出？](https://crossoverjie.top/2018/08/29/java-senior/OOM-Disruptor/)

没想到又来一处。

为了更加直观的查看线程的状态，我将快照信息上传到专门分析的平台上。

[http://fastthread.io/](http://fastthread.io/)

![](https://ws2.sinaimg.cn/large/006tNbRwly1fy7zciqp2ij311q0q5jzl.jpg)

其中有一项菜单展示了所有消耗 CPU 的线程，我查看了所有的发现几乎都是和上面的堆栈一样。

也就是都是 `Disruptor` 队列的堆栈，同时都在执行 `java.lang.Thread.yield`.


# 解决问题

# 总结


**你的点赞与分享是对我最大的支持**