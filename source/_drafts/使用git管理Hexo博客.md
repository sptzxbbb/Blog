---
title: 使用git管理Hexo博客
tags:
---

在github上clone Hexo博客到本地。

```
$ git clone git@github.com:sptzxbbb/sptzxbbb.github.io.git
```

建立本地hexo分支追踪origin/hexo。

```
$ git checkout -b hexo origin/hexo
```

安装hexo

```
$ npm install hexo --save
```

Hexo 3.0把服务器独立成了个别模块，需要额外安装才能本地预览网站。

```
$ npm install hexo-server --save
```
