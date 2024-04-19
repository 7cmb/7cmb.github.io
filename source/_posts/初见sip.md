---
title: 初识sip
tags:
  - network
  - sip
categories:
  - - network
    - sip
date: 2024-04-19 15:23:39
---


> 本文跳过注册保活等基本操作，默认通话双方已注册且能呼叫
> 
> 本文<del>复制粘贴</del>参考:
> 
> [技术解码 | GB28181/SIP/SDP 协议]( https://zhuanlan.zhihu.com/p/545703291)
> 
> [SIP 中的Dialog，call，session 和 transaction - 简书](https://www.jianshu.com/p/84d7289a4d3b)
> 
> [SIP Tutorial](https://www.tutorialspoint.com/session_initiation_protocol/index.htm)
> 
> [Difference between session, dialog and transaction in SIP? - Stack Overflow](https://stackoverflow.com/questions/35133331/difference-between-session-dialog-and-transaction-in-sip)
> 
> [Call-ID and Branch tags in SIP protocol - Stack Overflow](https://stackoverflow.com/questions/10610194/call-id-and-branch-tags-in-sip-protocol)
> 
> [一次完整的通话过程SIP报文分析-腾讯云开发者社区-腾讯云](https://cloud.tencent.com/developer/article/1828603)

# 1 - 概述

SIP（Session initialization Protocol，**会话初始协议**）是由IETF（Internet Engineering Task Force，因特网工程任务组）制定的多媒体通信协议。

它是一个基于文本的应用层控制协议，用于创建、修改和释放一个或多个参与者的会话。SIP
 是一种源于互联网的IP 语音会话控制协议，具有灵活、易于实现、便于扩展等特点。SIP（Session Initiation 
Protocol）是一种类似于http协议的纯文本应用层协议。SIP可以用来控制会话的建立、取消、关闭等操作。主要可以实现以下功能：

- **用户定位：** 检查终端用户的位置，用于通信；
- **用户有效性：** 检查用户参与会话的意愿程度；
- **用户能力：** 检查媒体和媒体参数；
- **建立会话：** “振铃”，在呼叫和被叫方同时建立会话的参数；
- **会话管理：** 包括会话的传输和终止，修改会话参数以及请求服务

# 2 - request method

SIP REQUEST用以建立一段communication。为了补全这个communication的过程，无论成功或者失败，总会生成SIP RESPONSE以对应SIP REQUEST

在SIP的REQUEST中，核心的方法（CORE METHOD）定义了6种：INVITE、ACK、BYE、CANCEL、OPTIONS和REGISTER。

- INVITE消息用于发起一个新的会话；
- ACK消息用于完成会话的建立；
- BYE消息用于结束一个会话；
- CANCEL消息用于取消一个请求（一般是针对INVITE）；
- OPTIONS消息用于查询服务器的能力；
- REGISTER消息用于发送注册请求消息。

SIP请求的类型，也称作SIP方法。RFC3261 中定义了六种方法(CORE METHOD)。另外八种方法(EXTEND METHOD)有独立的RFC扩展描述。如INFO、NOTIFY等等

> METHOD也能被认为是REQUEST，它指明了对其他USER AGENT或者SERVER的具体行动
> 
> [关于更多METHOD参阅 SIP - Messaging](https://www.tutorialspoint.com/session_initiation_protocol/session_initiation_protocol_messaging.htm)

# 3 - SIP CALL 基本工作流

下面的图片描述了一段SIP call的基本工作流，目的是建立一段SIP session

<img title="" src="https://www.tutorialspoint.com/session_initiation_protocol/images/sip_call_flow.jpg" alt="">

> 下文REQUEST直接称作请求，RESPONSE直接称作回复

流程描述:

- 一个INVITE请求发出以初始化这段session (注意，session还未开始，后文详述)

- Proxy立即回复一个100 Tring给caller(Alice)，防止Alice重复发送INVITE请求

- 接着proxy寻找Bob的地址所在的服务器，之后在转发一次INVITE请求

- 接着callee(Bob)生成180 Ringing(临时回复)回复给caller(Alice)

- 在Bob接电话后其生成正式回复200 OK给Alice

- Alice发送ACK报文给Bob以通知Bob她已接收到200 OK

- 就在这时session正式建立，session从这里开始

- 在这段session中，任意一个人都能主动挂断电话直接发送bye到对端结束session

- Alice挂断电话，绕过proxy，直接发送bye给Bob

- 最后Bob发送200 OK确认by和终结session，此时session结束

上述流程完整的call也被称作dialog，一共有三个transaction(在图中1、2、3处)

transaction、dialog和session的区别下文揭示

> bye相关:
> 
> 下面链接描述了不能使用bye来结束未确立的invite；也说了bye是不通过proxy server，直接p2p联系。但是不要忘了这个过程只是不经过proxy server没说绕开register server
> 
> [SIP - Messaging](https://www.tutorialspoint.com/session_initiation_protocol/session_initiation_protocol_messaging.htm)
> 
> 
> 例如下面过程:
> 
> 在这个过程中可以明显观察到无论是invite和bye都要通过register服务器转发一次，对应的转发过程开了条call-id，可以看到虽然是p2p但是并非完全透明，register server明显参与了这次完整的call。在caller与register server之间call-id为`1808150176@192.168.7.101`；而callee与register server之间call-id为`5fcdc4bf-3662-123a-9a87-1158c9642285`。这两个call-id决定了一个session的建立和结束
> 
> [一次完整的通话过程SIP报文分析-腾讯云开发者社区-腾讯云](https://cloud.tencent.com/developer/article/1828603)

# 4 - 关于transaction、dialog和session

- Transaction(事务):一个Transaction包含一个Rrequest和可能的多个Response。在一个Transaction在接收到特定的Final Response(2xx, 3xx, 4xx, 5xx, or 6xx)和这些Response的ACK报文后结束。结束前也可以接受多个非Final Response(1xx)。注意，在Final Responese(200)后面的ACK报文并非Transaction的一部分

- Dialog(对话):一对SIP peers对话中间有多个Transaction。Dialog被设立的目的是建立、修改、拆卸(teardown，英文直译，应该是指发送bye结束session的动作)。在一对peer中可能无论何时都有可能产生多个Dialog，一般可以通过SIP头的from、to、call-id确认相应的Dialog。例如一个peer同时收到了两个bye请求，可以检查上述SIP头的字段以确认到底是哪个Dialog

- Session(会话):Session的概念相对简单，它仅仅用于描述流媒体传输之间的过程，没有流媒体传输就没有Session

# 5 - 关于tag、call-id和branch

- tag:标识身份

- call-id:决定一段session，`caller和rigister server的call-id`和`callee和register server的call-id`共同决定了一段session(似乎跳脱出session，call-id也能当branch用，例如一对message的Request和200 OK)

- branch:决定了transaction，相对应的transaction有一样的branch值
