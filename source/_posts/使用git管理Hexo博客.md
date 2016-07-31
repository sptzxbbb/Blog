---
title: 使用git和Hexo管理博客
category: Coding 
date: 2016-05-25 10:41:41
tags: [git, hexo]
---

# 搭建并管理博客

在github上创建repo，`sptzxbbb.github.io`，同时开设新的分支`blog-source`

在本地执行`hexo init`创建blog，并关联到repo。

本地创建并切换到新的分支`blog-source`来写post和管理blog的setting。

修改站点设置`_config.yml`中deploy参数，分支设置为master。

在`blog-source`分支下push commit到远程仓库的远程分支`blog-source`下即可完成blog源代码的管理。

执行`hexo g -d`将blog自动部署到github的repo的master分支上。

<!--more-->
    

# 在新的机器上重新搭建博客

在github上clone Hexo博客到本地。

```
$ git clone git@github.com:sptzxbbb/sptzxbbb.github.io.git
```

建立本地hexo分支追踪origin/hexo。

```
$ git checkout -b blog-source origin/blog-source
```

安装`hexo`, 由于hexo3把服务器独立出来，作为一个附加模块，所以还要安装`hexo-server`来做本地预览。

```
$ npm install hexo
$ npm install hexo-server
```

安装依赖包
```
$ npm install 
```

本地预览时，遇到过原因不明的bug，hang在以下语句，

```
$ 19:43:10.411 DEBUG Processed: source/vendors/font-awesome/fonts/fontawesome-webfont.svg
```

经过无数尝试后，找到一个workaround。

删除掉`_drafts`后，再新建**空**的`_drafts`，将原来的草稿copy到空的`_drafts`中即可。

# hexo常用命令

```
$ hexo init  # 初始化hexo
$ hexo new draft Foo  # Hexo创建草稿Foo到source/_draft/。
$ hexo publish draft Foo  # Hexo把草稿Foo发布到source/_post/。
$ hexo server   # Hexo 会监视文件变动并自动更新，无须重启服务器。
$ hexo g -d  # Hexo生成文件和部署到服务器。
```

