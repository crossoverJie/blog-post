---
title: 【译】你可以用GitHub做的12件 Cool 事情
date: 2017/11/5 01:01:54       
categories: 
- 翻译
tags: 
- GitHub
---
### [原文链接](https://hackernoon.com/12-cool-things-you-can-do-with-github-f3e0424cf2f0)

## 1 在GitHub.com编辑代码

我将会从我认为大家都知道的一件事情开始(尽管我是直到一周前才知道)。

当你处于GitHub，查看文件时(任何文本文件，任何仓库中)，在右上角都会有一个小铅笔图标。点击就可以编辑文件了。当你完成之后点击 **Propose file change** 按钮GitHub将会自动帮你fork该项目并且创建一个 `pull request` 。

Isn’t that wild? It creates the fork for you!

不再需要 `fork` , `pull` ,本地编辑再 `push` 以及创建一个 `PR` 这样的流程了。
![](https://ws3.sinaimg.cn/large/006tKfTcgy1fl5eo3789hj30m80mjwhy.jpg)

这非常适用于修复拼写错误。

## 2 粘贴图片

在评论和`issue `描述时不仅限于输入文本，你知道可以直接从粘贴板中粘贴图片嘛？当你粘贴时，你会看到图片已经被上传了(毫无疑问肯定是被上传到云端了。)之后会变成`Markdown`语法来显示图片。


## 3 格式化代码
如果你要写一段代码，你可以使用三个反引号开始——就像你在学习[掌握`MarkDown`](https://guides.github.com/features/mastering-markdown/)时所学到的 —— 之后GitHub会尝试猜出你所写的那种语言。

但是如果你写了一些类似于 Vue, Typescript, JSX,你可以详细的指定出正确的高亮现实。

注意第一行中的
```
```jsx
```

![](https://ws1.sinaimg.cn/large/006tKfTcgy1fl5fe2vu3rj30b607kq39.jpg)

这意味着代码段将会呈现出:

![](https://ws4.sinaimg.cn/large/006tKfTcgy1fl5ffl8q7sj30bz06gq36.jpg)

(这个扩展于 `gists` 。顺便说一句，如果你使用 `.jsx` 后缀，就会得到JSX的语法高亮)

这儿有所有支持的[语法列表](https://github.com/github/linguist/blob/fc1404985abb95d5bc33a0eba518724f1c3c252e/vendor/README.md)。

## 4 在 PR 中用关键词关闭 Issues

假设你创建了一个 `PR` 用于修复`Issues #234`。你可以在你的 `PR` 填写 `fixes #234` (或者在该`PR` 的任意评论中填写都是可以的)。

之后合并这个 `PR` 时将会自动关闭填写的 `Issues`。怎么样很 cool 吧。

更多相关的[内容](https://help.github.com/articles/closing-issues-using-keywords/)。

## 5 链接到评论
你是否有过想要链接到特殊 `comment ` 的想法但却无法实现？那是因为你不知道怎么做。朋友那都是过去式了，现在我就告诉你，点击用户名旁边的日期/时间即可链接到该 `comment ` 。

![](https://ws3.sinaimg.cn/large/006tKfTcly1fl69xkm5rtj30d003zq37.jpg)

## 6 链接到代码

我知道你也想链接到具体的代码行上。

尝试:查看文件时，点击代码旁边的行号。

看到了吧，浏览器的 `URL` 已经被更新为行号了。如果你再按住 `shift` 点击其他行号，`URL` 再次被更新了，并且页面也会高亮现实一段代码。

分享这个 `URL` 访问时将会链接到该文件已经选中的那些代码段。等等，这指向的是当前的分支，如果文件改变了呢？也许你想要的是永久链接到当前状态。

我很懒，所以一张截图就完成了以上的所有操作。

![](https://ws2.sinaimg.cn/large/006tKfTcly1fl6lw7ji6uj30m80a6mym.jpg)

谈到网址。。。

## 7 像命令行一样使用GitHub链接



## 8 在Issues创建列表

## 10 GitHub wiki

## 11 GitHub Pages

## 12 把GitHub当做CRM


