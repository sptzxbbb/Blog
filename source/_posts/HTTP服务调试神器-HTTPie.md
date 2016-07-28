---
title: HTTP服务调试神器-HTTPie
date: 2016-07-27 17:24:43
categories: [Coding]
tags: [tool]
---

过去在进行web开发，restfulAPI，调试的工具一直是cURL。但cURL反人类的繁琐语法命令让我感到十分恶心，幸运的是我在Github上找到了一个交互十分友好的工具---**HTTPie**, 一个命令行式的HTTP客户端，致力于使用简单自然的语法和丰富多彩的输出与HTTP server测试，调试和交互。

![HTTPid](https://raw.githubusercontent.com/jkbrzt/httpie/master/httpie.png) 


<!--more-->

# Feature

HTTPie以Python编写，主要使用了`Request`和`Pygments`库。

+ 直观的语法
+ 格式化和彩色终端输出
+ 内置JSON支持
+ 支持上传表单和文件
+ HTTPS，代理和认证
+ 自定义头部
+ 持久性会话
+ 跨平台，Linux, Mac OS X和Windows
+ 插件
+ 重定向

# Usage

HTTPie的语法相当简洁，pattern如下。

```
$ http [flags] [METHOD] URL [ITEM [ITEM]]
```

请求中的ITEM格式如下，

- Name：Value， 头文件
- name==value, url查询参数
- field=value, field=@file.txt, 以JSON格式序列化请求数据
- field:=json, field:=@file.json, 原始JSON字段
- field@/dir/file, 表单中的上传文件，multipart/form-data


自定义头部和JSON数据

```
$ http PUT example.org X-API-Token:123 name=John
```

提交表单

```
$ http -f POST example.org hello=World
```

重定向上传文件

```
$ http example.org < file.json
```

更详细的用法请戳[这里](https://github.com/jkbrzt/httpie) 
