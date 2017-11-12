---
title: 【译】你可以用GitHub做的12件 Cool 事情
date: 2017/11/5 01:01:54       
categories: 
- 翻译
tags: 
- GitHub
---

![](https://ws3.sinaimg.cn/large/006tNc79ly1flef224anmj31kw0ebgnt.jpg)

### [原文链接](https://hackernoon.com/12-cool-things-you-can-do-with-github-f3e0424cf2f0)

## 1 在 GitHub.com 编辑代码

我将从我认为大家都知道的一件事情开始(尽管我是直到一周前才知道)。

当你在 GitHub 查看文件时(任何文本文件，任何仓库中)，右上角会有一个小铅笔图标，点击它就可以编辑文件了。完成之后点击 **Propose file change** 按钮 GitHub 将会自动帮你 fork 该项目并且创建一个 `pull request` 。

很厉害吧！他自动帮你 `fork` 了该 repo。

不再需要 `fork` , `pull` ,本地编辑再 `push` 以及创建一个 `PR` 这样的流程了。
![](https://ws3.sinaimg.cn/large/006tKfTcgy1fl5eo3789hj30m80mjwhy.jpg)

这非常适合修复编写代码中出现的拼写错误和修正一个不太理想的想法。

## 2 粘贴图片

你不仅仅受限于输入文本和描述问题，你知道你可以直接从粘贴板中粘贴图片吗？当你粘贴时，你会看到图片已经被上传了(毫无疑问被上传到云端)之后会变成 `Markdown` 语法来显示图片。


## 3 格式化代码
如果你想写一段代码，你可以三个反引号开始 —— 就像你在[研究`MarkDown`](https://guides.github.com/features/mastering-markdown/)时所学到的 —— 之后 GitHub 会试着猜测你写的语言。

但如果你写了一些类似于 Vue, Typescript, JSX 这样的语言，你可以明确指定得到正确的高亮。

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

看到了吧，浏览器的 `URL` 已经被更新为行号了。如果你再按住 `shift` 点击其他行号，`URL` 再次被更新了，并且页面也会高亮显示一段代码。

分享这个 `URL` 访问时将会链接到该文件已经选中的那些代码段。

等等，这指向的是当前的分支，如果文件改变了呢？也许你想要的是永久链接到当前状态。

我很懒，所以一张截图就完成了以上的所有操作。

![](https://ws2.sinaimg.cn/large/006tKfTcly1fl6lw7ji6uj30m80a6mym.jpg)

谈到网址。。。

## 7 像命令行一样使用GitHub链接
使用 GitHub 自带的 UI 浏览也还不错，但有时直接在 URL 中输入效率更高。比如，我想要跳转到我正在工作的分支并和 master 进行对比，就可以直接在项目名称后面接上 `/compare/branch-name` 。

与选中分支的对比页将会显示出来:
![](https://ws2.sinaimg.cn/large/006tKfTcly1fl8rqb8egpj30u50pbaea.jpg)

以上就是和 master 分支的差异，如果想要合并分支的话，只需要输入 `/compare/integration-branch...my-branch
` 即可。

![](https://ws2.sinaimg.cn/large/006tKfTcly1fl8s3yraqrj30u50pbgpx.jpg)

不仅如此，你还可以使用快捷键来达到同样的效果， `ctrl + L` 或者 `cmd + L` 可以将光标移动到 `URL` 上(至少在 Chrome 中可以)。 加上浏览器的自动补全 —— 即可方便的在两个分支之间切换了。


## 8 在Issues创建列表

你想在你的 issue 中看到复选框列表嘛?

![](https://ws2.sinaimg.cn/large/006tKfTcly1fl8snpyb0jj30e708sdgl.jpg)

并且当查看 issue 列表的时候你想看到一个漂亮的 `2 of 5` 进度条嘛？

![](https://ws1.sinaimg.cn/large/006tKfTcly1fl8u3bn88hj30dn06jdg9.jpg)

很好，你可以用以下语法来创建一个具有交互的复选框:

```
 - [ ] Screen width (integer)
 - [x] Service worker support
 - [x] Fetch support
 - [ ] CSS flexbox support
 - [ ] Custom elements
```

是由一个空格、中横线、空格、左括号、空格(或者是 X )、右括号、空格以及一些文本组成。

你甚至可以真正的 选中/取消 这些复选框！基于某些原因，对于我来说你看起来像是技术魔力。是真的能够选中这些复选框！甚至它还更新了底层源码。

> ps：以下包括第九点 基于GitHub的项目面板 由于用的不多就没有翻译。

## 10 GitHub wiki

作为一个非结构化的页面集合 —— 就像维基百科 —— `GitHub Wiki`(我把它称之为 `Gwiki` ) 是一个非常棒的功能。

对于结构化的页面来说 —— 就拿你的文档来举例:不能说这个页面是其他页面的子页面，或则是有 “下一节”，“上一节” 这样的好东西。并且 `Hansel` 和 `Gretel` 也没有，因为结构化页面并没有 `breadcrumbs` 这样的设计。

我们继续，让 Gwiki 动起来，我从 `NodeJS` 的文档中复制了几页来作为 wiki 页面。并创建了一个自定义侧边栏来更好的模拟一些实际的目录结构。尽管它不会突出的显示你当前的页面位置，但也会一直存在。

这些链接需要你自己手动维护，但总的来说，我认为它会做的很好。
如果需要的话可以[看看](https://github.com/davidgilbertson/about-github/wiki)。

![](https://ws2.sinaimg.cn/large/006tKfTcgy1fl9yl6mhlzj30rw0dqwgn.jpg)

它不会和 `GitBook` ( [Redux 文档](http://redux.js.org/)所使用的)或者是定制网站存在竞争关系。

But it’s a solid 80% and it’s all right there in your repo.
`在你的 repo 中他是 80% 值得信赖的。`

我的建议是: 如果你已经有了多个 `README.md` 文件了，并且想要为用户指南或更详细的文档需要一些不同的页面，那么你应该选择 Gwiki。

如果你开始觉得缺乏结构化/导航使你不爽的话，那就继续其他的就好了。

## 11 GitHub Pages
你可能已经知道可以使用 `GitHub Pages` 来托管一个静态网站，And if you didn’t now you do.，这一节是专门用于讨论使用 `Jekyll` 来构建一个站点的。

最简单的就是： `GitHub Pages + Jekyll` 通过一个漂亮的主题来渲染你的 `README.md` 文件。例如:通过 [about-github](https://github.com/davidgilbertson/about-github) 来查看的我的 `README` 页面。

![](https://ws3.sinaimg.cn/large/006tNc79ly1fle60d5j7hj311l0np78i.jpg)

如果我在 GitHub 中点击了 `settings`选项，切换到 `Github Pages` 设置，然后选择一个 `Jekyll theme`。。。

![](https://ws1.sinaimg.cn/large/006tNc79ly1fle73empc8j30l30bcdh1.jpg)

我就可以得到 [Jekyll-themed 页面](https://davidgilbertson.github.io/about-github/)。

![](https://ws4.sinaimg.cn/large/006tNc79ly1fle74nogxlj311l0npgos.jpg)

从这点上:可以主要围绕易编辑的 `Markdown` 文件来构建一个完整的静态站点。本质上是把 GitHub 变成了 `CMS`。

我虽然没有实际使用过，但是 `React` `Bootstrap` 的网站都是使用它来构建的。所以看起来还不错。

注意:它需要本地的 `Ruby` 运行环境( Windows 自行安装， macOS 自带)。



## 12 把 GitHub 当做 CRM 使用

假设你有一个存有一些文本内容的网站，你不想将文本内容存储于真正的 `HTML` 源码中。

相反的，你想要将这些文本块存储于某个地方可以方便的让非开发人员轻松的进行编辑。可能是一个版本控制系统，甚至也可以是一个审核流程。

我的建议是:使用 GitHub 厂库中的 Markdown 文件来存储这些文本内容，然后使用前端组件来拉取这些文本块并展示在页面上。

我是搞 React 的，这有一个 解析 `Markdown` 的组件例子，给定一些 Markdown 文件路径，它将会自动拉取并作为 `HTML` 显示出来。

```react
class Markdown extends React.Component {
    constructor(props) {
      super(props);
      
      // replace with your URL, obviously
      this.baseUrl = 'https://raw.githubusercontent.com/davidgilbertson/about-github/master/text-snippets';
      
      this.state = {
        markdown: '',
      };
    }

    componentDidMount() {
      fetch(`${this.baseUrl}/${this.props.url}`)
        .then(response => response.text())
        .then((markdown) => {
          this.setState({markdown});
        });
    }

    render() {
      return (
        <div dangerouslySetInnerHTML={{__html: marked(this.state.markdown)}} />
      );
    }
}
```

### 奖励环节 —— GitHub 工具

我已经使用了 [Octotree Chrome extension](https://chrome.google.com/webstore/detail/octotree/bkhaagjahfmjljalopjnoealnfndnagc?hl=en-US) 很长一段时间了，我推荐它！

无论你是在查看哪个 repo 它都会在左侧给你一个树状面板。

![](https://ws2.sinaimg.cn/large/006tNc79ly1flee4cevv3j30rs0hygnv.jpg)

通过这个[视频](https://www.youtube.com/watch?v=NhlzMcSyQek&index=2&list=PLNYkxOF6rcIB3ci6nwNyLYNU6RDOU3YyL)我了解到了 [octobox](https://octobox.io/)，它看起来相当不错，是用于管理你的 `GitHub Issues` 收件箱。

以上就是针对于 `octobox` 我要说的了。

## 其他

就是这样了，我希望这里有三件事你不知道，

最后:
hava a nice day！
