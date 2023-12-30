---
title: nmcli基本使用方法
date: 2023-12-24 21:49:58
tags:
- almalinux
- centos
- linux
- network
- nmcli
- networkmanager
categories:
- [linux, networkmanager]
- [linux, nmcli]
- [network, networkmanager]

---

> 参考:
> 
> [配置和管理网络 Red Hat Enterprise Linux 9 | Red Hat Customer Portal](https://access.redhat.com/documentation/zh-cn/red_hat_enterprise_linux/9/html/configuring_and_managing_networking/index)
> 
> [第 22 章 配置 NetworkManager DHCP 设置 Red Hat Enterprise Linux 9 | Red Hat Customer Portal](https://access.redhat.com/documentation/zh-cn/red_hat_enterprise_linux/9/html/configuring_and_managing_networking/configuring-networkmanager-dhcp-settings_configuring-and-managing-networking)

最近因为一些作业，有必要使用fedora系发行版。目前在象牙塔里教学的大多是cent os，但是由于众所周知的原因，我选择尝试almalinux 9。在这个发行版里网络配置工具是NetworkManager，和笔者日用的发行版不一样。即使曾经用过，以前用带有gui环境的其他发行版时也是相当的用户友好，基本不必读文档。但是在轻量安装的时候，没有gui配置工具下，就很有必要读文档了。此文用以记录在vbox中配置almalinux网络时所涉及的基本使用方法

> 在vbox容器中对虚拟机配置多网卡前，务必保证增强功能已经安装，否则无法成功对相关网卡进行配置

# 1、查看网卡基本情况

一般来说对网卡进行简单的配置，比较常见的的参数就是`connection`:

```bash
$ nmcli -h
Usage: nmcli [OPTIONS] OBJECT { COMMAND | help }

OPTIONS
  -a, --ask                                ask for missing parameters
  -c, --colors auto|yes|no                 whether to use colors in output
  -e, --escape yes|no                      escape columns separators in values
  -f, --fields <field,...>|all|common      specify fields to output
  -g, --get-values <field,...>|all|common  shortcut for -m tabular -t -f
  -h, --help                               print this help
  -m, --mode tabular|multiline             output mode
  -o, --overview                           overview mode
  -p, --pretty                             pretty output
  -s, --show-secrets                       allow displaying passwords
  -t, --terse                              terse output
  -v, --version                            show program version
  -w, --wait <seconds>                     set timeout waiting for finishing operations

OBJECT
  g[eneral]       NetworkManager's general status and operations
  n[etworking]    overall networking control
  r[adio]         NetworkManager radio switches
  c[onnection]    NetworkManager's connections
  d[evice]        devices managed by NetworkManager
  a[gent]         NetworkManager secret agent or polkit agent
  m[onitor]       monitor NetworkManager changes
```

<br>

<br>

例如要查看当前网卡的配置文件可以:

```bash
 nmcli connection show
NAME    UUID                                  TYPE      DEVICE
enp0s3  a3aaeb51-2a34-3ace-b48f-3d99c7f87415  ethernet  enp0s3
enp0s8  ca02b8ec-fc8b-3c78-9f90-4697654bd7bb  ethernet  enp0s8
lo      29bf7cf7-8148-46ed-985a-1a4c9a2dbf8c  loopback  lo

 # enp0s3为某张网卡设备
nmcli connection show enp0s8
```

输入以上代码块最后一条命令后将弹出来指定网卡当前的配置文件情况(即使配置未激活，这条命令仅仅只是直接读取配置文件情况)

另一种方法读取配置文件则是直接查看配置目录`/etc/NetworkManager/system-connections/`，同样查看enp0s3配置文件情况:

```bash
cat /etc/NetworkManager/system-connections/enp0s8.nmconnection
[connection]
id=enp0s8
uuid=ca02b8ec-fc8b-3c78-9f90-4697654bd7bb
type=ethernet
autoconnect-priority=-999
interface-name=enp0s8
timestamp=1703218246

[ethernet]

[ipv4]
address1=192.168.56.50/24
dhcp-timeout=30
method=manual

[ipv6]
addr-gen-mode=eui64
dhcp-timeout=30
method=auto

[proxy]
```

以上配置一样不代表当前网卡状态，可能修改配置后它并没有被激活

> 此配置文件的路径仅代表net-tools被弃用后的格式

# 2、更改网卡的当前配置文件并激活

## 配置

更改配置文件一样有两种方法，一种是通过命令方式，一般使用`nmcli connection modify`，一种是写入配置文件例如我要为我的`enp0s8`网卡添加一个ip `192.168.56.51/24` :

```bash
# 记得加上掩码的前缀
nmcli connection modify enp0s8 +ipv4.addresses 192.168.56.51/24
```

查看网卡配置文件是否被更改:

```bash
cat /etc/NetworkManager/system-connections/enp0s8.nmconnection
[connection]
id=enp0s8
uuid=ca02b8ec-fc8b-3c78-9f90-4697654bd7bb
type=ethernet
autoconnect-priority=-999
interface-name=enp0s8
timestamp=1703218246

[ethernet]

[ipv4]
address1=192.168.56.50/24
address2=192.168.56.51/24    #这里多了一行
dhcp-timeout=30
method=manual

[ipv6]
addr-gen-mode=eui64
dhcp-timeout=30
method=auto

[proxy]
```

用`ip address`查看网卡情况可以发现配置文件修改后并不会自动激活:

```
3: enp0s8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:6d:6c:09 brd ff:ff:ff:ff:ff:ff
    inet 192.168.56.50/24 brd 192.168.56.255 scope global noprefixroute enp0s8
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:fe6d:6c09/64 scope link noprefixroute
       valid_lft forever preferred_lft forever
```

> 直接编辑配置文件而不用命令的话需要重启NetworkManager服务再激活配置，否则配置并不会生效

## 激活

激活配置文件使用这句命令:

```bash
nmcli connection up enp0s8
Connection successfully activated (D-Bus active path: /org/freedesktop/NetworkManager/ActiveConnection/5)
```

查看网卡状态:

```bash
3: enp0s8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:6d:6c:09 brd ff:ff:ff:ff:ff:ff
    inet 192.168.56.50/24 brd 192.168.56.255 scope global noprefixroute enp0s8
       valid_lft forever preferred_lft forever
    inet 192.168.56.51/24 brd 192.168.56.255 scope global secondary noprefixroute enp0s8
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:fe6d:6c09/64 scope link noprefixroute
       valid_lft forever preferred_lft forever
```

# 总结

nmcli还有更多功能，例如为单张网卡创建多个名称不同的配置文件、修改 NetworkManager后端dhcp客户端为dhclient (**dhcpcd不支持**) 等等
