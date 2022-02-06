---
title: 【译】Java8 之后对新开发者非常友好的特性
date: 2022/02/07 08:03:13       
categories: 
- 翻译
tags: 
- Java
---


**[原文链接](https://piotrminkowski.com/2021/02/01/new-developer-friendly-features-after-java-8/)**


![](https://tva1.sinaimg.cn/large/008i3skNly1gz3vzlzrs6j30qo0f0dhz.jpg)


在这篇文章中，我将描述自 Java8 依赖对开发者来说最重要也最友好的特性，之所以选择 Java8 ，那是因为它依然是目前使用最多的版本。

具体可见这个调查报告：
![](https://tva1.sinaimg.cn/large/008i3skNly1gz3w509h16j30sg0ao74v.jpg)
<!--more-->


# Switch 表达式 (JDK 12)

使用 switch 表达式，你可以定义多个 case 条件，并使用箭头 `->` 符号返回值，这个特性在 JDK12 之后启用，它使得 switch 表达式更容易理解了。

```java
public String newMultiSwitch(int day) {
   return switch (day) {
      case 1, 2, 3, 4, 5 -> "workday";
      case 6, 7 -> "weekend";
      default -> "invalid";
   };
}
```
在 JDK12 之前，同样的例子要复杂的多：

```java
public String oldMultiSwitch(int day) {
   switch (day) {
      case 1:
      case 2:
      case 3:
      case 4:
      case 5:
         return "workday";
      case 6:
      case 7:
         return "weekend";
      default:
         return "invalid";
   }
}
```

# 
