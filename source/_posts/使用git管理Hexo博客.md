---
title: 使用git管理Hexo博客
date: 2016-05-25 10:41:41
tags: [git, hexo]
category: Coding 
---

# 搭建并管理博客

在github上创建repo，`sptzxbbb.github.io`，同时开设新的分支`hexo`

在本地执行`hexo init`创建blog，并关联到repo。

本地创建并切换到新的分支`hexo`来写post和管理blog的setting。

修改站点设置`_config.yml`中deploy参数，分支设置为master。

在hexo分支下push commit到远程仓库的远程分支hexo下即可完成blog源代码的管理。

执行`hexo g -d`将blog自动部署到github的repo的master分支上。

<!--more-->
    

# 在新的机器上重新搭建博客

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

安装依赖包
```
$ npm install 
```


本地预览时，遇到原因不明的bug，hang在以下语句，

```
$ 19:43:10.411 DEBUG Processed: source/vendors/font-awesome/fonts/fontawesome-webfont.svg
```

经过无数尝试后，找到一个workaround。

删除掉`_drafts`后，再新建**空**的`_drafts`，将原来的草稿copy到空的`_drafts`中即可。

