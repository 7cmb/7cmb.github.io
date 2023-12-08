---
title: scapy小记pt.3-路由追踪
date: 2023-11-29 19:28:37
tags:
 - document
 - network
 - scapy 
 - python
categories:
 - [document, scapy]
 - [python, scapy]
 - [network, scapy]
---

> 本文内容来自：
> 
> [Welcome to Scapy’s documentation! &mdash; Scapy 2023.10.20 documentation](https://scapy.readthedocs.io/en/latest)
> 
> [GitHub - wizardforcel/scapy-docs-zh: :book: [译] Scapy 中文文档](https://github.com/wizardforcel/scapy-docs-zh)
> 
> 此文章用以记录一点小小的感想
> PS:中文文档年代过于久远，最后还得要啃生肉

本机默认路由

```
➜  ~ ip route
default via 192.168.10.254 dev wlan0 proto dhcp src 192.168.10.148 metric 3003
172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1 linkdown
192.168.10.0/24 dev wlan0 proto dhcp scope link src 192.168.10.148 metric 3003
```

<br>

对百度进行路由追踪:

```
>>>  a,u=sr(IP(dst="baidu.com",ttl=(1,30),id=RandShort())/TCP(flags="S"),timeout=2)
Begin emission:
Finished sending 30 packets.
.****************...
Received 20 packets, got 16 answers, remaining 14 packets
>>> for s,r in a:
...  print(s.ttl,r.src,isinstance(r.payload,TCP))
...
1 192.168.1.1 False
2 100.64.0.1 False
3 219.158.41.1 False
4 110.242.68.66 True
5 110.242.68.66 True
6 110.242.68.66 True
7 110.242.68.66 True
8 110.242.68.66 True
9 110.242.68.66 True
10 110.242.68.66 True
11 110.242.68.66 True
12 110.242.68.66 True
13 110.242.68.66 True
14 110.242.68.66 True
15 110.242.68.66 True
16 110.242.68.66 True
```

> 开始时很意外，为什么第一跳不是默认路由指向的地址，通过搜索发现有的路由器能设置不处理ttl字段以节省cpu资源；另外，id缺省值为1,看起来十分敷衍，所以收到的数据包id也为1......也不是不能用，不知道遇到分片的情况也这么敷衍会不会出问题；另外这个方法用数据是否有承载的包来判断是否到达目的主机，所以端口需要正确，否则大概率只能受到中间路由的信息

查看第一跳的请求及应答包:

```
>>> IP(raw(a[0][0])) //请求
<IP  version=4 ihl=5 tos=0x0 len=40 id=59707 flags= frag=0 ttl=1 proto=tcp chksum=0x5224 src=192.168.10.148 dst=110.242.68.66 |<TCP  sport=ftp_data dport=www_http seq=0 ack=0 dataofs=5 reserved=0 flags=S window=8192 chksum=0x110e urgptr=0 |>>
>>> a[0][1] //应答
<IP  version=4 ihl=5 tos=0xc0 len=68 id=40808 flags= frag=0 ttl=62 proto=icmp chksum=0x4fab src=192.168.1.1 dst=192.168.10.148 |<ICMP  type=time-exceeded code=ttl-zero-during-transit chksum=0xbf43 reserved=0 length=0 unused=0 |<IPerror  version=4 ihl=5 tos=0x0 len=40 id=20047 flags= frag=0 ttl=1 proto=tcp chksum=0xed10 src=192.168.10.148 dst=110.242.68.66 |<TCPerror  sport=ftp_data dport=www_http seq=0 ack=0 dataofs=5 reserved=0 flags=S window=8192
chksum=0xc555 urgptr=0 |>>>>
```

在应答包中，带error字段作为ICMP的子字段存在，此接收包在wireshark的格式是Frame/Ether/IP/ICMP，error字段记录了请求数据包的IP/TCP字段中的内容（除了部分非固定的字段，例如id）

再查看响应了TCP握手请求的应答包:

```
>>> IP(raw(a[3][0]))
<IP  version=4 ihl=5 tos=0x0 len=40 id=22742 flags= frag=0 ttl=4 proto=tcp chksum=0xdf89 src=192.168.10.148 dst=110.242.68.66 |<TCP  sport=ftp_data dport=www_http seq=0 ack=0 dataofs=5 reserved=0 flags=S window=8192 chksum=0x110e urgptr=0 |>>
>>> a[3][1]
<IP  version=4 ihl=5 tos=0x0 len=44 id=11561 flags= frag=0 ttl=49 proto=tcp chksum=0xde32 src=110.242.68.66 dst=192.168.10.148 |<TCP  sport=www_http dport=ftp_data seq=163716185 ack=1 dataofs=6 reserved=0 flags=SA window=8192 chksum=0xd6c1 urgptr=0 options=[('MSS', 536)] |<Padding  load='\x00\x00' |>>>
```

此段数据是一段正常发出的TCP请求，最终判断是看应答包是最外层是TCP协议，或者查看TCP字段里flags的值是否"SA"

# 总结

TCPtrace可规避开针对udp的防火墙设置，因为看起来是一坨发起正常TCP握手的，一般不会遭到防火墙阻挡。当超时将接收到对应收到ttl=1的路由器发回的ICMP报文，直到到达目的主机，且对端主机的端口需开放，否则只能接收到路由中节点的ICMP报文，无法断定发送的数据包是否已经被目标主机接收，在scapy中还有已经封装好的函数traceroute()，只需传入相关参数即可立刻进行TCP路由追踪；另外此路由追踪函数区别于其他路由追踪功能，它同时发送全部数据包，因此他的优点在于能在三秒内结束追踪（官方doc的例子），缺点就是它会跑完所有ttl再提供结果，因此无法预测结束时间；另外traceroute()所返回的值区别于普通的结果，这个结果专门用以追踪，能对更加深入的数据进行操作
