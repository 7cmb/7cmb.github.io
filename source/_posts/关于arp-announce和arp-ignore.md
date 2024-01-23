---
title: 关于arp_announce和arp_ignore
date: 2024-01-22 17:15:59
tags:
 - linux
 - network
categories:
 - [ linux, network]
---
最近在看lvs相关的文档，配置过程中，对DR/TUN的HA,LB方案中控制系统行为的内核参数`arp_announce`和`arp_ignore`有一点不解。从前一直把交换过程中arp的源地址想得理所当然，所以对这两个参数需要一点时间来消化，本文记录一点感想

>本文参考:
>
>http://kb.linuxvirtualserver.org/wiki/Using_arp_announce/arp_ignore_to_disable_ARP
>
>https://www.kernel.org/doc/html/latest/networking/ip-sysctl.html

# 概述

## arp_announce
`arp_announce`对应本机生成对外广播arp request中，源ip地址的的选择。文件路径为`/proc/sys/net/ipv4/conf`的网卡配置文件中，在linux内核中应该取一个整数值:<br>

\- `0` 默认值，可以使用任意网卡的任意本地地址，使其作为arp request中的源地址，一般来说会根据上层数据包的源地址做选择<br>

\- `1` 尽量避免使用与目标主机不同子网的地址来作为arp request的源地址。在生成arp request的过程中，将检查是否有包含源地址和和目标主机的子网，如果有，将保留源地址，否则根据 `2` 来生成arp request的源地址<br>

\- `2` 使用最优地址，永远忽略上层数据包的源地址。根据目标主机地址选用最佳本地地址作为arp request的源地址<br>

## arp_ignore
`arp_ignore`对应本机收到`arp request`请求后，对于发送回应`arp reply`这个动作的策略。路径为`/proc/sys/net/ipv4/conf`的网卡配置文件中，在linux内核中应该取一个整数值:<br>

\- `0` 默认值，回应任何arp request，只要目标地址为任意本地网卡地址<br>

\- `1` 只回应arp request目标地址为本地接收网卡绑定地址的报文<br>

\- `2` 只回应arp request目标地址为本地接收网卡绑定地址的报文，并且arp request的源地址必须与接收网卡绑定地址处于一个子网<br>

\- `3` （这个没用上，选择性看不懂思密达）

\- `8` 不回应任意arp request<br>

# 思路
上述两个参数对应本地主机arp的请求和响应，两个直觉上本应该协同工作的功能却要分开配置（我猜除了配置类似lvs服务外还有是为了静态mac等需要安全的子网考虑）<br>

对于lvs的来说，上述两个参数的使用我的主观意见是为了分割逻辑通信和物理通信方式。逻辑上使用vip进行通信，实际路由交换使用各自的子网地址。vip的mac理论上应该只绑定活动的DR,所以通过禁止RS主机的vip参与实际交换过程来防止子网内各主机争夺vip的mac地址
> 莫名想到ibgp路由使用loopback通信的技巧。无需物理上建立直连关系，只需要逻辑上能连接便可建立对等体关系
>
> 说起路由和arp,想到的还有完全与此文无关的路由技巧-思科机器默认开启的arp代理，能在某些简单拓扑不配置默认网关的的情况下跨广播域通信，代价是路由器需要维护更大的mac表

修改的内核参数将作用于具体网卡。也就是说这些参数需要应用于RS实际通信的网卡而不是绑定vip的网卡<br>

下面将进行实验已验证参数是否生效

# 实验
 data1-server:`192.168.56.55/24`部署于vbox子网，在以下拓扑的vbox广播域:
<img title="" src="https://dlink.host/1drv/aHR0cHM6Ly8xZHJ2Lm1zL2kvcyFBckVNT01Ec2ZXcEdnUXVUREhwNXNBUkRoS1ljP2U9R2JBN29q.png" alt="">
本次实验将更改AlmaLinuxRouter的arp_ignore参数，并用data1-server对AlmaLinuxRouter发起arping以验证想法
> 对于如何验证arp_announce，暂时想不到直观的办法，此处忽略

当`/proc/sys/net/ipv4/conf/enp0s8/arp_ignore`为1:
<img title="" src="https://dlink.host/1drv/aHR0cHM6Ly8xZHJ2Lm1zL2kvcyFBckVNT01Ec2ZXcEdnUkt2TGppeUZqb0FibkVYP2U9aE85SGNG.png" alt="">

当`/proc/sys/net/ipv4/conf/enp0s8/arp_ignore`为0:
<img title="" src="https://dlink.host/1drv/aHR0cHM6Ly8xZHJ2Lm1zL2kvcyFBckVNT01Ec2ZXcEdnUkh2dF85Wm9hUTBma1Z6P2U9SzNCYjFt.png" alt="">

可以看到当接收网卡的"arp_ignore"为1时，即使arp请求目标地址为本机的其他网卡绑定的ip,也不会进行响应
