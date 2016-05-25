---
title: Git学习
date: 2016-05-25 10:15:46
tags: git
category: tool
---

 

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



```
$ git add *
$ git commit -m "Commit Messages"
$ git push origin local_branch：origin/branch
```

当我们进行add，commit，push一系列如上操作后，推送的过程其实是使**远程的origin/branch同步变成本地branch指针的状态**，如果origin/branch留空，默认为origin/master，远程的主分支。

从这个角度来理解，创建新的远程分支和删除已有远程分支的命令如下。

```
$ git push origin local_branch:origin/branch_to_create
$ git push origin :origin/branch_to_create
```

将一个**不存在的远程分支(可以为空)**同步为一个本地分支的状态。(当然github上repo的页面上也能直接创建远程分支。)

将一个远程分支的状态同步为一个本地**空分支**的状态。

