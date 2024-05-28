---
title: x0vnc/x11vnc通过ssh与osc52共享剪切板
date: 2024-05-28 22:46:13
tags:
 - linux
 - ssh
 - vnc
categories:
 - [linux, ssh]
 - [linux, vnc]
---
# 前言
没想到这次写文又又又又是因为剪贴板的事情。这次的场景是控制本地x11的x0vncserver下共享剪贴板。[archwiki](https://wiki.archlinux.org/title/TigerVNC#Running_x0vncserver_to_directly_control_the_local_display)没有写解决方案，而在该项目的[issue](https://github.com/TigerVNC/tigervnc/issues/529#issuecomment-1358864856)中，这位老哥`@mathewng`表示一种workaround，具体是多开一个带参数的无画面x11vnc会话捕抓键盘和剪贴板`x11vnc -nofb -clip 4x4+0+0`。

这意味着将开启两个本地x11的vnc会话，一个是一切正常除了剪贴板的`x0vncserver -rfbauth ~/.vnc/passwd`；另一个是啥都没有仅剩下捕抓键盘和剪贴板`x11vnc -nofb -clip 4x4+0+0`(BTW，这里为了前者的正常体验可以加上-repeat参数，就是有亿点点小问题)。

两个残疾人取长补短看起来很滑稽，但是确实很好地解决了两者的不足。唯一的缺点是无法将中文复制出来
> PS:x0vnc服务器中的rdp远程windows主机中文却能正确复制出来。鄙人由于不懂编码，觉得煞是奇怪

无法复制中文以汉语为主要语言的人无异于弃用。想起之前的x11转发和osc52，顿时心生一计——好像能通过bash脚本把两者结合在一起，以一种比较诡异的办法共享双方剪贴板

# 思路
通过ssh上的x11转发加上指定的配置能获得指定x11会话，在这个基础上就可以做到通过ssh控制指定的x11会话。例如像这样通过ssh在x11本地会话下打开`xclock`:
<img src='https://dlink.host/1drv/aHR0cHM6Ly8xZHJ2Lm1zL2kvcyFBckVNT01Ec2ZXcEdnVGtuQmdqcnZDZVFsUzBEP2U9U3dQanBI.png' alt=''>
而剪贴板是由x11提供的，那么说在ssh中提取或者操作剪贴板内容也不在话下了。很明显，`xclip`可以通过命令行胜任这个工作:
```bash
➜  ~ pacman -Ss xclip
extra/xclip 0.13-5 [installed]
    Command line interface to the X11 clipboard
#  example:
➜  ~ echo '我要被复制了' | xclip -i -r -selection clipboard # 复制到clipboard
➜  ~ xclip -o -selection clipboard # 输出clipboard内容
我要被复制了
```
直到这里也仅限于将指定x11会话的内容在ssh中输出出来，或者将ssh中输入的内容通过echo\管道\重定向等方法复制到指定x11会话的剪贴板中，不能单纯通过命令行真正做到双向粘贴

**但是**，osc52和终端软件配合就能真正做到只通过bash的命令行分享x0vnc服务器的剪贴板到ssh客户端机器，并且只通过bash命令更改x0vnc剪贴板内容

能通过命令行的事情自然能写脚本替代手打，脚本能通过快捷键以及各种方法快速执行。所以笔者将以一个简单的脚本实现标题目的

> 这里的复制演示仅仅只代表clipboard剪贴板，linux有三种剪贴板，详见以下链接:
>
>https://superuser.com/questions/90257/what-is-the-difference-between-the-x-clipboards
>
>https://unix.stackexchange.com/questions/139191/whats-the-difference-between-primary-selection-and-clipboard-buffer

# 先决条件
- ssh使用的`终端`与x0vncserver服务主机使用的`终端`支持`osc52转义序列`
- ssh服务器开启`X11Forwarding`,ssh客户端开启`ForwardX11`和`ForwardX11Trusted`(两个选项可以用open-ssh命令行参数`-Y`替代)
# 实践
## 1-脚本目录结构
文件一览:
```bash
➜  remote_clip ls -hl
total 24K
-rw-r--r-- 1 baka baka  37 May 29 01:08 clipboard
-rwxr-xr-x 1 baka baka 131 May 28 15:12 evnc.sh
-rwxr-xr-x 1 baka baka 131 May 28 00:51 sc.sh
-rwxr-xr-x 1 baka baka 131 May 27 23:16 vc.sh
-rwxr-xr-x 1 baka baka 142 May 27 23:31 wvnc.sh
-rw-r--r-- 1 baka baka 668 May 28 22:44 yank_head
```
- `clipboard`提供写入x0vncserver会话剪贴板的内容
- `evnc.sh`提供使用默认文件编辑器打开`clipboard`文件的方法
- `sc.sh`将x0vncserver会话的剪贴板的内容复制到ssh客户端机器(蹩脚英文ssh copy)
- `vc.sh`将`clipboard`内容复制到x0vncserver会话剪贴板(蹩脚英文vnc copy)
- `wvnc.sh`将传入的第一个参数写入到`clipboard`
-  `yank_head`上述脚本的实现方法
## 2-脚本内容
`yank_head`上述脚本的实现方法:
```bash
#!/usr/bin/bash

# OSC52YANK WORKAROUND FOR X0VNCSERVER VIA SSH
# YANK TO SSH CLIENT HOST FROM X CLIPBOARD

# X11Forwarding INITE
export DISPLAY=:0
xauth add $(xauth list $DISPLAY)

yank_to_client()
{
  echo -ne "\033]52;c;$(xclip -o | base64 )\a"
	echo "YANKED"
}

# YANK TO VNC SERVER HOST FROM "$yk_path/clipboard"
yank_to_server()
{
  echo -ne "\033]52;c;$(cat "$yk_path/clipboard" | base64 )\a"
	notify-send "VNC SERVER REV CLIPBOARD"
}

# WRITE SOMETHING TO "$yk_path/clipboard" 
open_clipboard()
{
  echo $input
  echo "$input" > "$yk_path/clipboard"
	echo "WRITTEN"
}


edit_clipboard()
{
#  TO WRITE SOMETHING VIA EDITOR
  $EDITOR "$yk_path/clipboard"
}
```
`evnc.sh`提供使用默认文件编辑器打开`clipboard`文件的方法:
```bash
#!/usr/bin/bash

# PATH TO SCRIPT DIR
yk_path=$(dirname $(realpath "${BASH_SOURCE[0]}"))
cd $yk_path

. ./yank_head
edit_clipboard
```
`sc.sh`将x0vncserver会话的剪贴板的内容复制到ssh客户端机器(蹩脚英文ssh copy):
```bash
#!/usr/bin/bash

# PATH TO SCRIPT DIR
yk_path=$(dirname $(realpath "${BASH_SOURCE[0]}"))
cd $yk_path

. ./yank_head
yank_to_client
```
`vc.sh`将`clipboard`内容复制到x0vncserver会话剪贴板(蹩脚英文vnc copy):
```bash
#!/usr/bin/bash

# PATH TO SCRIPT DIR
yk_path=$(dirname $(realpath "${BASH_SOURCE[0]}"))
cd $yk_path

. ./yank_head
yank_to_server
```
`wvnc.sh`将传入的第一个参数写入到`clipboard`:
```bash
#!/usr/bin/bash

# PATH TO SCRIPT DIR
yk_path=$(dirname $(realpath "${BASH_SOURCE[0]}"))
cd $yk_path

. ./yank_head
input="$1"
open_clipboard
```
## 3-食用方法
```bash
- evnc.sh
- 在ssh中
- 直接执行
- 使用默认文本编辑器打开文件clipboard
- end


- wvnc.sh 
- 在ssh中
- 执行wvnc.sh '要复制的内容'
- 将'要复制的内容'写入clipboard
- 写入内容不包含引号且内容不能含有
- 单引号
- end

- sc.sh
- 在ssh中
- 直接执行
- 将x0vncserver服务器中剪贴板的内容
- 复制到ssh客户端的主机
- end

- vc.sh
- 在x0vncserver中的终端
- 直接执行
- 将文件clipboard中的内容
- 复制到x0vncserver服务器中
- vnc
```
如果要将vc.sh放到图形界面的快捷键中，请务必使用终端执行此命令