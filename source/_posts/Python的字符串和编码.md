---
title: Python的字符串和编码
date: 2016-07-29 16:16:01
categories: [Coding]
tags: [Python]
---

最开始学Python时候一直对`str`和`bytes`理不清楚, 今天好好梳理以下两者的区别。

<!--more-->

# 简述

Python3中把字符串明确分为`str`和`bytes`两种对象。

`str`用`Unicode`编码，因此字符串处理可以兼容多国语言，免去编程时候的不同语言造成的乱码。

`bytes`为二进制编码，通过`str`对象使用某种编码得到，占用内存小，方便传输和保存。


# str

Python提供了`ord()`函数来获取单个字符的`Unicode`编码，反过来也提供了`chr()`把`Unicode`编码转换成对应字符的函数。

```python
>>> ord('A')
65
>>> ord('斌')
25996
>>> chr(65)
'A'
>>> chr(25996)
'斌
```
# bytes

`bytes`在Python中需要加上`b`前缀来表示。

```python
s = b'Hello, world!'
```


注意，对`bytes`对象使用`len()`函数时候，计算的是字节数。

```
>>> len('中文'.encode('utf-8'))
6
>>> len('中文')
2
```

# 转换

Python提供了`encode()`和`decode()`两个方法来互相转换`str`和`bytes`，这两个函数都需要提供具体的编码方法来进行翻译。

![demo](/images/Python的字符串和编码/demo.png) 

一般从网络和磁盘中读取的字节流都是`bytes`对象，这时候需要使用`decode()`来进行解析。
反过来，进行网络传输和保存文件到磁盘时候，这时候需要`encode()`来把`str`转换成`bytes`，提高效率。

Python源代码本身也是一个文本文件，所以当源代码中包含有`ASCII`范围以外的字符时候，需要保存为`UTF-8`编码。
在代码头部加上这一行。

```
# -*- coding: utf-8 -*-
```

这样Python解释器才会按照`UTF-8`的编码来翻译磁盘上的Python源代码。
