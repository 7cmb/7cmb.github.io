---
title: 利用WSL2打造Arch全GUI环境
date: 2023-09-28 22:50:12
tags:
 - linux
 - wsl
 - arch
categories:
 - [linux, arch]
 - [linux, wsl]
---

此文记录了本人利用微软wsl2搭建linux工作环境的过程，此过程理论上可用于 **[nullpo-head / wsl-distrod](https://github.com/nullpo-head/wsl-distrod)** 上所支持的发行版。

# 1、先决条件

系统已开启hyper-v虚拟机

Windows 10 版本 2004 及更高版本（内部版本 19041 及更高版本）或 Windows 11 。具体请参阅微软[官方手册]([旧版 WSL 的手动安装步骤 | Microsoft Learn](https://learn.microsoft.com/zh-cn/windows/wsl/install-manual#step-3---enable-virtual-machine-feature))。

> ps: Windows 10好像仅限专业版

# 2、安装 WSL2 命令（以windows 10为例）

使用管理员权限打开 powershell 以启用“适用于 Linux 的 Windows 子系统”可选功能

```powershell
dism.exe /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart
```

<br>
<br>

启用虚拟机功能

```powershell
dism.exe /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart
```

之后重启设备以完成系统对wsl2的更新

<br>
<br>

在官网链接上下载wsl2内核更新包并安装（wsl和wsl2原理不同，所以需要下载内核，具体可自行查找资料）

[内核更新包](https://wslstorestorage.blob.core.windows.net/wslblob/wsl_update_x64.msi)

> wsl与wsl2差别的官网资料： [比较 WSL 版本 | Microsoft Learn](https://learn.microsoft.com/zh-cn/windows/wsl/compare-versions#wsl-in-the-microsoft-store)

<br>
<br>

将wsl2设为默认运行版本

```powershell
wsl --set-default-version 2
```

自此，系统对wsl2的支持就已经完成，接下来便是安装发行版。

# 3、安装带有systemd的linux发行版（以debian为例）

刚好需要一个debian用以学习，写博客顺便搭了.....此步骤什么发行版都一样

打开以下链接下载你所需要的最新release后解压顺带找到 [这里](https://images.linuxcontainers.org/images/) 以下载你所需要的发行版

[nullpo-head / wsl-distrod](https://github.com/nullpo-head/wsl-distrod)

<br>
<br>

找到 **distrod_wsl_launcher.exe** 并运行

<img title="" src="https://onedrive.live.com/embed?resid=466A7DECC0380CB1%21113&authkey=%21ABArN5SxpHZEbMc&width=1953&height=408" alt="index_of_files">

运行后将出现以下界面

<img title="distrods_eg" src="https://onedrive.live.com/embed?resid=466A7DECC0380CB1%21110&authkey=%21ACxQ0_GQhkoybk8&width=3465&height=1596" alt="">

此时选择 ***[1]*** 则使用本地文件，选择 ***[2]*** 则使用指定连接以下载对应发行版，一般选 ***[2]*** 就行，笔者因为网络问题故选 ***[1]*** 以下示例为使用本地文件的操作

<br>
<br>

输入文件路径，摁shift加右键即可选择复制路径，之后删除引号

<img title="distrods_input" src="https://onedrive.live.com/embed?resid=466A7DECC0380CB1%21111&authkey=%21AOmn_oqukjPqfiM&width=3465&height=1596" alt="">

> 系统包选择rootfs,内核已经安装所以不需要的完整系统

此时系统wsl2发行版已安装，接着设置此发行版用户名及密码

<img title="distrods_complete" src="https://onedrive.live.com/embed?resid=466A7DECC0380CB1%21112&authkey=%21AHwhi0nX2XD8pEs&width=3840&height=2040" alt="">

<br>
<br>

为了保障开机自启你的wsl2虚拟机，还需要一点工作，在 **虚拟机的bash** 内输入（此步报错请尝试重启设备后重试）

```bash
sudo /opt/distrod/bin/distrod enable --start-on-windows-boot
```

接下来输入你的root密码后win10将会弹窗让你输入微软账户密码，照例输上，自此，你的wsl2 distrods with systemd就能开机自启了。

<br>
<br>

最后一步就是打开虚拟机的端口转发服务，以ssh服务端口22为例，你需要在虚拟机的bash输入（copy的注释我就留着了

```bash
echo 22 | sudo tee /opt/distrod/conf/tcp4_ports  # update the portproxy.service's configuration
sudo systemctl enable --now portproxy.service  # enable and start it
```

> 根据官网的信息，该服务在win11上会有问题，具体表现在该服务不会随着windows开机自启，并表示将来会修复
> 
> Updated at 2023-09-28

<br>
<br>

此时无图形界面的wsl2的发行版已经安装完毕，接下来安装软件和图形界面的步骤就自由发挥了，假设你不想安装完整的图形界面，wsl2自带wslg，能直接开启对应软件的图形服务，如果你想拥有一个完整的图形界面，请自行安装对应软件，一千个人就有一千个不同的图形界面，此步忽略。

# 4、以 [DWM](https://dwm.suckless.org/) 为图形界面，部署vnc服务

假设到此，你的图形界面已经安装好了，是时候部署你的linux桌面了，理论上来说除了vnc还能用xdrp和xserver，但是xdrp过于卡顿，Xserver因为笔者的图形界面原因不便使用，故选择vnc作为图形界面的中转。

安装vnc服务端

```bash
sudo pacman -S tigervnc
```

<br>
<br>

安装完成后为当前用户设置密码

```bash
vncpasswd
```

## 修改配置文件（此步尤为重要，当时排查了半天问题最后发现自己是傻逼）

首先确认自己的系统有什么桌面环境 `ls /usr/share/xsessions` 记录下输出结果，例如我的是（自行编辑则参考 [Desktop entries](https://wiki.archlinux.org/title/Desktop_entries)）

```bash
➜  ~ ls /usr/share/xsessions
dwm.desktop
```

<br>
<br>

之后修改本地用户配置文件，路径在 `~/.vnc/config` 

```bash
## Default settings for VNC servers started by the vncserver service
#
# Any settings given here will override the builtin defaults, but can
# also be overriden by ~/.vnc/config and vncserver-config-mandatory.
#
# See HOWTO.md and the following manpages for more details:
#     vncsession(8) Xvnc(1)
#
# Several common settings are shown below. Uncomment and modify to your
# liking.

# session=gnome    //桌面
# securitytypes=vncauth,tlsvnc
# geometry=2000x1200    //分辨率
# localhost    //一定一定记得要删掉这行
# alwaysshared    //多用户
```

这是默认配置，session请参考上一步的结果，我的图形界面为dwm则修改为

```bash
## Default settings for VNC servers started by the vncserver service
#
# Any settings given here will override the builtin defaults, but can
# also be overriden by ~/.vnc/config and vncserver-config-mandatory.
#
# See HOWTO.md and the following manpages for more details:
#     vncsession(8) Xvnc(1)
#
# Several common settings are shown below. Uncomment and modify to your
# liking.

session=dwm
geometry=2560x1440
alwaysshared
```

<br>
<br>

接下来修改系统配置，路径在 `/etc/tigervnc/vncserver-config-defaults` ，系统配置会和本地用户配置一起读取，不进行配置将导致奇怪问题

```bash
## Default settings for VNC servers started by the vncserver service
#
# Any settings given here will override the builtin defaults, but can
# also be overriden by ~/.vnc/config and vncserver-config-mandatory.
#
# See HOWTO.md and the following manpages for more details:
#     vncsession(8) Xvnc(1)
#
# Several common settings are shown below. Uncomment and modify to your
# liking.

# session=gnome
# securitytypes=vncauth,tlsvnc
# geometry=2000x1200
# localhost
# alwaysshared
# $(ip addr |grep inet|grep eth0|awk -F '[/]' '{print $1}'|awk '{print $2}')

session=dwm
geometry=2560x1440
alwaysshared
```

<br>
<br>

然后在同目录下检查 `vncserver-config-mandatory` 确保没有localhost参数

接着编辑同目录下 `vncserver.users` 将用户和端口进行绑定

```bash
 TigerVNC User assignment
#
# This file assigns users to specific VNC display numbers.
# The syntax is <display>=<username>. E.g.:
#
# :2=andrew
# :3=lisa
  :1=alicemargandroid
```

这个例子表示第八行表示5900+1端口绑定用户为alicemargandroid

<br>
<br>

最后编辑的文件是 `/etc/X11/Xwrapper.config` ，修改为

```bash
allowed_users=anybody
needs_root_rights=no
```

自此，vnc配置文件修改完成

接着启动服务即可愉快连接

```bash
sudo systemctl enable vncserver@:1
sudo systemctl start vncserver@:1
```

直接使用loopback:5900+X连接

<img title="vnc1" src="https://onedrive.live.com/embed?resid=466A7DECC0380CB1%21118&authkey=%21AGuEcuN9uP5VMmw&width=2670&height=1905" alt="">

<img title="vnc2" src="https://onedrive.live.com/embed?resid=466A7DECC0380CB1%21119&authkey=%21APdERf-TzgC3sss&width=2565&height=1530" alt="">

这时你就会发现即使正确配置了fcitx输入法也可能在某些场景无法唤出，这时就又要修改配置文件了

# 5、修改 `~/.xprofile` 确保输入法正常运行

在 `~/.xprofile` 中添加如下几行

```bash
export INPUT_METHOD=fcitx5
export GTK_IM_MODULE=fcitx5
export QT_IM_MODULE=fcitx5
export XMODIFIERS=@im=fcitx5
```

自此，全gui环境配置完成

# 结语

这时你会发现俺声音服务呢，这个只需要配置个简单的声音转发，这个坑晚点填。
