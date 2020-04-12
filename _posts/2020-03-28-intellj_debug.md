---
layout:     post
title:      "Intellij IDEA高阶DEBUG大杀器"
subtitle:   "\"解决复杂调试问题必须良药\""
date:       2020-03-28 12:00:00
author:     "wangyapu"
header-img: "img/post-idea-debug.jpeg"
tags:
    - 工具
---

# Intellij IDEA高阶DEBUG大杀器

## 前言

目前工作中由于环境复杂等客观因素，无法在本地启动项目进行 Trouble Shooting，需要打开测试环境的 DEBUG 端口，进行远程调试。为了不影响其他用户同时使用测试环境以及相关系统的正常请求，只好再祭出Intellij IDEA 的 DEBUG 大杀器了。本文主要介绍平时用到几种 DEBUG 高阶用法。

## 快速上路

### 安装Intellij IDEA

Intellij IDEA 每次更新都会有些小惊喜，谁用谁知道。

If you don't have intellij idea installed, return. 😂

### 条件断点

1. 找到需要断点的代码行，鼠标点击左侧边栏设置断点（CMD + F8 ）。好吧我废话了，这个大家都会。
2. 右击断点，在 Condition 处填入你所需要的判断条件。

    ![](http://wangyapu.iocoder.cn/15864909588043.jpg)

3. 当 DEBUG 时候断点会停留在 i = 5 的位置。

    ![](http://wangyapu.iocoder.cn/15864906470601.jpg)
 

### 不暂停断点调试

1. 依然找到需要断点的代码行。右击设置好的红色断点，选择 More（SHIFT + CMD + F8 ），此时会弹出断点的设置菜单。

2. `启用不暂停断点，去除 Suspend 勾选框，勾选 “Breakpoint hit” message 以及 Evaluate and log。`如果你对代码的调用层次不清楚或者你在阅读学习源码，你可以勾选 “Stack trace”，

    ![](http://wangyapu.iocoder.cn/15866666619415.jpg)

    配合 Grep Console 插件，调试效果无敌。蓝色部分是我们打印的调试日志，红色框部分记录了断点是否执行以及代码调用的堆栈。

    ![](http://wangyapu.iocoder.cn/15866667689018.jpg)

3. 条件断点 & 变量值修改：条件断点就不再赘述了，`Evaluate and log 不仅可以记录日志而且可以修改变量。`如图所示，在 i==5 的时候，我人为添加了一条测试的假数据。

    ![](http://wangyapu.iocoder.cn/15866676874067.jpg)

    ![](http://wangyapu.iocoder.cn/15866678334658.jpg)

### 多线程断点调试

多线程调试蛮头疼的，因为代码执行的先后顺序完全看 CPU，断点跳来跳去，这就给调试带来了很多麻烦。IDEA 具备多线程调试的切换能力，你可以按一定顺序来调试线程的代码。

1. 设置线程断点。右击断点设置 Suspend 挂起条件为 Thread。

    ![](http://wangyapu.iocoder.cn/15866759835238.jpg)

2. 断点挂起时，可以切换线程进行调试。

    ![](http://wangyapu.iocoder.cn/15866767894973.jpg)

### “后悔药”可以有

平时调试的过程中经常遇到断点不小心跳过了，想回头再看看刚才的值或者更进一步的 DEBUG，是不是傻头傻脑地重新再跑一遍？

可能你会想到使用代码调用栈返回查看调用情况，但是你没办法在历史的某一步再重新调试。其实 IDEA 提供了返回上一步的功能，这个功能在学习源码的时候特别有用，可以几个断点之间反复来回研究并 DEBUG。

1. 如下图所示，如果我想回到第一个断点重新执行调试怎么办呢？

    ![](http://wangyapu.iocoder.cn/15866784918277.jpg)

2. 执行两次 Drop Frame 即可，可以看到 value 的值重新回到 0。

    ![](http://wangyapu.iocoder.cn/15866785893517.jpg)

> `该方法切勿在真实环境中使用！！！因为丢弃栈帧如果没有操作释放干净可能会影响变量的值，导致程序结果与真实结果不一致！！！`

### 结束语

你可能习惯了单步 Debug，短时间不会适应不暂停断点的调试方式，然而在复杂的功能场景下，如多线程、多层循环等，这种 Debug 方式不仅调试的速度更快，而且更有利于加深对代码整体的理解。

上面介绍的条件断点、不暂停断点以及多线程断点的调试方式是平时最常用的，我们平时所调试的代码远比文中例子复杂的多，熟能生巧，快点尝试下吧，希望对大家有所帮助。如果你有其他好的经验和技巧，欢迎留言分享。

