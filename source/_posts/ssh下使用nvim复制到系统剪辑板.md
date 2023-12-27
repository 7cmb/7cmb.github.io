---
title: ssh下使用nvim复制到系统剪辑板
date: 2023-12-27 14:13:12
tags:
 - linux
 - nvim
 - ssh
categories:
 - [linux, nvim]
 - [linux,ssh]
---

在DE环境下使用ssh复制到系统剪辑版十分方便，在WM却十分蛋疼.....ssh下使用终端下的文本编辑器不搞定复制问题可以说是痛不欲生了

在neovim中有多种解决系统剪辑版的方案，可以用`:help provider-clipboard`查看解决官方提供的方法。本文将使用`xsel`和`xclip`解决这个问题

# 一、思路

nvim不直接连接系统剪辑板，首先需要安装`xsel`和`xclip`之后在nvim配置文件加入`set clipboard+=unnamedplus`这一行以配置完成**nvim连接本地系统剪辑版**

`xsel`和`xclip`是运行在x服务上的软件，在x服务器上显然是可以使用的。ssh服务器提供x转发，所以可以在远程服务器上使用这两款软件，再利用x转发到本地x服务器以获取远程服务器上nvim复制的内容到本地剪辑板

但是事情并没有这么简单，即使开启ssh的x转发，本地x服务器能使用远程服务器的x软件了，但是远程nvim的内容还是死活复制不上。经过一轮搜索，发现其中还有一个叫`xhost`的x服务使用的认证软件

`xhost`是一个适用于单用户环境的为x服务器提供隐私控制和安全措施的软件。他有一个表来控制使用x服务器的客户端，以使得本地软件更安全。也就是说x转发过程中，除了ssh认证以外，x服务器本身也有一个认证

而ssh客户端配置中有一个叫`ForwardX11Trusted`的选项，对应参数`-Y`。指定这个参数将信任远程x客户端，即远程客户端将不再通过x服务器的认证

> 参考:
> 
> [x11 - ssh, is better to use -X or -Y? - Unix & Linux Stack Exchange](https://unix.stackexchange.com/questions/619083/ssh-is-better-to-use-x-or-y)
> 
> [x11 - What does this `xhost ...` command do? - Unix & Linux Stack Exchange](https://unix.stackexchange.com/questions/177557/what-does-this-xhost-command-do)
> 
> [Xhost - Arch Linux 中文维基](https://wiki.archlinuxcn.org/wiki/Xhost)

# 二、配置

## ssh客服端

添加以下配置到ssh客户端配置文件以跳过x认证

对于全局配置`/etc/ssh/ssh_conf`:

```
# 星号替换为需要配置的主机名或者ip，这是通配符
Host *
   ForwardAgent no
   ForwardX11 yes
   ForwardX11Trusted yes
```

对于个人配置`~/.ssh/config`:

```
ForwardAgent no
ForwardX11 yes
ForwardX11Trusted yes
```

> 不配置ssh客户端而改为用xhsot添加指定远程主机到控制列表里也行

## ssh服务端

安装x转发必要包及`xsel`、`xclip`:

```
# 红帽为例
sudo dnf install xorg-x11-xauth xorg-x11-fonts-\* xorg-x11-utils dbus-x11 xsel xclip
```

添加此行到`/etc/ssh/sshd_config`:

```
X11Forwarding yes
```

然后编辑nvim配置文件，对于个人配置 `~/.config/nvim/init.vim`,对于全局配置`$VIM/sysinit.vim`:

```
# 添加这一行
set clipboard+=unnamedplus
```

自此，ssh客户端可使用远程nvim复制到本地剪辑板
