---

title: su或sudo下使用neovim剪切板
date: 2024-02-08 18:49:40
tags:
 - linux
 - nvim
 - ssh
categories:
 - [linux, nvim]
 - [linux, ssh]

---

本文承接[ssh下使用nvim复制到系统剪切板](https://7cmb.com/ssh%E4%B8%8B%E4%BD%BF%E7%94%A8nvim%E5%A4%8D%E5%88%B6%E5%88%B0%E7%B3%BB%E7%BB%9F%E5%89%AA%E8%BE%91%E6%9D%BF/)。目的是在ssh切换用户后也能和本地机器共用剪切板。本文将在以上链接配置的基础下继续

# 概述

在上文中即使正确配置了服务器与客户端的ssh配置，在切换用户后也无法使用x应用。而笔者使用的nvim剪切板是通过x实现的，这就十分地蛋疼，特别是需要调试系统服务的时候。根据实践，ssh用户登陆并切换账户后`DISPLAY`变量是存在的，但是x用不了一点，根据搜索，是`xauth`没有正确配置导致的      <del>怎么又是你</del> 

# xauth配置

xauth通过cookie认证，所以需要把ssh登陆用户的cookie写入到需要切换的用户xauth配置文件中(先决条件:ssh客户地端设置x11Forwarding，按需设置X11Trusted)

```bash
# 查看当前登陆用户的DISPLAY变量的cookie
$ xauth list $DISPLAY
routerAlmaLinux/unix:11  MIT-MAGIC-COOKIE-1  一串神秘数字

# 以root为例，写入以上结果
$ sudo xauth add routerAlmaLinux/unix:11  MIT-MAGIC-COOKIE-1  一串神秘数字
```

这样就能以root身份在本地使用远程x应用了

# 自动配置

需要登陆远程主机后自动配置需要使用x应用的其他用户只需要在远程shell的启动文件写入上述命令，以bash为例，文件位于`~/.bashrc`:

```bash
ugly=$(xauth list $DISPLAY)
sudo xauth add $ugly
unset ugly
```
