---
title: 线程池没你想的那么简单（续）
date: 2019/06/05 08:10:00
categories: 
- 并发
tags: 
- concurrent
- ThreadPool
---

![](http://ww2.sinaimg.cn/large/006tNc79ly1g3pl5jscpsj31gw0u0ju4.jpg)

# 前言

前段时间写过一篇[《线程池没你想的那么简单》](https://crossoverjie.top/2019/05/20/concurrent/threadpool-01/)，和大家一起撸了一个基本的线程池，具备：

- 线程池基本调度功能。
- 线程池自动扩容缩容。
- 关闭线程池。

这些功能，最后也留下了三个待实现的 `features` 。

- 执行带有返回值的线程。
- 异常处理怎么办？
- 所有任务执行完怎么通知我？


这次就实现这三个特性来看看 `j.u.c` 中的线程池是如何实现的。

# 任务完成后的通知

# 带有返回值的线程

# 异常处理


# 总结

线程池关闭的常规做法

统计完成的任务数量

本文所有源码：

[https://github.com/crossoverJie/JCSprout/blob/master/src/main/java/com/crossoverjie/concurrent/CustomThreadPool.java](https://github.com/crossoverJie/JCSprout/blob/master/src/main/java/com/crossoverjie/concurrent/CustomThreadPool.java)

**你的点赞与分享是对我最大的支持**
