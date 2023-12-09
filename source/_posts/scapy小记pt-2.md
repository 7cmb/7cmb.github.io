---
title: scapy小记pt.2
date: 2023-11-04 14:36:19
tags:
 - documentation
 - network
 - scapy 
 - python
categories:
 - [documentation, scaoy]
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

# sr()函数补充

sr过后需要处理数据，处理数据就要知道此函数返回的数据类型：

The “send’n’receive” functions family is the heart of Scapy. They `return
 a couple of two lists`. The first element is a list of couples (packet 
sent, answer), and the second element is the list of unanswered packets.
 These two elements are lists, but they are wrapped by an object to 
present them better, and to provide them with some methods that do most 
frequently needed actions:

```
>>> sr(IP(dst="192.168.8.1")/TCP(dport=[21,22,23]))
Received 6 packets, got 3 answers, remaining 0 packets
(<Results: UDP:0 TCP:3 ICMP:0 Other:0>, <Unanswered: UDP:0 TCP:0 ICMP:0 Other:0>)
>>> ans,unans=_
>>> ans.summary()
IP / TCP 192.168.8.14:20 > 192.168.8.1:21 S ==> Ether / IP / TCP 192.168.8.1:21 > 192.168.8.14:20 RA / Padding
IP / TCP 192.168.8.14:20 > 192.168.8.1:22 S ==> Ether / IP / TCP 192.168.8.1:22 > 192.168.8.14:20 RA / Padding
IP / TCP 192.168.8.14:20 > 192.168.8.1:23 S ==> Ether / IP / TCP 192.168.8.1:23 > 192.168.8.14:20 RA / Padding
```

sr()函数是返回一对两个列表，在以上例子分别是ans应答和unans无应答，可见需要两个变量来接受其返回的两个列表。这一对列表的第一个元素（ans）是应答的n对两个元素的列表（又是列表），分别是发送的数据包和对应的应答数据包；而第二个元素则是未应答的数据包列表。以下是应答数据包的结构:

```
                list:sr() //tuple
                [0]:<answer>  // scapy.plist.SndRcvList
                [1]:<unanswer>   // scapy.plist.PacketList                 
               
                list:<answer>   // scapy.plist.SndRcvList
                [0]:a couple of two list // scapy.plist.QueryAnswer
                [1]:a couple of two list
                [2]:a couple of two list
                ...
                [n]  :a couple of two list   
                
                list:<answer>[index] // scapy.plist.QueryAnswer
                [0]:pkt sent    //pkt
                [1]:pkt receive    //pkt
                
```

以百度为例，发起半连接以测试:

```
>>> test=sr(IP(dst="baidu.com")/TCP(flags="S",dport=[80,443,666]),timeout=1)
Begin emission:
Finished sending 3 packets.
**..............
Received 16 packets, got 2 answers, remaining 1 packets
>>> test //sr()别名
(<Results: TCP:2 UDP:0 ICMP:0 Other:0>, <Unanswered: TCP:1 UDP:0 ICMP:0 Other:0>)
>>> test[0] //应答数据包的列表
<Results: TCP:2 UDP:0 ICMP:0 Other:0>
>>> test[0][0] //应答数据包的列表其中一个元素，比较冗长，还可再拆分为一对元素的列表
QueryAnswer(query=<IP  frag=0 proto=tcp dst=39.156.66.10 |<TCP  dport=https flags=S |>>, answer=<IP  version=4 ihl=5 tos=0x4 len=44 id=1 flags= frag=0 ttl=51 proto=tcp chksum=0x5652 src=39.156.66.10 dst=172.16.27.191 |<TCP  sport=https dport=ftp_data seq=4122746057 ack=1 dataofs=6 reserved=0 flags=SA window=8192 chksum=0x35e7 urgptr=0 options=[('MSS', 536)] |<Padding  load='7\\xa7' |>>>)
>>> test[0][0][0] //以发送的数据包
<IP  frag=0 proto=tcp dst=39.156.66.10 |<TCP  dport=https flags=S |>>
>>> test[0][0][1] //服务器应答的数据包
<IP  version=4 ihl=5 tos=0x4 len=44 id=1 flags= frag=0 ttl=51 proto=tcp chksum=0x5652 src=39.156.66.10 dst=172.16.27.191 |<TCP  sport=https dport=ftp_data seq=4122746057 ack=1 dataofs=6 reserved=0 flags=SA window=8192 chksum=0x35e7 urgptr=0 options=[('MSS', 536)] |<Padding  load='7\\xa7' |>>>
```

<br>

<br>

# sr()返回对象部分方法的使用

把sr()返回的对象一个个变成字符串在利用循环语句和控制语句来查找目标数据包效率太低了，同一个行为可以慢两秒（使用scapy提供的方法和自己瞎写的SYN扫描脚本对比），所以我认为要掌握scapy的高效用法，一点学习成本是必要的。

## (scapy.plist.SndRcvList instance).summary()方法

这个方法的使用，老实说对一个python新手来说比较反直觉，为什么这个方法传递的参数这么奇怪，这玩意到底是怎么传递的......但是人既然这样写了，用就完了，这是用户手册，为每个数据包打印摘要（所以snd和rcv是怎么传递的啊喂）：

```
Help on method summary in module scapy.plist:

summary(prn=None, lfilter=None) method of scapy.plist.SndRcvList instance
    prints a summary of each packet

    :param prn: function to apply to each packet instead of
                lambda x:x.summary()
    :param lfilter: truth function to apply to each packet to decide
                    whether it will be displayed
(END)
```

这是示例代码：

```
ans.summary( lambda s,r: r.sprintf("%TCP.sport% \t %TCP.flags%") )
440      RA
441      RA
442      RA
https    SA
```

> 胡言乱语：
> 
> 既然手册写着是为每scapy.plist.SndRcvList实例的每一个数据包打印一份摘要，而且注明了“通过一个简单的循环”，那么是否可以理解为这个实例中的每一个元素先轮流传入这个lambda里，lambda通过两个s，r参数将传入的元素拆为两个IP数据包类型，针对单个数据包进行操作，之后再通过lambda返回的单个参数形成 一个新的scapy.plist.QueryAnswer或者scapy.plist.PacketList集合，最后再通过该实例的summary()方法形成一个新集合的摘要

## (scapy.plist.SndRcvList instance).nsummary()方法

同上方法，但是会显示数据包序号：

```
ans.nsummary(lfilter = lambda s,r: r.sprintf("%TCP.flags%") == "SA")
0003 IP / TCP 192.168.1.100:ftp_data > 192.168.1.1:https S ======> IP / TCP 192.168.1.1:https > 192.168.1.100:ftp_data SA
```

## (scapy.plist.SndRcvList instance).nsummary()和(scapy.plist.SndRcvList instance).summary()方法的两个参数

这两个方法有两个参数可以传入，一个是prn,另一个则是lfilter,，它们传入的方法都是通过lambda的方式，都具有过滤功能。区别在于prn与其说是过滤，它更像取对应变量的值再输出，也能输出额外内容；而lfilter用于过滤条件，筛选出符合条件的数据包的所有数据：

```
>>> a=sr(IP(dst="baidu.com")/TCP(dport=[80,443,666],flags="S"),timeout=1)[0]
Begin emission:
Finished sending 3 packets.
**.............
Received 15 packets, got 2 answers, remaining 1 packets
>>> a.summary(lambda s,r : r.sprintf("%TCP.flags%")=="SA")
True
True
>>> a.summary(lfilter=lambda s,r : r.sprintf("%TCP.flags%")=="SA")
IP / TCP 192.168.1.36:ftp_data > 110.242.68.66:www_http S ==> IP / TCP 110.242.68.66:www_http > 192.168.1.36:ftp_data SA
IP / TCP 192.168.1.36:ftp_data > 110.242.68.66:https S ==> IP / TCP 110.242.68.66:https > 192.168.1.36:ftp_data SA
```

这两个参数也能混合使用，先用lfilter参数过滤符合条件的数据包，然后再佐以prn参数过滤需要的字符串：

```
>>> a.summary(lfilter=lambda s,r:r.sprintf("%TCP.sport%")=="https")
IP / TCP 192.168.1.36:ftp_data > 110.242.68.66:https S ==> IP / TCP 110.242.68.66:https > 192.168.1.36:ftp_data SA
>>> a.summary(prn=lambda s,r:r.sprintf("%IP.dst%"),lfilter=lambda s,r:r.sprintf("%TCP.sport%")=="https")
192.168.1.36
```

## (scapy.plist.SndRcvList instance).filter()方法的使用

此方法用以过滤lambda函数传入的条件后，返回数据包列表：

```
>>> a.filter(lambda s,r:TCP in r and r[TCP].flags&2)
<filtered Results: TCP:2 UDP:0 ICMP:0 Other:0>
```

> `flags&2`代表的是flags位“S”的Hex值，如果换成`&18`将是“SA”的Hex值，wireshark抓包也能确定

可以通过filter返回的新数据包列表重新制表，这个表代表首先过滤本机接收的应答数据包中发送标志位为"S"的数据包后，再在制表的方法中过滤这些数据包发送人的端口及地址，并自动为相应的端口标记"🪝"标记：

```
>>> a.filter(lambda s,r:TCP in r and r[TCP].flags&2).make_table(lambda s,r:(s.dst,s.dport,'🪝'))
    110.242.68.66
80  🪝
443 🪝
```
