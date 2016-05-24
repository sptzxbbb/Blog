---
title: spacemacs buffer bug
tags:
---

# Scenario

  * 使用Spacemacs打开一个~/Desktop/test/example.cpp
  * 删除test文件夹
  * 尝试Spacemacs当前的buffer(example.cpp)进行操作，无论是kill还是switch都无法进行
    * Minibuffer显示：Setting current directory: no such file or directory, ~/Desktop/test/
