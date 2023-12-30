---
title: Pulseaudio实现linux的声音转发
date: 2023-09-29 23:08:26
tags:
 - linux
categories:
 - [ linux, pulseaudio]
---

此文章用以记录wsl2部署vnc后没有音频的解决方法，虽然个人认为我的wsl2环境下没有音频甚至没有完整桌面都无伤大雅，毕竟只是用作写写配置文件和博客，但是本着做事做全套的作风，还是给加上了音频（即使我的gui 2k分辨率下帧率只有几十）........ <br>

<br>

## 1、安装pulseaudio

[Windows版本下载链接](https://www.freedesktop.org/wiki/Software/PulseAudio/Ports/Windows/Support/) ，下载完随便找个地解压就行了。linux照例，相信安装过arch的老哥应该不会对陌生 [PulseAudio - ArchWiki](https://wiki.archlinux.org/title/PulseAudio) ，为了控制音量顺带把pulseaudio-alsa和alsa-utils也给装了。

## 2、编辑服务端配置文件

windows的软件为**服务端**，此时linux的软件为**客户端**，注意区分。

首先编写**服务端**的配置文件，路径在 `pulseaudio-1.1/etc/pulse`（因为编码问题，所以这是linux的文件格式下的相对路径，正确写法略），找到对应词条，照例更改

`daemon.conf`

```注意前面的注释标志
exit-idle-time = -1
```

<br>

<br>

`default.pa`

```
load-module module-native-protocol-tcp auth-ip-acl=客户端机器ip
```

> 注意前面的注释是否删掉

## 3、编辑客户端配置文件

然后编写**客户端**配置文件，路径在 `/etc/pulse` 

`daemon.conf`

```
exit-idle-time = -1
```

<be>

<br>

最后添加当前环境转发的环境变量，我是xrog环境下的vnc，所以路径是X服务的`~/.xprofile`,理论上修改`/etc/environment`也可以

```bash
export HOST_IP="$(ip route | awk '$1=="default" {print $3}')"
export PULSE_SERVER="tcp:$HOST_IP"
```

此时已经完成转发，如果想要先行测试请在gui的终端下输入以上两端命令，之后打开服务端`bin`目录下的的`pulseaudio.exe`后在刚刚输入命令的终端窗口输入`speaker-test`即可。

> 注意客户端和服务端必须路由可达，且已防火墙已正确配置

## 结尾

测试通过后就能直接打开服务端的pulseaudio.exe后放在后台，享受声音转发服务了

<video src='https://dlink.host/1drv/aHR0cHM6Ly8xZHJ2Lm1zL3YvcyFBckVNT01Ec2ZXcEdlU19tUzBqNnRHODVtdkE_ZT1xZ3RQMWY.mp4' type='video/mp4' controls='controls'  width='100%' height='100%'>
</video>
