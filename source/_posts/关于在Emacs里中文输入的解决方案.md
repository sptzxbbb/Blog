---
title: 关于在Emacs里中文输入的解决方案
date: 2016-05-12
update: 2016-05-21 15:53:01
tags: Emacs
---

为了在Emacs中自由的写Blog，这两天一直在研究Emacs中文输入的solution。

我使用的是Spacemacs的配置，Spacemacs虽然增加了chinese这一层layer的支持，但实际使用时内置的中文输入法表现非常差，和Fcitx的体验判若云泥。这样就坚定了我使用外援Fcitx的决心。

![Embedded Input Method](/images/关于在Emacs里中文输入的解决方案/1.png) 


在Emacs里面，Fcitx是无法激活的。查阅了Fcitx的文档后发现是GTK程序(Emacs的图形界面是由GTK编写)的im(input module)不是Fcitx，例如我正在使用系统是Ubuntu16.04，此时GTK默认的im是ibus。所以Fcitx在Emacs里面无法激活。

<!--more-->

知道原因后，其实解决方案十分简单，通过改变环境变量来修改GTK的im模块。

将这段代码放在系统初始化文件里面即可，例如`.profile`, `.bashrc`等。

```
export GTK_IM_MODULE=fcitx
export XMODIFIERS=@im=fcitx
```

此时Emacs应该可以正常激活Fcitx，但中文输入法可能无法使用，再把下面这段代码补上。

```
export LC_CTYPE=zh_CN.UTF-8
```

运行`locale`和`fcitx-diagnose`查看上面的变量是否正确设置，没问题的话，Emacs应该可以正常使用中文。

进行了上述操作后依然可能出现环境变量没有发生变化，这时候可以尝试直接粗暴的卸载ibus框架(核心问题就是ibus和fcitx发生了框架冲突)，我在Ubuntu16.04上卸载ibus后，没有出现过去Ubuntu版本的系统模块功能缺失bug，看来Ubuntu16.04已经把ibus和系统模块彻底分离。

同理，如果其他QT，QTK程序无法输入中文，都可以尝试这种做法，例如WPS for linux。

绝大多数输入法的激活Key都是Ctrl+Space，这个组合键`Emacs`已经用于某个功能，因此还需要改输入法的激活Key或者将这个功能的组合键作一个新的映射。

Reference:
    [Input method related environment variables](https://fcitx-im.org/wiki/Input_method_related_environment_variables) 
    [Fcitx](https://wiki.gentoo.org/wiki/Fcitx) 


