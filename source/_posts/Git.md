---
title: Git学习
date: 2016-05-25 10:15:46
tags: git
category: tool
---

# 远程仓库的使用

## 远程仓库的添加

使用`git remote add <shortname> <url>`命令，url可以是ssh或者https。

```
$ git remote add origin git@github.com:sptzxbbb/sptzxbbb.github.io.git
```

## 远程仓库的查看

`git remote`可以查看当前git project里面所有远程仓库信息，加上`-v`会显示读写远程仓库的对应URL

```
$ git remote -v
origin	git@github.com:sptzxbbb/sptzxbbb.github.io.git (fetch)
origin	git@github.com:sptzxbbb/sptzxbbb.github.io.git (push)
```

如果想要查看某一远程仓库的详细信息，使用`git remote show remote_name`命令。

```
$ git remote show origin
* remote origin
  Fetch URL: git@github.com:sptzxbbb/sptzxbbb.github.io.git
  Push  URL: git@github.com:sptzxbbb/sptzxbbb.github.io.git
  HEAD branch: master
  Remote branches:
    blog-source tracked
    master      new (next fetch will store in remotes/origin)
  Local refs configured for 'git push':
    blog-source pushes to blog-source (up to date)
    master      pushes to master      (local out of date)
```

上面列出了远程仓库的所有信息，以及本地分支跟踪远程分支的情况。


## 远程仓库的同步

使用`git fetch remote_name`来访问远程仓库，从中拉取所有本地没有的最新数据，更新到本地的`remote_branch`。

如果当前的本地分支追踪了一个远程分支，那么可以使用`git pull`来更新追踪的远程分支，并自动合并到当前的本地分支，是一个很方便的功能。

## 远程仓库的移除和重命名

如果要修改一个远程仓库的名称，可以用`git remote rename`

```
$ git remote
remote_repo
origin

$ git remote rename remote_repo some_name

$ git remote
some_name
origin
```

这同时会修改远程分支的名字，如`some_name/master`。

如果需要移除某个远程仓库，使用`git remote rm`

```
$ git remote rm some_name
$ git remote
origin
```

# 关于Git Branch

git的branch实质上是一个指针，学过c/c++的人理解起来会很轻松，实际上当我们clone一个远程仓库时候，git branch背后是怎么操作的？

<!--more-->

当clone完成后，git会在本地生成一个master指针，初始化为origin/master的值，以我blog的repo为例。

```
$ git clone git@github.com:sptzxbbb/sptzxbbb.github.io.git
```

我们使用`git branch -a`来看看里面的分支情况。

```
* master
  remotes/origin/HEAD -> origin/master
  remotes/origin/hexo
  remotes/origin/master
```

这个repo有两个远程分支，`master`和`hexo`，默认的本地分支`master`指向前者。

`origin/HEAD`是一个remote仓库设定的符号分支，它指定了clone操作后当前本地分支追踪的默认远程分支，默认情况下是本地创建`master`分支，追踪`origin/HEAD`指向的`origin/master`分支。


```
$ git add *
$ git commit -m "Commit Messages"
$ git push origin local_branch：origin/branch
```

当我们进行add，commit，push一系列如上操作后，推送的过程其实是使**远程的origin/branch同步变成本地branch指针的状态**，如果origin/branch留空，默认为当前local_branch追踪的remote_branch.

```
推送到local_branch追踪的remote_branch
$ gti push origin local_branch
```

从这个角度来理解，创建新的远程分支和删除已有远程分支的命令如下。

```
$ git push origin local_branch:origin/branch_to_create
$ git push origin :origin/branch_to_create
```

将一个**不存在的远程分支(可以为空)**同步为一个本地分支的状态。(当然github上repo的页面上也能直接创建远程分支。)

将一个远程分支的状态同步为一个本地**空分支**的状态。

