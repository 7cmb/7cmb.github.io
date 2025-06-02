---
title: 关于TLS中的密钥交换与密钥同意
tags:
  - network
categories:
  - - network
    - tls
date: 2025-06-02 22:53:05
---


> <del>作为数学苦手，每次接触和数学相关的东西真是太痛苦了</del>
>
> 本文参考：
>
> https://www.cnblogs.com/xiaolincoding/p/14318338.html
>
> https://www.cnblogs.com/xiaolincoding/p/14274353.html
>
> https://en.wikipedia.org/wiki/Curve25519
> 
> https://evilpan.com/2022/05/15/tls-basics/#server-certificate-1

# 前言

最近需要用到 openssl 做一些事情，正好再次学习一下非对称加密。TLS 作为日常中最常接触的协议，便以此入手开始学习。在大致了解了 RSA 及 DH（本文代指 ECDHE，并且以 x25519 为例子） 的一些特性后（主要是前向加密），发现使用 DH 作为加密手段的 TLS 1.3 并没有密钥交换，且证书中的 Key Usage 也没有公钥加密对称加密相关字段（Key Encipherment）

众所周知，TLS 握手过程中会使用非对称加密交换对称加密的相关信息，握手完成将切换为对称加密完成上下文。而使用 DH 算法实现的 TLS  1.3 根本没有密钥交换，证书里也没有更没有 Key Encipherment 这就有点反直觉了。所以本文有三个问题：

 - 在 TLS 1.3 中，为什么不需要密钥交换且证书中不含 Key Encipherment 

 - 在 TLS 1.3 中，对称加密的密钥是从哪来？

 - 在 TLS 1.3 中，这个对称加密的密钥双方又是如何得到的？

# TLS 1.2 和 TLS 1.3 信息对比

以访问 google.com 为例，现代浏览器一般都会使用 TLS 1.3 ,下面以 Firefox 为例查看网站的 TLS 及证书信息：
<img src="https://dlink.host/1drv/aHR0cHM6Ly8xZHJ2Lm1zL2kvYy80NjZhN2RlY2MwMzgwY2IxL0VkQS01SWtqYjNOTnBDd254MVdHRmtJQjZ0Z3RVdVFyVklOdFh2cE9Jc3BHV0E_ZT1BdWRROWY.jpg" alt="TLS_lab_04">

<img src="https://dlink.host/1drv/aHR0cHM6Ly8xZHJ2Lm1zL2kvYy80NjZhN2RlY2MwMzgwY2IxL0VZakNVLXVrMmxSSW1FWUlOSEhneHNNQlphOW1reDBDZk1TT2JrQm55cXpxSmc_ZT1lbldWUjg.jpg" alt="TLS_lab_03">

可以看到证书的标黄 Key Usage 部分，并没有关于 `Key Encipherment` 这个字段。也就是说这张证书并不参与密钥交换/同意的过程（并且该字段已经被 TLS 1.3 弃用）

再以访问 google.com 为例，使用一个比较典型的 Chrome 49 以使用 TLS 1.2 ：

<img src="https://dlink.host/1drv/aHR0cHM6Ly8xZHJ2Lm1zL2kvYy80NjZhN2RlY2MwMzgwY2IxL0VWS3RESmpyeXlGQmh0a3pHaGhqTVZBQktqNXQzb0dEX201dE52aElSNzRxMHc_ZT1CRkpaMGM.jpg" alt="TLS_lab_02">

<img src="https://dlink.host/1drv/aHR0cHM6Ly8xZHJ2Lm1zL2kvYy80NjZhN2RlY2MwMzgwY2IxL0VkYWlUaUFYOVVGR2h4X0YzbXAwNzdjQjhkVC1XZFlhUC1DZk9ja29FZE9PUnc_ZT1mSXRkN0U.jpg" alt="TLS_lab_01">

在 1.2 版本，可以看到 `Key Encipherment` 是存在的，也就是说证书参与了密钥交换/同意的过程

对于 TLS 1.2 来说，使用 RSA 生成预主密钥这个过程（客户端生成一段作为对称加密随机数的参数，并用服务器公钥加密）发生在验证服务器证书以后，也就是说当服务器身份验证通过后，客户端将会利用服务器公钥将生成对称加密密钥的参数加密再发给服务端。因为客户端没有一对公私钥没法做数字签名，客户端再发一个「Encrypted Handshake Message（Finishd）」消息，把之前所有发送的数据做个摘要，再用会话密钥（master secret）加密一下，让服务器做个验证，验证加密通信是否可用和之前握手信息是否有被中途篡改过。很明显证书对对称加密密钥起到了加密作用，所以我认为 `Key Encipherment` 这个字段在这里是合理的

而同样是 TLS 1.2 ,使用 DH 生成对称密钥这个过程同样发生在验证服务器证书以后。但是不同的是客户端并不会生成预主密钥，而是由服务器主动发起交换（在 Client Hello 阶段客户端已经发送过支持的密码学套件且 Server Hello 中服务器已经选择相应的密码学套件），服务器发送公钥并挑选一组算法如 x25519 以确定模数 P 及阶数 G ，并且双方已经在 Hello 阶段确认了随机数，之后只需要客户端将其的公钥发回来服务器，双方再用自己的私钥（临时私钥）和已知的信息算出对称加密密钥。这个过程客户端发出的公钥由证书公钥对应的私钥（TLS 证书私钥）签名，很明显证书参与了这个过程，所以我认为 `Key Encipherment` 这个字段在这里也是合理的

在 TLS 1.2 中，密钥交换字段很好理解，毕竟真的有 RSA 预主密钥交换的过程；`Key Encipherment`也很好理解，毕竟在上述两个过程中，服务器证书相关的私钥都直接参与了加密/签名

那么为什么 TLS 1.3 不需要密钥交换与 Key Encipherment 呢？

## 在 TLS 1.3 中，为什么不需要密钥交换且证书中不含 Key Encipherment

针对第一个小问，为什么不需要密钥交换。我的理解是：

 - 在 TLS 1.3 中，因为裁剪了生成对称密钥所需要的密码学套件（只支持 DH 算法），所以在 Client Hello 阶段客户端完全可以将所有需要的参数提供给服务器选择，例如 key_group（确认 ECC 的模数 P 以及阶数 G，不同的 Group 有不同的固定模数与阶数）、 临时公钥 及其他生成对称密钥所需要的参数。服务端收到这些参数之后选择完毕在 Server Hello 发送给客户端，之后双方根据这些参数直接计算出对称密钥就好了。这个过程中所有参数都是公开的，甚至不用验证双方身份（下文会做补充），需要的对称密钥算出来就行了。由于强制使用 DH 做为生成对称密钥的工具，且直接合并在了 Hello 中，不需要额外的 1-RTT 做密钥交换，密钥同意的信息已经在 Hello 中了，自然也不需要密钥交换这个过程

针对第二个小问，为什么不需要 Key Encipherment ，我的理解是：

 - 在 TLS 1.3 中，由于强制使用 DH 算法，即使公开所有运算所需要的信息，只要临时私钥不泄露，那么就能保证当次会话安全（起码目前大多数资料是这样说的），自然也不需要加密任何信息。这个过程中根本用不上证书，自然不需要 Key Encipherment

> 未亲自验证内容：
>
> 在 TLS 1.3 中，会话进入对称加密阶段之后，才会在加密会话中做身份验证。和 TLS 1.2 在明文中做身份验证相反

## 在 TLS 1.3 中，对称加密的密钥是从哪来

实际上，第一个回答已经蕴含了这段内容。因为使用 DH 算法，所以对称加密密钥不需要双方共享一个预主密钥作为参数算出最终密钥，只需要依靠 DH 算法中双方公开的参数和自身的临时私钥，就可以算出接下来会话的对称密钥。这是 DH 算法的特性决定的

## 在 TLS 1.3 中，这个对称加密的密钥双方又是如何得到的？

这个答案应该在之前的章节里都有体现，是能根据 DH 算法直接算出来的。利用临时私钥以及 Hello 中交换的信息就能直接算出会话的对称密钥；当然理论上 Hello 中交换的信息理论上是能反推出双方的私钥的，但是起码以目前科技的算力是不足以做到这件事情的
