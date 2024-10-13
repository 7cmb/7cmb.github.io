---
title: w11利用lxc/lxd的rootfs导入发行版到wsl2
tags:
  - linux
  - wsl
  - 折腾
categories:
  - - linux
    - wsl
  - - linux
    - 折腾
date: 2024-10-14 00:34:15
---


本文记录本人利用lxc/lxd里的rootfs导入任意发行版到wsl2所踩的坑

> 这里"利用"一词应该符合"leverage"这个词的语境，最近对这个词印象比较深刻

# 前言

最近喜提新工作，也拿到了新的办公电脑。在windows不知道有什么好用的文本编辑器，也懒得折腾windows上的neovim，遂打算把配置搬到wsl2里(这个9p性能是差，但是哥们的需求就打几个字而已)。之前装wsl2都是依靠脚本大幅度简化安装流程的，这次打算看看微软的doc自己来一次

借用[lxc/lxd里的rootfs](https://images.linuxcontainers.org/images/)再加上[微软的doc](https://learn.microsoft.com/en-us/windows/wsl/use-custom-distro)很快就起了，但是难绷的是开了微软的systemd支持后wsl2启动后的二十秒内就会断网。且无论是mirrored还是nat的情况，检查网络配置后都查不出端倪......但是网络栈就是有问题，ping loopback都不通

# 排障过程

用以前的脚本装了一下一样的发行版发现也有一样的问题，不用systemd就没问题了。<del>即使systemd多么不好，作为linux的beginner也不想在没有准备的情况下换一个init</del>

检查hyper-v vds无果后百思不得其解下装了微软商店的ubuntu对比了一下systemd-networkd的配置文件，发现我用的debian多了个配置文件:
```bash
baka@z-x13co-debian:/etc/systemd/network$ ls
eth0.network
```
检查这个配置文件貌似也没什么不对的，抱着试一试的心删了发现一切正常了......
