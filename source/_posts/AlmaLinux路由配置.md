---
title: AlmaLinux路由配置
date: 2023-12-31 04:21:31
tags:
 - linux
 - network
 - centos
 - almalinux
categories:
- [linux, firewalld]
- [linux, nmcli]
---

# 前言

此文章搁置了一段时间，只因为少写了一条静态路由...如果实验环境为全套专业设备的话应该很快就能排查出障碍。傻瓜设备总是会让人觉得某些功能是理所当然，导致排障时会直接忽略.....

本片文章记录配置一台almalinux主机作为软路由。配置的机器为`AlmaLinux Router`，如以下拓扑所示。其为虚拟网络提供基本网络服务，pxe环境等等

实验环境:

```
路由:almalinux9
客户机:almalinux9
环境:vbox虚拟网络 家宽
宿主:windows10
默认用户:root
工具:firewalld networkmanager
```

拓扑:

<img title="" src="https://dlink.host/1drv/aHR0cHM6Ly8xZHJ2Lm1zL2kvcyFBckVNT01Ec2ZXcEdnUXVUREhwNXNBUkRoS1ljP2U9R2JBN29q.png" alt="">

目的:

- 为虚拟网路划分一个广播域，方便后续实验

- 暂时不nat不伪装，方便调试虚拟网络里的机器

- 虚拟网络正常上网

# 实践

## Step1-开启内核ipv4转发

根据`sysctl`的文档，修改以下任意文件以写入配置:

```
FILES
       /proc/sys
       /etc/sysctl.d/*.conf
       /run/sysctl.d/*.conf
       /usr/local/lib/sysctl.d/*.conf
       /usr/lib/sysctl.d/*.conf
       /lib/sysctl.d/*.conf
       /etc/sysctl.conf
```

添加以下配置于指定文件并激活:

```bash
echo "net.ipv4.ip_forward=1" > /etc/sysctl.d/95-IPv4-forwarding.conf
sysctl -p /etc/sysctl.d/95-IPv4-forwarding.conf
```

验证配置是否生效:

```bash
[root@routerAlmaLinux baka]# cat /proc/sys/net/ipv4/ip_forward
1    # "1"代表已开启否则为"0"
```

## Step2-将指定接口配置到firewalld预设的区域

执行此步骤前确保控制内核网络框架`netfilter` 的工具只有`firewalld`

 `iptables`、`firewalld`、`nftables`都是内核网络框架`netfilter`用户空间的工具。 
在rhel9中`iptables`工具使用 `nf_tables` 内核 API 而不是 `legacy` 后端。`nf_tables` API 提供了向后兼容性，以便使用 `iptables` 命令的脚本仍可在 Red Hat Enterprise Linux 上工作。对于新的防火墙脚本，红帽建议使用 `nftables`。

> 碎碎念:firewalld可比四表五链简单多了 
> 另外，对于iptables和nftables，之前一直疑惑为什么SNAT一定要放在postrouting链，明明放在prerouting链貌似可行。直到参考了[iptables - Why does SNAT happen in POSTROUTING chain and DNAT in PREROUTING chain? - Unix &amp; Linux Stack Exchange](https://unix.stackexchange.com/questions/280114/why-does-snat-happen-in-postrouting-chain-and-dnat-in-prerouting-chain)之后，发现自己忽视了插在本地属于nat网段的那张网卡

此步骤能使用的工具有`nmcli`和`firewall-cmd`，根据[2.5. 在连接配置文件文件中手动将区分配给网络连接 Red Hat Enterprise Linux 9 | Red Hat Customer Portal](https://access.redhat.com/documentation/zh-cn/red_hat_enterprise_linux/9/html/configuring_firewalls_and_packet_filters/manually-assigning-a-zone-to-a-network-connection-in-a-connection-profile-file_working-with-firewalld-zones)，使用`nmcli`效率更高

### nmcli修改接口区域到trusted

首先查看接口信息:

```bash
[root@routerAlmaLinux baka]# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:85:94:9e brd ff:ff:ff:ff:ff:ff
    inet 192.168.1.50/24 brd 192.168.1.255 scope global noprefixroute enp0s3
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:fe85:949e/64 scope link noprefixroute
       valid_lft forever preferred_lft forever
3: enp0s8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:6d:6c:09 brd ff:ff:ff:ff:ff:ff
    inet 192.168.56.50/24 brd 192.168.56.255 scope global noprefixroute enp0s8
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:fe6d:6c09/64 scope link noprefixroute
       valid_lft forever preferred_lft forever
```

需要配置的接口为enp0s3和enp0s8，分别将他们配置到firewalld预设的`trusted`区域。此区域默认允许一切连接，而同一个区域内转发默认是允许的

配置命令:

```bash
# 假设要使用的NetworkManager的配置文件还是默认名称
nmcli connection modify enp0s3 connection.zone trusted
nmcli connection modify enp0s8 connection.zone trusted
nmcli connection up enp0s3
nmcli connection up enp0s8
```

检查配置是否生效:

```bash
[root@routerAlmaLinux baka]# firewall-cmd --get-active-zones
trusted
  interfaces: enp0s3 enp0s8
```

### firewall-cmd修改接口到指定区域

[2.3. 将网络接口分配给区 Red Hat Enterprise Linux 9 | Red Hat Customer Portal](https://access.redhat.com/documentation/zh-cn/red_hat_enterprise_linux/9/html/configuring_firewalls_and_packet_filters/assigning-a-network-interface-to-a-zone_working-with-firewalld-zones)

## Step3-配置路由表

查看`AlmaLinux Router`路由表:

```bash
[root@routerAlmaLinux baka]# ip r
default via 192.168.1.1 dev enp0s3 proto static metric 100
192.168.1.0/24 dev enp0s3 proto kernel scope link src 192.168.1.50 metric 100
192.168.56.0/24 dev enp0s8 proto kernel scope link src 192.168.56.50 metric 101
```

基本来说只用设置默认网关，使用nmcli或者iproute2配置即可

为上级网关配置路由表:

```
 # 梅林固件web后台直接设定就完事了
 DST:192.168.56.0/24 NEXT_HOP:192.168.1.50 FLAGS:UG 
```

这时虚拟网络已经能和家宽内网双向连通及和互联网正常通信

# 测试

在`AlmaLinux Router`上运行嗅探域名的[scapy脚本(修改版本)](https://7cmb.com/%E5%88%A9%E7%94%A8python%E6%8A%93%E5%8F%96%E5%88%86%E6%9E%90%E6%95%B0%E6%8D%AE%E5%8C%85%E5%B9%B6%E5%AD%98%E5%85%A5mysql/)可以看到访问记录:

<img title="" src="https://dlink.host/1drv/aHR0cHM6Ly8xZHJ2Lm1zL2kvcyFBckVNT01Ec2ZXcEdnUW1KdHd1VFRSVzNvNFN6P2U9eHluVm0y.png" alt="">

 路由正常
