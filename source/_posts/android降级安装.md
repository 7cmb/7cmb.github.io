---
title: android降级安装
tags:
  - android
  - 折腾
categories:
  - - 折腾
    - android
date: 2024-03-25 17:38:36
---

# 动机

最近整了台17年的洋垃圾——Xperia XZ1Compact。很喜欢这台机器，想着在它EOL的这段最后的日子还能日常用用，便一口气整备到能用的状态

<img title="" src="https://dlink.host/1drv/aHR0cHM6Ly8xZHJ2Lm1zL2kvcyFBckVNT01Ec2ZXcEdnU3hDcmZ4QkhUTXBFS0dxP2U9TXZVdmRQ.jpg" alt="">

4G RAM 放在骁龙820的年代还算是中端，这台机子偏偏是835年代的机器。即使放在当年也是画面太美。解锁bl、root、刷上volte、融卡、划SWAP外总算是达到24年日常能用的水平<del>总算不杀后台了</del>，但是还是遭不住某绿色国产超级即时通信软件的折腾。这软件毫不夸张就是个性能杀手

重装后低版本后发现`XX版本过低，无法登录`顿时火冒三丈。查阅资料后发现有一个可行方案——高版本登陆后使用adb降级安装app

> 参考:
> 
> https://lmshuai.com/archives/472/
> 
> [Android root环境下设置全局可调试(ro.debuggable = 1) - 『移动安全区』 - 吾爱破解 - LCG - LSG |安卓破解|病毒分析|www.52pojie.cn](https://www.52pojie.cn/thread-1512749-1-1.html)
> 
> https://zhuanlan.zhihu.com/p/100583752

# 先决条件

- 设备已root，并安装管理器

- shamiko模块(不确定是否硬性条件)

- 一台能够adb的设备，只用本地终端模拟器实测不行

- 目标设备的selinux需要设置为宽容即permissive

# 实操

## 1-开启全局调试模式

打开adb命令行，如果多台设备请指定选项:

```powershell
PS C:\Users\whoami> adb shell
G8142:/ $ su
G8142:/ # magisk resetprop ro.debuggable 1
G8142:/ # stop;start;
```

完成上述操作之后设备将自动重启，进入调试模式

# 2-临时设置selinux为permissive

本人的设备因为要融卡，这行命令已经写进magisk的post脚本里了:

```powershell
PS C:\Users\whoami> adb shell
G8142:/ $ su
G8142:/ # setenforce 1
```

# 3-降级安装apk

准备apk到对应路径并开始保留数据降级安装:

```powershell
G8142:/ # cd /storage/E440-F2BE/base_apks/
G8142:/storage/E440-F2BE/base_apks # pm install -d -r ./green_shit.apk
```

这是会弹出selinux的avc权限提示，已经是宽容模式了理论上只会把上述操作记录在案，而不会阻挡，所以直接忽略:

```shell
avc:  denied  { read } for  scontext=u:r:system_server:s0 tcontext=u:object_r:sdcardfs:s0 tclass=file permissive=1
Success # 出现则代表安装完成
```

结束，这时打开app查看数据是否还在即可

# 版本

绿色国产超级即时通信软件版本记录:

7.0.15 Android Q及以上 有独立黑暗模式开关

7.0.17 Android O以上     也能开机黑暗模式

8.0.0   该版本才能在24年正常使用某些小程序

> 不升级连核心功能都不让用的东西，先进捏。。。
