---
title: curl常用功能笔记
tags: tool
date: 2016-05-27 14:51:13
---


curl是一种命令行工具，能够发出网络请求，得到feedback，输出在console上面。

下面记录一些我在学习工作中常用的功能。

<!--more-->

## 查看源码

直接在curl后面加上网址，curl返回网页源码。

```
$ curl www.sina.com
```

使用`-o filename`参数可以保存网页。

## 自动跳转

有些网址会自动跳转，使用`-L`参数，curl随着网页自动跳转。

```
$ curl -L www.sina.com
```

输入以上命令，curl会跳转到`www.sina.com.cn`.

## 显示头信息

`-i`参数可以附加显示Http Response头部的信息

```
$ curl -i www.sina.com
HTTP/1.1 301 Moved Permanently
Server: nginx
Date: Fri, 27 May 2016 06:17:39 GMT
Content-Type: text/html
Location: http://www.sina.com.cn/
Expires: Fri, 27 May 2016 06:19:39 GMT
Cache-Control: max-age=120
Age: 75
Content-Length: 178
X-Cache: HIT from cmnet180-77.sina.com.cn

<html>
<head><title>301 Moved Permanently</title></head>
<body bgcolor="white">
<center><h1>301 Moved Permanently</h1></center>
<hr><center>nginx</center>
</body>
</html>
```

大写的`-I`参数则只会显示头部信息。

```
HTTP/1.1 301 Moved Permanently
Server: nginx
Date: Fri, 27 May 2016 06:18:42 GMT
Content-Type: text/html
Location: http://www.sina.com.cn/
Expires: Fri, 27 May 2016 06:20:42 GMT
Cache-Control: max-age=120
Age: 64
Content-Length: 178
X-Cache: HIT from cmnet.xxg.18a4.34.spool.sina.com.cn

```

## 显示通信过程

`-v`参数可以显示一次http通信的完整过程，包括端口连接和Http Response的头部信息。

```
curl -v www.sina.com
* Rebuilt URL to: www.sina.com/
*   Trying 221.179.180.75...
* Connected to www.sina.com (221.179.180.75) port 80 (#0)
> GET / HTTP/1.1
> Host: www.sina.com
> User-Agent: curl/7.47.0
> Accept: */*
> 
< HTTP/1.1 301 Moved Permanently
< Server: nginx
< Date: Fri, 27 May 2016 06:20:11 GMT
< Content-Type: text/html
< Location: http://www.sina.com.cn/
< Expires: Fri, 27 May 2016 06:22:11 GMT
< Cache-Control: max-age=120
< Age: 74
< Content-Length: 178
< X-Cache: HIT from cmnet180-75.sina.com.cn
< 
<html>
<head><title>301 Moved Permanently</title></head>
<body bgcolor="white">
<center><h1>301 Moved Permanently</h1></center>
<hr><center>nginx</center>
</body>
</html>
* Connection #0 to host www.sina.com left intact

```

更详细的通信过程可以用`--trace`参数获得。

```
$ curl --trace output www.sina.com
```

结果保存在`output`文件里。


## 发送表单信息

发送http的表单信息有`GET`和`POST`两种方法。

`GET`只需要把表单附加在网址后。

```
$ curl examplecom/form.cgi?data=xxx
```

`POST`方法必须把数据和网址分开，需要用到`-data`参数

```
$ curl -X POST --data "data=xxx" example.com/form.cgi
```

## 指定Http动作

curl默认http的动作是`GET`，使用`-X`参数可以指定执行其他动作。

```
$ curl -X POST www.example.com
$ curl -X DELETE www.example.com
```

## 文件上传

假设现在有一张文件上传的表单如下。

```
　　<form method="POST" enctype='multipart/form-data' action="upload.cgi">
　　　　<input type=file name=upload>
　　　　<input type=submit name=press value="OK">
　　</form>
```

curl可以这样帮你上传文件。

```
$ curl --form upload=@localfilename --form press=OK [URL]
```

## Referer字段

http request的头部信息里面，提供一个referer字段，表示当前页面是从哪个页面跳转过来的。

```
$ curl --referer http://refer_page.com  http://curren_page.com
```

## User Agent字段

这个字段是用来表示客户端的设备信息，有些服务器会根据这个字段，针对不同设备，返回不同格式的网页，如手机版和桌面版。

```
$ curl --user-agent "User Agent" URL
```

## cookie

使用`cookie`参数，可以发送`cookie`。

```
$ curl --cookie "name=xxx" www.example.com
```

至于具体的`cookie`值，可以从Http Response头部信息的`Set-Cookie`中得到。

`-c cookie-file`可以用来保存服务器返回的cookie为文件。
`-b cookie-file`可以使用这个文件作为cookie信息。

```
$ curl -c cookies-file www.example.com
$ curl -b cookies-file www.example.com
```


## 增加头部信息

有时候需要在Http Response中增加一个头部信息，`--header`参数可以实现这一点。

```
$ curl --header "Content-Type:application/json" http://example.com
```

## HTTP认证

有些网域需要HTTP认证，这是curl需要添加`--user`参数。

```
$ curl --user name:password www.example.com
```
