---
title: "ArchLinux 安装及初步配置"
subtitle: "ArchLinux 安装及初步配置"
summary: "花了几天时间装好了 Arch，终于基本可以用了，记录一下过程吧"
description: "花了几天时间装好了 Arch，终于基本可以用了，记录一下过程吧"
image: "https://raw.githubusercontent.com/calendar0917/images/master/20251103190835625.png"
date: 2025-11-03
lastmod: 2025-11-03
draft: false
toc:
 enable: true
hiddenFromHomePage: false
weight: false
categories: ["技术杂项"]
tags: ["技术"]
---

> 好几天没写博客了，一是期中考，二是不知道要学些什么了。网上看见推荐 Linux 系统的文章，正好试一试自己装系统。

## 安装

安装花了大概一天时间，参考了 [Arch Linux 简明指南](https://arch.icekylin.online/guide/rookie/basic-install )，没什么太大问题。主要是教程中的做法是将 Arch 的 boot 挂载到了 Win 里面，做下来发现 Win 的 EFi 分区已经满了……

然后就是研究[[Btrfs子卷挂载]]，和挂载顺序有比较大关系。

大体步骤是：

1. Windows 当中划分出空硬盘区
2. U盘拷贝镜像，做系统盘，把 Win 的磁盘解密
3. 进 Bios 引导安装
4. 用 cfdisk 对空闲盘进行分区，swap、boot、btrfs
5. 划分子卷，挂载分区
6. 启动！

## 配置

配置还是比较伤脑筋的，首先是适应安装工具 **pacman**。大概就是一个官方提供的包管理工具，需要自己换源。后续还添加了yay、paru（第三方包库，资源丰富但是没有镜像站）、flatpak（体验比较好，速度能接受）。

然后是桌面环境。尝试了 gnome、hyprland、niri 以后，都感觉不太适应，还没有从 win 的操作模式转换过来。主要是各个组件都要自己配。从 waybar、swaybg，到 bluez、swaylock，有很多选择，又眼花缭乱。并且每有体系化的教学、整合，就感觉很乱。

比较喜欢 niri 的操作模式，觉得比较酷，就着重配置 niri 的环境了。比较麻烦的是截图，在 firefox 试了半天，发现 firefox 不能贴图……其实截图功能也已经比较完善了，基本能和 win 一样地操作。接下来是网络，被 yay、paru 的下载速度折磨了很久，clash 的机场又不太稳定了，不知道是哪里的问题。半天才发现 shell 需要额外配置代理……但速度还是不太能用，有机会再优化一下吧，先这样。

## 其他

其实 Linux 也就是一个操作系统，不可能换了个系统就一劳永逸了，还是会有很多问题。算是满足自己的好奇心，去尝试一下新的可能。其实确实能感受到 Linux 的环境已经很完善了，比如现在写博客的流程，就能从 Win 复刻到 Linux 上。可能需要的就是一些解决问题的耐心。

虽然把系统装好了，但是对原理没什么更多的理解，基本就是需求-碰到问题-搜索-解决问题，不能形成整体性的知识。既然装好了就试着用下去吧。

![image-20251103202847494](https://raw.githubusercontent.com/calendar0917/images/master/image-20251103202847494.png)  
