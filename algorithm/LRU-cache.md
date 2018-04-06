---
title: 动手实现一个 LRU cache
date: 2018/04/07 01:01:36 
categories: 
- 算法
- LRU cache
---

![](https://ws1.sinaimg.cn/large/006tNc79gy1fq3fey7n97j31340o8myw.jpg)

## 前言
LRU 是 `Least Recently Used` 的简写，字面意思则是`最近最少使用`。

通常用于缓存的淘汰策略实现，由于缓存的内存非常宝贵，所以需要根据某种规则来剔除数据保证内存不被撑满。

如常用的 Redis 就有以下几种策略：

| 策略 | 描述 |
| :--: | :--: |
| volatile-lru | 从已设置过期时间的数据集中挑选最近最少使用的数据淘汰 |
| volatile-ttl | 从已设置过期时间的数据集中挑选将要过期的数据淘汰 |
|volatile-random | 从已设置过期时间的数据集中任意选择数据淘汰 |
| allkeys-lru | 从所有数据集中挑选最近最少使用的数据淘汰 |
| allkeys-random | 从所有数据集中任意选择数据进行淘汰 |
| no-envicition | 禁止驱逐数据 |

> 来自:[https://github.com/CyC2018/Interview-Notebook/blob/master/notes/Redis.md#%E5%8D%81%E4%B8%89%E6%95%B0%E6%8D%AE%E6%B7%98%E6%B1%B0%E7%AD%96%E7%95%A5](https://github.com/CyC2018/Interview-Notebook/blob/master/notes/Redis.md#%E5%8D%81%E4%B8%89%E6%95%B0%E6%8D%AE%E6%B7%98%E6%B1%B0%E7%AD%96%E7%95%A5)


## 实现一

之前也有接触过一道面试题，大概需求是：

- 实现一个 LRU 缓存，当缓存数据达到 N 之后需要淘汰掉最近最少使用的数据。
- N 小时之内没有被访问的数据也需要淘汰掉。

## 实现二

## 实现三

## 总结


