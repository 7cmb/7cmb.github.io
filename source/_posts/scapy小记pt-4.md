---
title: scapy小记pt.4-嗅探
date: 2023-11-29 19:28:42
tags:
 - documentation
 - network
 - scapy 
 - python
categories:
 - [documentation, scapy]
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

在scapy中嗅探用的函数主要是sniff()，分析、过滤数据包主要依靠sniff()中传入的参数，以及scapy自带的其他模块，本文将以访问的web服务器的数据包为例，记录sniff()的简单使用方法

## 1、参数

这是sniff()帮助手册中关于参数的内容:

```
sniff(*args, **kwargs)
    Sniff packets and return a list of packets.

    Args:
        count: number of packets to capture. 0 means infinity.
        store: whether to store sniffed packets or discard them
        prn: function to apply to each packet. If something is returned, it
             is displayed.
             --Ex: prn = lambda x: x.summary()
        session: a session = a flow decoder used to handle stream of packets.
                 --Ex: session=TCPSession
                 See below for more details.
        filter: BPF filter to apply.
        lfilter: Python function applied to each packet to determine if
                 further action may be done.
                 --Ex: lfilter = lambda x: x.haslayer(Padding)
        offline: PCAP file (or list of PCAP files) to read packets from,
                 instead of sniffing them
        quiet:   when set to True, the process stderr is discarded
                 (default: False).
        timeout: stop sniffing after a given time (default: None).
        L2socket: use the provided L2socket (default: use conf.L2listen).
        opened_socket: provide an object (or a list of objects) ready to use
                      .recv() on.
        stop_filter: Python function applied to each packet to determine if
                     we have to stop the capture after this packet.
                     --Ex: stop_filter = lambda x: x.haslayer(TCP)
        iface: interface or list of interfaces (default: None for sniffing
               on all interfaces).
        monitor: use monitor mode. May not be available on all OS
        started_callback: called as soon as the sniffer starts sniffing
                          (default: None).

    The iface, offline and opened_socket parameters can be either an
    element, a list of elements, or a dict object mapping an element to a
    label (see examples below).
```

在这些参数中，主要用到的有`iface` `filter` `lfilter` `count` `prn` <br>

<br>

`iface`用以指定嗅探网卡，不指定则默认一个，具体忘了，官方文档有写，`conf.iface`变量能指定当前全局的网卡配置；
`filter`是伯克利套接字过滤器，能在TCP/IP模型内一到四层进行简单过滤，这里是随便找的参考文档[IBM QRadar Security Intelligence Platform](https://www.ibm.com/docs/en/qsip/7.4?topic=queries-berkeley-packet-filters)；

`lfilter`和之前的用法一样，对每个数据包进行过滤并筛选出符合条件的数据包保存；

`count`为指定保存数据包的个数，保存到了这么多个数据包就会自动停止；

`prn`为嗅探过程中，需要打印的内容，针对嗅探到且过滤出来的每一个数据包，如果你传入的数据返回了值将会被打印出来；<br>

<br>

对不同层数据包结构有了解的前提下，有了这些概念就能简单的筛选需要的数据包，配合scapy的其他模块，甚至能对raw的内容进行分析（其实保存下pcap文件后交由其他软件分析也可以对raw分析）

下面将在命令行模式下编写代码，以实现监听本机所访问过的web服务器及其对应的域名(仅限不使用代理下访问过的80和443端口的web服务器)

## 2、‘http’及‘tls’模块的加载和使用

需要完成上述需求，则需要对借助scapy提供的模块对应用层及传输层协议能解析的内容进行拓展，分别是`http`(仅限http1.X)和`tls`模块<br><br>

### http

这个模块载入有两种方法:

```python
#1、仅限交互模式，及控制台下使用
load_layer("http")
#2、怎么用都行
from scapy.layers.http import *
```

这是官方api参考文档[scapy.layers.http &mdash; Scapy 2.5.0 documentation](https://scapy.readthedocs.io/en/latest/api/scapy.layers.http.html)，在执行了以上两段代码之一后，就能对http文本进行分析:

```python
>>> load_layer("http") #tls模块提前加载过了，只加载http模块无法运行下面一行代码
>>> plist=sniff(filter="tcp dst port 80 or 443", lfilter=lambda x:((x.haslayer(TLS_Ext_ServerName) or x.haslayer(HTTPRequest))==True),count=10,prn=lambda x:x.summary())
Ether / IP / TCP / TLS 172.16.27.191:36870 > 36.158.231.204:443 / TLS / TLS Handshake - Client Hello
Ether / IP / TCP / HTTP / 'GET' '/' 'HTTP/1.1'
Ether / IP / TCP / HTTP / 'GET' '/s?z=people&c=9' 'HTTP/1.1'
Ether / IP / TCP / HTTP / 'GET' '/s?z=people&c=10' 'HTTP/1.1'
Ether / IP / TCP / HTTP / 'GET' '/s?z=people&c=11' 'HTTP/1.1'
Ether / IP / TCP / HTTP / 'GET' '/s?z=people&c=1' 'HTTP/1.1'
Ether / IP / TCP / HTTP / 'GET' '/s?z=people&c=2' 'HTTP/1.1'
Ether / IP / TCP / HTTP / 'GET' '/s?z=people&c=101' 'HTTP/1.1'
Ether / IP / TCP / HTTP / 'GET' '/GB/399635/index.html' 'HTTP/1.1'
Ether / IP / TCP / HTTP / 'GET' '/s?z=people&c=3' 'HTTP/1.1'
>>> plist[1]
<Ether  dst=20:76:93:52:c9:1f src=bc:f1:71:35:69:1b type=IPv4 |<IP  version=4 ihl=5 tos=0x0 len=594 id=65455 flags=DF frag=0 ttl=64 proto=tcp chksum=0x8fb4 src=172.16.27.191 dst=120.232.104.138 |<TCP  sport=35810 dport=www_http seq=3233705982 ack=127270836 dataofs=5 reserved=0 flags=PA window=502 chksum=0xab86 urgptr=0 |<HTTP  |<HTTPRequest  Method='GET' Path='/' Http_Version='HTTP/1.1' Accept='text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8' Accept_Encoding='gzip, deflate' Accept_Language='en-US,en;q=0.7,zh-CN;q=0.3' Connection='keep-alive' Cookie='wdcid=503f838c92509908; sso_c=0; wdlast=1701253257; wdses=5bd8a7f4e328acf2' Host='www.people.com.cn' If_Modified_Since='Wed, 29 Nov 2023 10:01:03 GMT' If_None_Match='W/"65670bdf-1bfa9"' Referer='https://www.bing.com/' Upgrade_Insecure_Requests='1' User_Agent='Mozilla/5.0 (X11; Linux x86_64; rv:120.0) Gecko/20100101 Firefox/120.0' |>>>>>
```

太难看了，贴张图片:

<img title="" src="https://telegraph.7cmb.com/file/0fcb4427553316beb96fb.png" alt="sniff1" >

可见本来应该为field应该为Raw的数据可以解析出来了，并且可以通过字典类型访问，pkt["key"].field:

```python
>>> plist[1][HTTPRequest].Host
b'www.people.com.cn'
>>> plist[1][HTTPRequest].Host.decode()
'www.people.com.cn'
```

如上就提取了访问的域名信息

### tls

在tls加密的数据包之中，可以在tcp三次握手后，进行tls四次握手过程中的第一次握手`Client Hello`数据包中探测`SNI`信息，因为历史原因，这个字段不会加密(现在可以了，即`ESNI`)。这个field将指向即将访问的主机名。在使用tls模块之前，要用pip安装另一个模块详见官方api文档[scapy.layers.tls package — Scapy 2.5.0 documentation](https://scapy.readthedocs.io/en/latest/api/scapy.layers.tls.html#module-scapy.layers.tls):

```bash
#也可用系统自带的包管理安装
pip install cryptography
```

之后加载模块:

`load_layer("tls")`

再进行嗅探:

```python
>>> plist.nsummary()    #该对象有Client Hello包，所以不再抓包，直接使用http阶段的结果
0000 Ether / IP / TCP / TLS 172.16.27.191:36870 > 36.158.231.204:443 / TLS / TLS Handshake - Client Hello
0001 Ether / IP / TCP / HTTP / 'GET' '/' 'HTTP/1.1'
0002 Ether / IP / TCP / HTTP / 'GET' '/s?z=people&c=9' 'HTTP/1.1'
0003 Ether / IP / TCP / HTTP / 'GET' '/s?z=people&c=10' 'HTTP/1.1'
0004 Ether / IP / TCP / HTTP / 'GET' '/s?z=people&c=11' 'HTTP/1.1'
0005 Ether / IP / TCP / HTTP / 'GET' '/s?z=people&c=1' 'HTTP/1.1'
0006 Ether / IP / TCP / HTTP / 'GET' '/s?z=people&c=2' 'HTTP/1.1'
0007 Ether / IP / TCP / HTTP / 'GET' '/s?z=people&c=101' 'HTTP/1.1'
0008 Ether / IP / TCP / HTTP / 'GET' '/GB/399635/index.html' 'HTTP/1.1'
0009 Ether / IP / TCP / HTTP / 'GET' '/s?z=people&c=3' 'HTTP/1.1'
```

分析:

<img title="" src="https://telegraph.7cmb.com/file/2173aada94c83659aa827.png" alt="sniff2">

至此，一个简单的分析完成
