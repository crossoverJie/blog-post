---
title: 图床失效了？也许你应该试试这个工具
date: 2019/05/08 01:50:36 
categories: 
- 工具
tags: 
- Java
- 策略模式
---

![](https://i.loli.net/2019/05/08/5cd1cbf638ec7.png)

# 前言

经过几个小伙伴的提醒，发现个人博客中的许多图片都裂了无法访问；原因就不多说，既然出现问题就得要解决。

<!--more-->

![](https://i.loli.net/2019/05/08/5cd1c6bfd8ff2.jpg)

![](https://i.loli.net/2019/05/08/5cd1cc4598afb.jpg)

原本我的处理方式非常简单粗暴：找到原有的图片重新下载下来上传到新的可用图床再把图片地址替换。

这样搞了一两篇之后我就绝望了。。。

之前为了代码能在公众号里也有好的阅读体验，所以能截图的我绝不贴代码，导致一篇文章多的得有十几张图片。


好在哪位大佬说过“以人肉XX为耻”，这种重复劳动力完全可自动化；于是便有了本次的这个工具。

它可以一行命令把你所有 `Markdown` 写的内容中的图片全部替换为新的图床。

运行效果如下：

![](https://i.loli.net/2019/05/08/5cd1cc7612c25.gif)

![](https://i.loli.net/2019/05/08/5cd1cd6062d2a.png)



# 使用

可以直接在这个地址下载 jar 包运行：https://github.com/crossoverJie/blog.toolbox/releases/download/v0.0.1/blog.toolbox-0.0.1-SNAPSHOT.jar

当然也可以下载源码编译运行：

```shell
git clone https://github.com/crossoverJie/blog.toolbox
mvn clean package
java -jar nows-0.0.1-SNAPSHOT.jar --app.downLoad.path=/xx/img /xx/xx/path 100
```

看运行方式也知道，其实就是用 `SpringBoot` 写了一个工具用于批量下载文中出现的图片同时上传后完成替换。

- 其中 `app.downLoad.path` 是用于将下载的图片保存到本地磁盘的目录。
- `/xx/xx/path` 则是扫描 `.md` 文件的目录，会递归扫描所有出所有文件。
- 100 则是需要替换文件的数量，默认是按照文件修改时间排序。

如果自己的图片较多的话还是有几个坑需要注意下。


## 线程数量 

默认是启动了两个线程去遍历文件、上传下载图片、更新文本等内容，其中的网络 IO 其实挺耗时的，所以其实可以适当的多开些线程来提高任务的执行效率。

但线程过多也许会触发图床的保护机制，同时也和自己电脑配置有关，这个得结合实际情况考虑了。

所以可以通过 `--app.thread=6` 这样的参数来调整线程数量。

## 图床限制

这个是图片过多一定是大概率出现的，上传请求的频次过高很容易被限流封 IP。

```json
{"code":"error","msg":"Upload file count limit. Time left 1027 second."}
```

目前来看是封 IP 居多，所以可以通过走代理、换网络的方式来解决。

当然如果是自搭图床可以无视。


## 重试

由于我使用的是免费图床，上传过程中偶尔也会出现上传失败的情况，因此默认是有 5 次重试机制的；如果五次都失败了那么大概率是 IP 被封了。

> 即便是 ip 被封后只要换了新的 ip 重新执行程序它会自动过滤掉已经替换的图片，不会再做无用功，这点可以放心。

## 图片保存

![](https://i.loli.net/2019/05/08/5cd1d002b6cff.jpg)

默认情况下,下载的图片会保存在本地，我也建议借此机会自己本地都缓存一份，同时名字还和文中的名字一样，避免今后图床彻底挂掉后连恢复的机会都没有。


# 总结

这个程序的代码就没怎么讲了，确实也挺简单，感兴趣的可以自己下来看看。

目前功能也很单一，自用完全够了；看后续大家是否还有其他需求再逐渐完善吧，比如：

- 图床上传失败自动切换到可用图床。
- 整体处理效率提升。
- 任务执行过程中更好的进度展现等。


再次贴一下源码地址：

[https://github.com/crossoverJie/blog.toolbox](https://github.com/crossoverJie/blog.toolbox)

**你的点赞与分享是对我最大的支持**
