---
title: x220解决wlan时延问题
tags:
  - network
  - nmcli
categories:
  - - linux
    - nmcli
date: 2024-04-22 00:36:49
---


之前x220跑windows还算勉强能用，除了需要耐心和屏幕瞎眼意外好像也没有太大的缺点。在debian11-xfce上不跑服务仅仅当作浏览器启动器也算体验良好

然而最近重新整备的时候却发现使本机运行debian12的情况下在内网中访问这台机器的无线网卡延时居然游离在1~300ms不等，这在内网算是天文数字了......换了第二张无线网卡，换网络服务后端也不灵，详情在:[关于x220掌托排线更换提问 - 经典ThinkPad专区 - 专门网](https://www.ibmnb.com/thread-2048807-1-1.html)。找不到解决方法后暂时搁置了一段时间

后来抱着试一试的心态从debian12-lxqt换了manjaro-cinnamon发现在某种特定配置下无线网卡就能回到一个相对稳定的状态。但我的需求在manjaro上存在某些我无法接受又找不到workaround和solution的问题，而且都用manjaro了我为什么不用他的上游archlinux呢？因此决定在archlinux live cd上再次debug无线网卡延时问题

> 参考:
> 
> [Is iwlwifi or iwldvm or wext the wireless driver? - Ask Ubuntu](https://askubuntu.com/questions/618283/is-iwlwifi-or-iwldvm-or-wext-the-wireless-driver)
> 
> [Fixing Intel Wi-Fi 6 AX200 latency and ping spikes in Linux &#8211; The Z-Issue](https://z-issue.com/wp/fixing-intel-wi-fi-6-ax200-latency-and-ping-spikes-in-linux/)
> 
> [wifi - Make &quot;iw wlan0 set power_save off&quot; permanent - Raspberry Pi Stack Exchange](https://raspberrypi.stackexchange.com/questions/96606/make-iw-wlan0-set-power-save-off-permanent)
> 
> [Reddit - Dive into anything](https://www.reddit.com/r/archlinux/comments/tbnyq4/iwd_device_wlan0_not_found_no_station_on_device/)

# 解决思路

## 1 - 排查后端

首先检查无线服务的后端。archlinux的livecd默认运行iwd；而manjaro-cinnamon的无线服务后端为wpa_supplicant，可以为NetworkManager控制。我的manjaro不存在这个问题，当然一把梭wpa_supplicant+NetworkManager了。这时踢到第一个铁板——把iwd服务停掉了无线网卡驱动出问题了，`ip l`不识别我的无线网卡了。无奈，再度google发现[Reddit - Dive into anything](https://www.reddit.com/r/archlinux/comments/tbnyq4/iwd_device_wlan0_not_found_no_station_on_device/):

```bash
rmmod iwlmvm
rmmod iwlwifi
modprobe iwlwifi
modprobe iwlmvm
```

尝试后网卡回来了，就是名字从wlan0变成了wlp3s0，连上wifi后测试问题依旧

## 2 - 排查内核参数

经过多轮搜索发现有部分老哥也有高延迟或者类似问题:

[High latency and sound distortion when using WiFi (X200t) - Thinkpads Forum](https://forum.thinkpads.com/viewtopic.php?t=120592)

[networking - Thinkpad x220 wireless network slows down after some time - Ask Ubuntu](https://askubuntu.com/questions/154916/thinkpad-x220-wireless-network-slows-down-after-some-time)

他们都指出了这个参数`11n_disable`。开启后问题依旧

## 3 - 问题源头

继续查阅资料发现了一点端倪[Fixing Intel Wi-Fi 6 AX200 latency and ping spikes in Linux &#8211; The Z-Issue](https://z-issue.com/wp/fixing-intel-wi-fi-6-ax200-latency-and-ping-spikes-in-linux/)。这篇博文跟我的情况几乎一模一样，尝试了同样的解决方法就可以了`iw wlan0 set power_save off`

关于这个配置的持久化可以参考这里[wifi - Make "iw wlan0 set power_save off" permanent - Raspberry Pi Stack Exchange](https://raspberrypi.stackexchange.com/questions/96606/make-iw-wlan0-set-power-save-off-permanent)

显然配置一个systemd unit作为脚本开机自启的方法有点笨重，并且我也懒得配置iwlwifi的模块参数。因此我决定用NetworkManager自带的nmcli工具来实现这个配置的持久化

# 解决手段

nmcli的connection配置中有一段:

```bash
root@archiso ~ # nmcli connection show myssid |grep 802-11-wireless.powersave
802-11-wireless.powersave:              2 (disable)
```

这个选项默认是3，只需要把他改成2并up一下即可解决这个问题:

```bash
root@archiso ~ # nmcli connection modify myssid 802-11-wireless.powersave 2
root@archiso ~ # nmcli connection up myssid
Connection successfully activated (D-Bus active path: /org/freedesktop/NetworkManager/ActiveConnection/7)
```

关于其他选项具体作用详细查看`nm-settings-nmcli`手册

> 尚未连接过的wifi没有相关配置文件，因此modify无法补全参数。建议先使用nmcli连接一次wifi生成connection文件再修改这个选项
> 
> 可以使用`nmcli device wifi connect myssid --ask`或者[添加connection文件](https://access.redhat.com/documentation/zh-cn/red_hat_enterprise_linux/8/html/configuring_and_managing_networking/proc_connecting-to-a-wifi-network-by-using-nmcli_assembly_managing-wifi-connections)以连接wifi

# 疑问

latency和delay的区别实在没分清，稍后解决
