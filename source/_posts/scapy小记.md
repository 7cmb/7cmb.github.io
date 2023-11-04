---
title: scapy小记
date: 2023-11-04 14:35:50
tags:
 - network
 - scapy 
 - python
categories:
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

# 1、[构造数据包](https://scapy.readthedocs.io/en/latest/usage.html#first-steps)

```
raw(IP())
'E\x00\x00\x14\x00\x01\x00\x00@\x00|\xe7\x7f\x00\x00\x01\x7f\x00\x00\x01'

IP(_)
<IP version=4L ihl=5L tos=0x0 len=20 id=1 flags= frag=0L ttl=64 proto=IP
 chksum=0x7ce7 src=127.0.0.1 dst=127.0.0.1 |>

 a=Ether()/IP(dst="www.slashdot.org")/TCP()/"GET /index.html HTTP/1.0 \n\n"

 hexdump(a)
00 02 15 37 A2 44 00 AE F3 52 AA D1 08 00 45 00  ...7.D...R....E.
00 43 00 01 00 00 40 06 78 3C C0 A8 05 15 42 23  .C....@.x<....B#
FA 97 00 14 00 50 00 00 00 00 00 00 00 00 50 02  .....P........P.
20 00 BB 39 00 00 47 45 54 20 2F 69 6E 64 65 78   ..9..GET /index
2E 68 74 6D 6C 20 48 54 54 50 2F 31 2E 30 20 0A  .html HTTP/1.0 .
0A                                               .

b=raw(a)

b
'\x00\x02\x157\xa2D\x00\xae\xf3R\xaa\xd1\x08\x00E\x00\x00C\x00\x01\x00\x00@\x06x<\xc0
 \xa8\x05\x15B#\xfa\x97\x00\x14\x00P\x00\x00\x00\x00\x00\x00\x00\x00P\x02 \x00
 \xbb9\x00\x00GET /index.html HTTP/1.0 \n\n'

c=Ether(b)

c
<Ether dst=00:02:15:37:a2:44 src=00:ae:f3:52:aa:d1 type=0x800 |<IP version=4L
 ihl=5L tos=0x0 len=67 id=1 flags= frag=0L ttl=64 proto=TCP chksum=0x783c
 src=192.168.5.21 dst=66.35.250.151 options='' |<TCP sport=20 dport=80 seq=0L
 ack=0L dataofs=5L reserved=0L flags=S window=8192 chksum=0xbb39 urgptr=0
 options=[] |<Raw load='GET /index.html HTTP/1.0 \n\n' |>>>>
```

We see that a dissected packet has all its fields filled. That’s because
 I consider that each field has its value imposed by the original 
string.If this is too verbose, the method hide_defaults() will delete every field that has the same value as the default:

```
c.hide_defaults()

c
<Ether dst=00:0f:66:56:fa:d2 src=00:ae:f3:52:aa:d1 type=0x800 |<IP ihl=5L len=67
 frag=0 proto=TCP chksum=0x783c src=192.168.5.21 dst=66.35.250.151 |<TCP dataofs=5L
 chksum=0xbb39 options=[] |<Raw load='GET /index.html HTTP/1.0 \n\n' |>>>>
```

<br>

此段我理解为c数据包的解包之所以显示所有字段，是因为作者提取了默认字段的字符串值（根据文档，字符串也可以作为自定义层的字段，不一定是值），并且用该值**覆盖**了一次默认值，所以作者最后的c数据包所有字段可见。也就是说初始化默认值后，该值将在数据包的默认方法里显示，否则在默认方法中隐藏未初始化的值。

<br>

而且在构建数据包的对象中有一方法objet.hide_defaults()用以隐藏和默认值相同的字段

这是测试代码：

```
>>> a=Ether()/IP()
>>> a
<Ether  type=IPv4 |<IP  |>>
>>> a=(raw(a))
>>> a
b'\xff\xff\xff\xff\xff\xff\x00\x00\x00\x00\x00\x00\x08\x00E\x00\x00\x14\x00\x01\x00\x00@\x00|\xe7\x7f\x00\x00\x01\x7f\x00\x00\x01'
>>> a=Ether(_)
>>> a
<Ether  dst=ff:ff:ff:ff:ff:ff src=00:00:00:00:00:00 type=IPv4 |<IP  version=4 ihl=5 tos=0x0 len=20 id=1 flags= frag=0 ttl=64 proto=hopopt chksum=0x7ce7 src=127.0.0.1 dst=127.0.0.1 |>>
>>> raw(a)
b'\xff\xff\xff\xff\xff\xff\x00\x00\x00\x00\x00\x00\x08\x00E\x00\x00\x14\x00\x01\x00\x00@\x00|\xe7\x7f\x00\x00\x01\x7f\x00\>>> a.hide_defaults()
>>> a.hide_defaults()
>>> a
<Ether  dst=ff:ff:ff:ff:ff:ff src=00:00:00:00:00:00 type=IPv4 |<IP  ihl=5 len=20 chksum=0x7ce7 src=127.0.0.1 dst=127.0.0.1 |>>
>>>
x00\x01'
```

<br>

<br>

对于数据包的值，我猜是通过字典的方法存储的（不通过字典访问将修改object默认方法的最外层字段），可以通过如下例子修改:

```
>>> a
<Ether  dst=00:00:00:00:00:00 src=00:00:00:00:00:00 type=IPv4 |<IP  ihl=5 len=20 chksum=0x7ce7 src=127.0.0.1 dst=127.0.0.1 |>>
>>> a[IP].dst='192.168.1.10'
>>> a
<Ether  dst=00:00:00:00:00:00 src=00:00:00:00:00:00 type=IPv4 |<IP  ihl=5 len=20 chksum=0x7ce7 src=127.0.0.1 dst=192.168.1.10 |>>
>>>
```

> 还可以通过笛卡尔积的方法批量生成数据包，具体通过迭代器和列表，详见[Usage &mdash; Scapy 2023.10.20 documentation](https://scapy.readthedocs.io/en/latest/usage.html#generating-sets-of-packets)

<br>

<br>

这是笛卡尔积批量生成包的例子，我猜第一行的`/30`应该类似于掩码，如不指定只能获取一个ip，对于域名来说，加了掩码也不一定能获取准确ip，我认为这更倾向一种参数之类的东西，用以指定一个ip所在某个掩码范围

```
    print(i)的所有地址:
```

[管理 GitHub Pages 站点的自定义域 - GitHub 文档](https://docs.github.com/zh/pages/configuring-a-custom-domain-for-your-github-pages-site/managing-a-custom-domain-for-your-github-pages-site#configuring-an-apex-domain) 这是github的文档，用以对照以下代码最后一行的ip

```
>>> a=IP(dst="www.slashdot.org/30")
>>> [i for i in a]
[<IP  dst=216.105.38.12 |>, <IP  dst=216.105.38.13 |>, <IP  dst=216.105.38.14 |>, <IP  dst=216.105.38.15 |>]
>>> a=IP(dst="github.io/30")
>>> [i for i in a] //可见只有第二个ip是准确的，以2^2位子网长度，这是第39个子网
[<IP  dst=185.199.111.152 |>, <IP  dst=185.199.111.153 |>, <IP  dst=185.199.111.154 |>, <IP  dst=185.199.111.155 |>]
```

# 2、[收发数据包](https://scapy.readthedocs.io/en/latest/usage.html#send-and-receive-packets-sr)

发送数据包大概有两种形式，一种是只发，一种是收发。这里先说只发函数的用法。

<br>

<br>

对于只发的数据包，一种函数使send()一种是sendp()。send()作用于第三层，只需指定三层的参数，对于下层数据将自动处理；sendp()则需手动指定二层数据，需要指定网卡，以及二层协议。以下是官方文档示例:

```
send(IP(dst="1.2.3.4")/ICMP())
.
Sent 1 packets.

sendp(Ether()/IP(dst="1.2.3.4",ttl=(1,4)), iface="eth1")
....
Sent 4 packets.

sendp("I'm travelling on Ethernet", iface="eth1", loop=1, inter=0.2)
................^C
Sent 16 packets.

sendp(rdpcap("/tmp/pcapfile")) # tcpreplay
...........
Sent 11 packets.

Returns packets sent by send()

send(IP(dst='127.0.0.1'), return_packets=True)
.
Sent 1 packets.
<PacketList: TCP:0 UDP:0 ICMP:0 Other:1>
```

<br>

<br>

对于收发数据包一般使用sr()函数，此函数将发送目标数据包并返回一对数据包及其应答，包括无应答包。sr1()函数是一种变体，将发送目标数据包并只返回一个接收方用以应答的数据包。srp()是用以二层的收发函数。无应答时，当超时一个空数据包将被分配以取代超时信息。

```
>>> p=sr1(IP(dst="www.slashdot.org")/ICMP()/"XXXXXXXXXXX")
Begin emission:
...Finished to send 1 packets.
.*
Received 5 packets, got 1 answers, remaining 0 packets
>>> p
<IP version=4L ihl=5L tos=0x0 len=39 id=15489 flags= frag=0L ttl=42 proto=ICMP
 chksum=0x51dd src=66.35.250.151 dst=192.168.5.21 options='' |<ICMP type=echo-reply
 code=0 chksum=0xee45 id=0x0 seq=0x0 |<Raw load='XXXXXXXXXXX'
 |<Padding load='\x00\x00\x00\x00' |>>>>
>>> p.show()
---[ IP ]---
version   = 4L
ihl       = 5L
tos       = 0x0
len       = 39
id        = 15489
flags     =
frag      = 0L
ttl       = 42
proto     = ICMP
chksum    = 0x51dd
src       = 66.35.250.151
dst       = 192.168.5.21
options   = ''
---[ ICMP ]---
   type      = echo-reply
   code      = 0
   chksum    = 0xee45
   id        = 0x0
   seq       = 0x0
---[ Raw ]---
      load      = 'XXXXXXXXXXX'
---[ Padding ]---
         load      = '\x00\x00\x00\x00'
```

<br>

<br>

> 下面找到熟肉了：
> 
> https://wizardforcel.gitbooks.io/scapy-docs/content/3.html
> 
> 直接copy

发送和接收函数族是scapy中的核心部分。它们返回一对两个列表。第一个就是发送的数据包及其应答组成的列表，第二个是无应答数据包组成的列表。为了更好地呈现它们，它们被封装成一个对象，并且提供了一些便于操作的方法：

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

# 3、[SYN 扫描](https://scapy.readthedocs.io/en/latest/usage.html#syn-scans)

以下是一个经典的SYN扫描初始化案例:

```
sr1(IP(dst="72.14.207.99")/TCP(dport=80,flags="S"))
```

以上代码执行后将向谷歌发送一个SYN包并在接受一个应答后结束：

```
Begin emission:
.Finished to send 1 packets.
*
Received 2 packets, got 1 answers, remaining 0 packets
<IP  version=4L ihl=5L tos=0x20 len=44 id=33529 flags= frag=0L ttl=244
proto=TCP chksum=0x6a34 src=72.14.207.99 dst=192.168.1.100 options=// |
<TCP  sport=www dport=ftp-data seq=2487238601L ack=1 dataofs=6L reserved=0L
flags=SA window=8190 chksum=0xcdc7 urgptr=0 options=[('MSS', 536)] |
<Padding  load='V\xf7' |>>>
```

从以上的输出中可以看出，Google返回了一个SA（SYN-ACK）标志位，表示80端口是open的。

使用其他标志位扫描一下系统的440到443端口：

```
>>> sr(IP(dst="192.168.1.1")/TCP(sport=666,dport=(440,443),flags="S"))
```

或者

```
>>> sr(IP(dst="192.168.1.1")/TCP(sport=RandShort(),dport=[440,441,442,443],flags="S"))
```

<br>

<br>

在多数据包对象中可以使用make_table()方法来建立表格：

```
> > > ans,unans = sr(IP(dst=["192.168.1.1","yahoo.com","slashdot.org"])/TCP(dport=[22,80,443],flags="S"))
> > > Begin emission:
> > > .......*.**.......Finished to send 9 packets.
> > > **.*.*..*..................
> > > Received 362 packets, got 8 answers, remaining 1 packets
> > > ans.make_table(
> > > ...    lambda(s,r): (s.dst, s.dport,
> > > ...    r.sprintf("{TCP:%TCP.flags%}{ICMP:%IP.src% - %ICMP.type%}")))
> > >     66.35.250.150                192.168.1.1 216.109.112.135
> > > 22  66.35.250.150 - dest-unreach RA          -
> > > 80  SA                           RA          SA
> > > 443 SA                           SA          SA
```

# 结语

大概到这就能写个TCP端口扫描/SYN洪水工具了。
