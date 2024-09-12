---
title: 只通过proc获取cpu负载和主要接口ip
tags:
  - linux
  - 折腾
categories:
  - - linux
    - 折腾
  - - 折腾
    - linux
date: 2024-09-12 14:40:30
---


# 前言

最近打算用k8s做一个demo,实现在web中查看节点系统信息。对于内容生成器，大概有三点需求:
- 1、fastfetch获取节点机器发行版及容器信息
- 2、获取节点机器从启动到当前时刻的cpu负载情况和当前时刻之前一定时间内的cpu负载情况
- 3、获取节点机器主要出口接口的ip地址以及该出口的详细信息

第一点很简单，只需要将容器宿主机器的`/etc/os-release`用hostpath的方式挂载到相同路径即可

第二点可以将容器所在宿主机器的`/proc`目录挂载(hostpath)到容器内的某个路径，再通过这个路径提供的接口写cpu usage的计算脚本

第三点也是需要容器所在宿主机器的`/proc`挂在到容器内的某个路径，之后通过这个路径提供的接口计算节点机器主要出口接口的ip地址以及相关信息

其中第三点需求对笔者来说十分具备挑战性，其中的内容值得单开一篇文章详细记录

> 本文不会出现容器编排的内容，所有操作在docker中完成

# 实践
## 1-fastfetch实现显示节点发行版信息
假设我有一台debian节点:
```bash
root@alice:~# uname -a
Linux alice 6.1.0-21-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.1.90-1 (2024-05-03) x86_64 GNU/Linux
```

我需要在里面跑archlinux的容器并且获取宿主机的发行版信息(将输出信息保存到当前目录output.txt):
```bash 
root@alice:~#  docker run --rm -it \
>  -v /etc/os-release:/etc/os-release:ro \
>  -e http_proxy=192.168.1.4:10810 \
>  -e https_proxy=192.168.1.4:10810 \
>  -e all_proxy=192.168.1.4:10810 \
>  --entrypoint '/bin/bash' \
>  archlinux/archlinux \
>  '-c' 'pacman -Sy --noconfirm;pacman -S --noconfirm fastfetch;fastfetch' > output.txt
```
第二行即是bind mount对应文件夹

查看对应输出内容可以发现得到的是宿主机发行版的信息:
```bash
root@alice:~# cat ./output.txt
:: Synchronizing package databases...
 core downloading...
 extra downloading...
resolving dependencies...
looking for conflicting packages...

Package (2)      New Version  Net Change  Download Size

extra/yyjson     0.10.0-1       0.59 MiB       0.13 MiB
extra/fastfetch  2.24.0-1       2.59 MiB       0.56 MiB

Total Download Size:   0.69 MiB
Total Installed Size:  3.18 MiB

:: Proceed with installation? [Y/n]
:: Retrieving packages...
 fastfetch-2.24.0-1-x86_64 downloading...
 yyjson-0.10.0-1-x86_64 downloading...
checking keyring...
checking package integrity...
loading package files...
checking for file conflicts...
:: Processing package changes...
installing yyjson...
installing fastfetch...
Optional dependencies for fastfetch
    chafa: Image output as ascii art
    dbus: Bluetooth, Player & Media detection [installed]
    dconf: Needed for values that are only stored in DConf + Fallback for
    GSettings
    ddcutil: Brightness detection of external displays
    directx-headers: GPU detection in WSL
    glib2: Output for values that are only stored in GSettings [installed]
    imagemagick: Image output using sixel or kitty graphics protocol
    libelf: st term font detection and fast path of systemd version detection
    [installed]
    libpulse: Sound detection
    mesa: Needed by the OpenGL module for gl context creation.
    libxrandr: Multi monitor support
    ocl-icd: OpenCL module
    hwdata: GPU output [installed]
    vulkan-icd-loader: Vulkan module & fallback for GPU output
    xfconf: Needed for XFWM theme and XFCE Terminal font
    zlib: Faster image output when using kitty graphics protocol [installed]
    libdrm: Displays detection
:: Running post-transaction hooks...
(1/1) Arming ConditionNeedsUpdate...
       _,met$$$$$gg.           root@f7a77c9144e6
    ,g$$$$$$$$$$$$$$$P.        -----------------
  ,g$$P"         """Y$$.".     OS: Debian GNU/Linux bookworm 12 x86_64
 ,$$P'               `$$$.     Kernel: Linux 6.1.0-21-amd64
',$$P       ,ggs.     `$$b:    Uptime: 100 days(!), 16 hours, 2 mins
`d$$'     ,$P"'   .    $$$     Packages: 121 (pacman)
 $$P      d$'     ,    $$$P    Shell: bash 5.2.32
 $$:      $.   -    ,d$$'      Terminal: xterm
 $$;      Y$b._   _,d$P'       Terminal Font: fixed (8.0pt)
 Y$$.    `.`"Y$$$$P"'          CPU: Intel(R) Celeron(R) G1610 (2) @ 2.60 GHz
 `$$b      "-.__               GPU: Intel Xeon E3-1200 v2/3rd Gen Core processor Graphicsontroller @ 1.05 GHz [Integrated]
  `Y$$                         Memory:  C1.46 GiB / 7.47 GiB (20%)
   `Y$$.                       Swap: 713.23 MiB / 976.00 MiB (73%)
     `$$b.                     Disk (/): 19.42 GiB / 291.41 GiB (7%) - overlay
       `Y$$b.                  Local IP (eth0): 172.17.0.2/16
          `"Y$b._              Locale: C.UTF-8
             `"""


```

这正是想要的结果

## 2-计算cpu usage

> 本节参考:
>
> [Linux Kernel Doc | 1.7 Miscellaneous kernel statistics in /proc/stat](https://docs.kernel.org/filesystems/proc.html#miscellaneous-kernel-statistics-in-proc-stat)
>
> 当然还有一系列中文文章，首先表达感谢，其次此处略；笔者认为闻着味找到原文会更好

从`/proc/stat`找到所有cpu在各种工作中的时间总和。这个时间的解释和内核与操作系统有关，笔者才疏学浅，看不懂思密达。但是可以知道一点的是只要把这个各种工作时间的比例算出来，那么是能求出cpu负载情况的:

```bash
root@alice:~# grep -w cpu /proc/stat 
cpu  252807252 186645 8692808 1403268967 65964476 0 457941 0 0 0
```

这一行代表所有cpu在各种工作中时间的总和，每一列的字段代表在不同工作中耗费的时间；从左到右依次是:
- [第2列] - 用户态的普通进程执行所花费cpu时间
- 第3列 - nice值为负(优先级高)的进程所花费的cpu时间
- [第4列] - 内核态进程所花费的时间
- [第5列] - 空闲时间(原文是玩手指)
- 第6列 - 等待io的时间(不可靠的)
......

目前来说只需要`[第2列]`、`[第4列]`、`[第5列]`作为输入即可求出所需要的值，因此其他信息笔者选择略过(没学过操作系统)

这一行的格式很舒爽，明显可以用`awk`来处理这段数据，先算个开机到目前时刻的总负载情况:
```bash
grep -w cpu /proc/stat | awk \
'{ \
  u=$2+$4; \
  t=$2+4+$5; \
  print u/t*100"%" \
  }'
```

如果要算当前一秒:
```bash

# 下意识写法
((grep -w cpu /proc/stat);sleep 1;( grep -w cpu /proc/stat)) | awk \
'{ \
    if (NR==1) { \
      u0=$2+$4; \
      t0=$2+$4+$5; \
    } \
    else { \
      print ($2+$4-u0)*100/($2+$4+$5-t0)"%" \
    }\
}'


# stackoverflow抄来的，用一条并发pipe把东西串在一起
# 了，值得参考（并发pipe不适用于zsh），可读性也强很多
awk '{u=$2+$4; t=$2+$4+$5; if (NR==1){u1=u; t1=t;} else print ($2+$4-u1) * 100 / (t-t1) "%"; }' \
<(grep 'cpu ' /proc/stat) <(sleep 1;grep 'cpu ' /proc/stat)
```

## 3-计算接口ip
这个难点主要在于需要将人类做逻辑运算的步骤抽象到脚本里。这个实在是有点难为笔者了，但是下意识的思路还是有的,以下命令能把人类可读的所有接口ip及网段给显示出来:
```bash
cat /proc/net/fib_trie |\
 sed -e 's/+--//g' \
     -e 's/|--//g' \
     -e 's/^\ *//g' |\
grep '^[0-9]' | awk '{print $1}' | sort | uniq
```
然后就是通过内核路由表出口网段以人脑推测网段(以当前路由表中最后一条路由为例):
```bash
raw=$(cat /proc/net/route | tail -n1 | awk '{print $2}')

# awk写法，利用"$1=$1"重构awk读入的单条
# record(即$0)以达成将整条record的分隔
# 符替换成OFS变量指定的，有意思在这里记录
# 一下
for i in {${raw:6:2},${raw:4:2},${raw:2:2},${raw:0:2}};do
printf 'ibase=16;%s\n' $i | bc
done | xargs echo | awk 'BEGIN{OFS="."}{$1=$1;print}'

# 比较好理解的写法
for i in {${raw:6:2},${raw:4:2},${raw:2:2},${raw:0:2}};do
printf 'ibase=16;%s\n' $i | bc
done | xargs echo | tr ' ' '.'
```

接下来看看输出(不想截图了，tmux开了两个tab，代码块拉到右边凑合看看):
```bash
➜  ~ cat /proc/net/fib_trie |\                                │➜  ~ for i in {${raw:6:2},${raw:4:2},${raw:2:2},${raw:0:2}};do
 sed -e 's/+--//g' \                                          │printf 'ibase=16;%s\n' $i | bc
     -e 's/|--//g' \                                          │done | xargs echo | awk 'BEGIN{OFS="."}{$1=$1;print}'
     -e 's/^\ *//g' |\                                        │192.168.1.0
grep '^[0-9]' | awk '{print $1}' | sort | uniq                │➜  ~
0.0.0.0                                                       │
0.0.0.0/0                                                     │
127.0.0.0                                                     │
127.0.0.0/31                                                  │
127.0.0.0/8                                                   │
127.0.0.1                                                     │
127.255.255.255                                               │
172.16.0.0/14                                                 │
172.17.0.0                                                    │
172.17.0.0/31                                                 │
172.17.0.1                                                    │
172.17.255.255                                                │
172.18.0.0                                                    │
172.18.0.0/31                                                 │
172.18.0.1                                                    │
172.18.255.255                                                │
172.19.0.0                                                    │
172.19.0.0/31                                                 │
172.19.0.1                                                    │
172.19.255.255                                                │
192.168.1.0                                                   │
192.168.1.0/24                                                │
192.168.1.0/29                                                │
192.168.1.255                                                 │
192.168.1.4                                                   │
➜  ~                                                          │
[1] 0:zsh*                                                                                       "z-blacket-arch" 14:33 12-Sep-24
```

可以看到是可以做到把东西提出来的，但是这些是人类可读格式，点分十进制说到底还是二进制，把运算抽象出来确实没什么思路，晚点琢磨