---
title: 用位运算为你的程序加速
date: 2022/07/12 08:12:36 
categories: 
- xjson
- compiler
tags: 
- go
- Bitwise operation
---

![](https://tva1.sinaimg.cn/large/e6c9d24ely1h4p37zeb9aj20xc0go0tv.jpg)

# 前言

最近在持续优化之前编写的 `JSON` 解析库 [xjson](https://github.com/crossoverJie/xjson)，主要是两个方面的优化。

第一个是支持将一个 JSONObject 对象输出为 JSON 字符串。

这点在上个版本中只是利用自带的 Print 函数打印数据：

```go
func TestJson4(t *testing.T)  {
	str := `{"people":{"name":{"first":"bob"}}}`
	first := xjson.Get(str, "people.name.first")
	assert.Equal(t, first.String(), "bob")
	get := xjson.Get(str, "people")
	fmt.Println(get.String())
	//assert.Equal(t, get.String(),`{"name":{"first":"bob"}}`)
}
```

Output:
```shell
map[name:map[first:bob]]
```


本次优化之后便能直接输出 JSON 字符串了：
![](https://tva1.sinaimg.cn/large/e6c9d24ely1h4p3ihijjgj20ne05s0sw.jpg)



# 用位运算优化

# 其他奇淫巧技

