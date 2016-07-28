---
title: Spacemacs常用功能
tags:
  - emacs
categories: Coding
date: 2016-07-27 17:08:05
---



一直使用`Spacemacs`作为我的开发IDE，`Spacemacs`提供了许多十分强大，让我爱不释手的`Feature`，但同时也要记得许多的快捷键才能用得痛快，下面列出一些我常用的快捷键作为备忘录。

<!--more-->    

## 配置文件

`<SPC f e d>`, 打开`.spacemacs`文件。

`<SPC f e R>`，重新加载`.spacemacs`文件，主要用于修改配置后直接apply新的配置，避免重启`spacemacs`的麻烦。


## 注释代码

`<SPC ;>`, 在当前行尾加入注释符号，不建议采用这种注释，因为注释独立成行成段更清晰明了。

`<g c>`, 注释/取消注释目标。例子如下，
+ `<g c c>`, 注释当前行代码。
+ `<g c a p>`, 注释一段代码。

## evil-surround插件

`<c s " '>`, change surround, 把成对的符号""换成''。

`<d s ">`, delete surround, 把成对的符号""去掉。

## 编辑

`<SPC x d w>`, 移除所有拖尾的空格。

`<SPC '>`, 打开终端， Spacemacs特意对终端进行了优化，比起裸Emacs时候好用了太多太多。

`<C-c $>`, 弹出flyspell的纠错列表，更正word, 但我认为这个快捷键相当不方便。

`<SPC s a b>`, 在所有打开的buffer里使用ag搜索。

