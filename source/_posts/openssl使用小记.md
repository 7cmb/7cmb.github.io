---
title: openssl使用小记
tags:
  - openssl
  - network
categories:
  - - network
    - openssl
  - - network
    - tls
date: 2024-07-13 21:29:31
---


> 本文参考:
> https://www.cnblogs.com/efzju/p/4976144.html
>
> https://github.com/openssl/openssl/issues/10458
> 
> https://stackoverflow.com/questions/33989190/subject-alternative-name-is-not-copied-to-signed-certificate
>
> https://kubernetes.io/zh-cn/docs/tasks/administer-cluster/certificates/
> 
> https://docs.redhat.com/zh_hans/documentation/red_hat_enterprise_linux/7/html/security_guide/sec-Using_OpenSSL#sec-Creating_a_Self-signed_Certificate
>
> https://www.cnblogs.com/f-ck-need-u/p/6091027.html

本文记录笔者使用`openssl`命令行遇到的一点小问题。由于本人对安全只有最粗浅的理解，故本文只记录该命令不同`command`、`option`和`parameters`下的行为差异。先放出openssl的manpage:
```
OPENSSL(1ssl)                       OpenSSL                      OPENSSL(1ssl)

NAME
       openssl - OpenSSL command line program

SYNOPSIS
       openssl command [ options ... ] [ parameters ... ] 

       openssl no-XXX [ options ]

       openssl -help | -version

```

# 前言

近期看k8s相关文档及资料时遇到一系列tls/ssl的内容，遇到了正好也复习一下相关理论顺便也稍微使用`openssl`命令行做实验

众所周知，生成一张证书大致来说有三个步骤:
 - step-1 生成一条 **私钥**
 - step-2 使用 上述私钥 生成一张包含对应公钥及其他内容的 **csr**
 - step-3 使用上步 **csr** 及 一条用于签名的私钥 生成一张证书
 
 > 未验证内容:应该能在step-3加上签名私钥所属机构的证书。如果是这样，那么猜测签名私钥所属机构的证书摘要将对应生成下级证书`aki`；也对应这张证书本身的`ski`

综上所述，生成一张自签名证书，也有三个步骤:
 - step-1 生成一条 **私钥**
 - step-2 使用 上述私钥 生成一张包含对应公钥及其他内容的 **csr**
 - step-3 使用上步 **csr** 及 step-1的那条**私钥** 生成一张证书
 
大致流程如上，但是使用`openssl`命令行实践及读文档的时候遇到一点问题

## 疑问

读文档的过程中遇到一个概念`pseudo-commands(伪命令)`。所以第一个问题:什么是`pseudo-commands(伪命令)`

在使用openssl签署为csr签名颁发证书时，不同`command`下签署出来的证书字段不同。例如有一个相同的csr，使用`openssl req`与`openssl x509`签署出来字段不一样。所以第二个问题为:两条命令在签署自签名证书时的差异在哪？

> `command`概念参见上述manpage

本文章将记录本人的答案:

- 问题一: 什么是`pseudo-commands(伪命令)`？
- 问题二: `openssl req`与`openssl x509`两条命令在签署自签名证书时的差异在哪？

# 思路

## 问题一

对于`pseudo-commands(伪命令)`，网上貌似搜索不出什么东西。所以我的理解暂且是只用于openssl的一众命令中只用以显示信息而不做实际操作的命令。例如:`openssl list [option]` `openssl no-XXX` 这类命令

> 对于`openssl list [option]`，还有一种弃用的用法，网上搜索到的过时资料及手册极具误导性，包括但不限于rhel 7的文档、manpage 等等。这种弃用用法大多属为`pseudo-commands list-standard-commands`/`openssl list-standard-commands`。而新版本openssl的用法应该为`openssl list -standard-commands`

## 问题二

首先分步制作一张自签名证书

**step-1** 生成私钥:
```bash
openssl genrsa -out tls.key
```

-----

**step-2** 使用上述私钥生成csr:
```bash
openssl req -new -key ./tls.key -out tls.csr
```

接下来会弹出提示词，这时候除了`Common Name`填写域名外，其他一律填写`.`留空,extra属性直接回车

如果使用非交互式命令行可以添加`-sunj`实现，一下命令与上述操作等价:
```bash
openssl req -new -key ./tls.key -subj /CN=test.com -out tls.csr
```

-----

**step-3-1** 使用`openssl req`对csr自签名:
```bash
openssl req -x509 -in ./tls.csr -key ./tls.key -out ./tls.cert
```

执行完毕将跳出两条`warning`:
```bash
Warning: Not placing -key in cert or request since request is used
Warning: No -copy_extensions given; ignoring any extensions in the request
```

第一条可以忽略，只有没有给定`-in`下`-key`输入的私钥才会为新生成的csr或者证书提供对应公钥(用于自签名)。`openssl-req`手册已经说明了`-key`在`-in`给定csr的情况下是用以提供私钥为csr签名的:
```bash
       -key filename|uri
           This  option provides the private key for signing a new certificate
           or certificate request.  Unless -in  is  given,  the  corresponding
           public key is placed in the new certificate or certificate request,
           resulting in a self-signature.

```

第二条应该也能忽略，只要command是`req`，应该会读取`/etc/ssl/openssl.cnf`的配置；而csr也是通过`req`这个command生成的，对应配置应该是一样的

读取证书:
```bash
➜  1 openssl x509 -text -in ./tls.cert| nl
     1	Certificate:
     2	    Data:
     3	        Version: 3 (0x2)
     4	        Serial Number:
     5	            22:81:bc:7e:13:6a:43:65:63:b8:96:bf:2e:d5:91:db:ea:5b:64:ed
     6	        Signature Algorithm: sha256WithRSAEncryption
     7	        Issuer: CN=test.com
     8	        Validity
     9	            Not Before: Jul 13 08:48:16 2024 GMT
    10	            Not After : Aug 12 08:48:16 2024 GMT
    11	        Subject: CN=test.com
    12	        Subject Public Key Info:
    13	            Public Key Algorithm: rsaEncryption
    14	                Public-Key: (2048 bit)
    15	                Modulus:
    16	                    00:8e:ec:98:52:69:4f:80:f5:8f:4f:4b:02:6a:5e:
    17	                    d1:10:e7:74:19:70:95:c5:0f:9d:16:4c:31:a6:12:
    18	                    42:f1:27:b0:bb:f0:a6:6c:c2:0c:f6:28:f7:f2:19:
    19	                    62:78:31:78:84:3d:3d:61:e5:21:3a:f5:6d:ff:e6:
    20	                    b8:ac:20:b1:5c:e9:aa:8f:de:c6:fc:7c:50:39:c1:
    21	                    a8:e5:72:e1:f0:48:7b:53:f6:a3:5c:43:1b:e1:3c:
    22	                    96:1d:64:e8:ed:64:db:4e:86:96:c7:2f:c2:af:1f:
    23	                    66:18:72:ca:90:6f:46:6c:08:f0:9d:3a:89:52:56:
    24	                    70:0b:88:f3:49:d3:f3:4b:4f:f5:b3:87:ab:d4:38:
    25	                    39:b0:0f:89:70:88:4b:d7:e5:da:59:8c:2e:06:36:
    26	                    91:10:37:90:10:61:32:f4:d2:83:48:1f:1f:84:b0:
    27	                    f9:40:d6:27:65:e4:95:1f:54:10:6f:c7:8f:2b:e1:
    28	                    0a:70:f2:d4:b7:ad:18:cd:6e:2b:ad:31:e6:a6:41:
    29	                    1f:c6:69:0d:c4:fc:90:41:01:d6:93:04:18:89:2f:
    30	                    44:e4:a2:8a:86:11:0a:fd:3f:7d:13:55:9f:5b:8e:
    31	                    49:62:fa:18:94:fd:29:49:5e:b1:7a:9a:c8:9b:6f:
    32	                    13:1d:32:b9:5a:46:34:52:02:b9:31:f2:2e:7d:53:
    33	                    f3:19
    34	                Exponent: 65537 (0x10001)
    35	        X509v3 extensions:
    36	            X509v3 Subject Key Identifier: 
    37	                02:6C:AA:E7:28:7C:BB:25:71:AE:1A:E4:B1:E4:F7:67:ED:2B:A3:7D
    38	            X509v3 Authority Key Identifier: 
    39	                02:6C:AA:E7:28:7C:BB:25:71:AE:1A:E4:B1:E4:F7:67:ED:2B:A3:7D
    40	            X509v3 Basic Constraints: critical
    41	                CA:TRUE
    42	    Signature Algorithm: sha256WithRSAEncryption
    43	    Signature Value:
    44	        56:ca:74:c3:15:fa:2a:65:aa:fc:1d:04:aa:b9:1d:a6:bc:9d:
    45	        e8:60:a0:78:b2:fd:64:7f:5f:ee:14:fe:ea:d0:af:f3:98:99:
    46	        db:3e:0e:10:7d:a3:7c:ea:a1:6c:b5:05:0f:e1:01:7c:66:be:
    47	        1c:be:28:59:a8:1a:01:00:6a:64:22:5f:87:f5:03:1e:17:6f:
    48	        dc:1a:f8:4d:42:60:52:03:90:39:90:75:5d:48:82:5a:f2:58:
    49	        a2:3d:8b:62:89:34:49:c8:0d:35:b8:93:95:37:9b:10:97:ca:
    50	        56:7e:bc:99:09:7e:7e:94:2e:e2:f3:a0:1e:45:94:c2:6d:15:
    51	        21:c0:56:47:03:d9:48:85:72:9d:b8:e4:ae:39:97:d9:1a:92:
    52	        78:ee:7e:c3:95:6d:33:b4:d9:40:72:29:d3:af:eb:ce:8d:77:
    53	        a3:39:6f:00:36:9c:ed:3c:af:cf:de:24:88:f7:91:ff:7b:ef:
    54	        7c:e7:0e:c7:10:bb:44:e7:c6:0a:f8:c5:9f:ae:bd:2e:67:6a:
    55	        c9:9e:fb:be:cc:f4:c0:d1:87:57:12:ce:67:60:cf:ec:e8:ec:
    56	        45:5a:62:3b:81:00:07:fc:72:43:c8:b7:db:16:82:ef:cd:28:
    57	        46:e2:66:0b:18:56:3e:47:3f:9d:cd:b1:50:20:5f:1e:b4:99:
    58	        f0:19:eb:59
    59	-----BEGIN CERTIFICATE-----
    60	MIIDBzCCAe+gAwIBAgIUIoG8fhNqQ2VjuJa/LtWR2+pbZO0wDQYJKoZIhvcNAQEL
    61	BQAwEzERMA8GA1UEAwwIdGVzdC5jb20wHhcNMjQwNzEzMDg0ODE2WhcNMjQwODEy
    62	MDg0ODE2WjATMREwDwYDVQQDDAh0ZXN0LmNvbTCCASIwDQYJKoZIhvcNAQEBBQAD
    63	ggEPADCCAQoCggEBAI7smFJpT4D1j09LAmpe0RDndBlwlcUPnRZMMaYSQvEnsLvw
    64	pmzCDPYo9/IZYngxeIQ9PWHlITr1bf/muKwgsVzpqo/exvx8UDnBqOVy4fBIe1P2
    65	o1xDG+E8lh1k6O1k206Glscvwq8fZhhyypBvRmwI8J06iVJWcAuI80nT80tP9bOH
    66	q9Q4ObAPiXCIS9fl2lmMLgY2kRA3kBBhMvTSg0gfH4Sw+UDWJ2XklR9UEG/Hjyvh
    67	CnDy1LetGM1uK60x5qZBH8ZpDcT8kEEB1pMEGIkvROSiioYRCv0/fRNVn1uOSWL6
    68	GJT9KUlesXqayJtvEx0yuVpGNFICuTHyLn1T8xkCAwEAAaNTMFEwHQYDVR0OBBYE
    69	FAJsqucofLslca4a5LHk92ftK6N9MB8GA1UdIwQYMBaAFAJsqucofLslca4a5LHk
    70	92ftK6N9MA8GA1UdEwEB/wQFMAMBAf8wDQYJKoZIhvcNAQELBQADggEBAFbKdMMV
    71	+iplqvwdBKq5Haa8nehgoHiy/WR/X+4U/urQr/OYmds+DhB9o3zqoWy1BQ/hAXxm
    72	vhy+KFmoGgEAamQiX4f1Ax4Xb9wa+E1CYFIDkDmQdV1IglryWKI9i2KJNEnIDTW4
    73	k5U3mxCXylZ+vJkJfn6ULuLzoB5FlMJtFSHAVkcD2UiFcp245K45l9kaknjufsOV
    74	bTO02UByKdOv686Nd6M5bwA2nO08r8/eJIj3kf9773znDscQu0Tnxgr4xZ+uvS5n
    75	asme+77M9MDRh1cSzmdgz+zo7EVaYjuBAAf8ckPIt9sWgu/NKEbiZgsYVj5HP53N
    76	sVAgXx60mfAZ61k=
    77	-----END CERTIFICATE-----
```

注意这条命令打印到基本输出的第`35-41`行，稍会会用`openssl x509`重新为同一份csr签名，这其中的内容将有所不同。这里重新截取35-41行内容:
```bash 
➜  1 openssl x509 -text -in ./tls.cert | awk 'NR>=35&&NR<=41'
        X509v3 extensions:
            X509v3 Subject Key Identifier:
                02:6C:AA:E7:28:7C:BB:25:71:AE:1A:E4:B1:E4:F7:67:ED:2B:A3:7D
            X509v3 Authority Key Identifier:
                02:6C:AA:E7:28:7C:BB:25:71:AE:1A:E4:B1:E4:F7:67:ED:2B:A3:7D
            X509v3 Basic Constraints: critical
                CA:TRUE

```

**step-3-2** 使用`openssl x509`对csr自签名:
```bash
openssl x509 -req -in ./tls.csr -key ./tls.key -out tls-x509.cert
```

提取证书信息，同样筛选`35-41`行:
```bash
➜  1 openssl x509 -text -in ./tls-x509.cert | awk 'NR>=35&&NR<41'
        X509v3 extensions:
            X509v3 Subject Key Identifier:
                02:6C:AA:E7:28:7C:BB:25:71:AE:1A:E4:B1:E4:F7:67:ED:2B:A3:7D
    Signature Algorithm: sha256WithRSAEncryption
    Signature Value:
        0e:2c:28:c7:4c:4e:4a:58:a4:95:93:39:4b:cd:8a:7a:3b:82:
```

同样的csr，使用`openssl req`和`openssl x509`签署出来的证书字段明显不一样。说明它们的行为可能不同

根据[骏马金龙的这篇文章](https://www.cnblogs.com/f-ck-need-u/p/6091027.html)和`/etc/ssl/openssl.cnf`的注释段:
```
# To use this configuration file with the "-extfile" option of the
# "openssl x509" utility, name here the section containing the
# X.509v3 extensions to use:
# extensions		=
# (Alternatively, use a configuration file that has only
# X.509v3 extensions in its main [= default] section.)
```

可以推测的结论是`openssl x509`工具不但需要手动使用`-extfile`参数指定配置文件，还需要在配置文件中指定需要的section(否则为default)。所以需要达成`openssl req`签署出的效果最少需要一份配置得当的配置文件，且在其中配置`default`这个section中的extensions

### 尝试使用上述命令生成拥有相同字段的证书

从上述输出中的不同之初可以推测问题大致出在extensions上，从`openssl-x509`的手册中可以查到:
```
       -copy_extensions arg
           Determines how to handle X.509 extensions when  converting  from  a
           certificate  to a request using the -x509toreq option or converting
           from a request to a certificate using the -req option.  If  arg  is
           none or this option is not present then extensions are ignored.  If
           arg  is copy or copyall then all extensions are copied, except that
           subject identifier and authority key identifier extensions are  not
           taken over when producing a certificate request.

           The -ext option can be used to further restrict which extensions to
           copy.
           
----------分隔符----------

       -ext extensions
           Prints out the certificate extensions in text form.   Can  also  be
           used   to  restrict  which  extensions  to  copy.   Extensions  are
           specified with a comma  separated  string,  e.g.,  "subjectAltName,
           subjectKeyIdentifier".   See  the  x509v3_config(5) manual page for
           the extension names.

----------分隔符----------

       -extfile filename
           Configuration   file   containing  certificate  and  request  X.509
           extensions to add.
         
----------分隔符----------
           
       -extensions section
           The section in the extfile to add X.509 extensions from.   If  this
           option  is  not  specified  then  the  extensions  should either be
           contained in the unnamed (default) section or the  default  section
           should  contain  a  variable called "extensions" which contains the
           section to use.

           See the x509v3_config(5) manual page for details of  the  extension
           section format.

           Unless  specified otherwise, key identifier extensions are included
           as described in x509v3_config(5).


```

从第一个参数`-copy_extensions`的说明可以得知如果使用`openssl x509 -req`时如果没有指明该参数或者该参数为`none`那么就会忽略所有csr中的extensions；另外，对于SKI与AKI插件(AKI就是缺少的字段之一)，即使该参数为copy或者copyall，那么也不会复制到证书中。且即便加了此参数，也能在后续的`-ext`参数中严格限制复制的extensions

> 这里似乎有详细说明原因，但是看不懂https://github.com/openssl/openssl/issues/10458

第二个参数`-ext`严格限制复制的extensions，多个extensions用逗号分隔

第三个参数`-extfile`指定配置文件如`/etc/ssl/openssl.cnf`

第四个参数`-extensions`在指定配置文件(如openssl.cnf)中的某个section指定命令行为，具体参见[骏马金龙的这篇文章](https://www.cnblogs.com/f-ck-need-u/p/6091027.html)

综合上述所有的资料，如果需要用`openssl x509`签署csr并与`openssl req`达成相同的行为不但需要使用`-extfile`显式指定一份配置文件，还需要在section中定义需要的extensions。如果含有extensions的section不为default，还需要用`-extensions`指定需要的section

> 一些问题:在默认配置文件中按照注释提示配置了default这条section并且其中定义了相关sections,不知道为什么会报错；将default改为其他字段并使用`-extension`指定  或者  将含有default这条section及对应extensions放在一份新的文件中并使用`-extfile`指定却能正确执行
>
> 我猜这个问题和默认的其他参数有关

例如这里指定默认配置文件`/etc/ssl/openssl.cnf`，并且指定section为`v3_ca`:
```bash
openssl x509 -req -in ./tls.csr -key ./tls.key -copy_extensions copyall -extensions v3_ca -extfile /etc/ssl/openssl.cnf -out ./tls-x509-cnf.cert
```

同样检查`35-41`行并使用diff对比`openssl req`签署的证书:
```bash
➜  1 openssl x509 -in ./tls-x509-cnf.cert -text | awk 'NR>=35&&NR<=41' > ./2
➜  1 openssl x509 -in ./tls.cert -text | awk 'NR>=35&&NR<=41' > ./1
➜  1 diff ./1 ./2
```

diff发现并无差异

自此，对于`openssl req`和`openssl x509`的签署自签名证书的默认行为也算有初步的了解了...