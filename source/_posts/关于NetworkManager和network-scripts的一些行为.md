---
title: 关于NetworkManager和network-scripts的一些行为
tags:
  - linux
  - network
categories:
  - - linux
    - network
  - - linux
    - network-scripts
  - - linux
    - NetworkManager
date: 2024-12-09 02:04:18
---

已测试发行版 `CentOS 7.9` 、 `RHEL 8.8` 、 `RockyLinux 9.4`

本文名词简称声明:
```bash
NetworkManager="nm"
connection="con"
```

本文推荐在了解三个概念下阅读:
 - 各种 linux 网络管理器
 - NetworkManager.conf 配置文件中 `[main]` section 下的 `plugins` 键 (`man NetworkManager.conf` 搜索 "PLUGINS")
 - NetworkManager `ifcfg-rh` 和 `keyfile` 配置接口下的配置选项 (`man nm-settings-ifcfg-rh` `man nm-settings-keyfile`)

# NetworkManager 下遇到的奇怪问题

一般在 RHEL 系发行版，我会偏好 `network-scripts` 和 `NetworkManager` 这两款工具来管理网络。我的个人机器大多也使用后者管理网络，一般来说，我只会维护一份 connection 文件，无论是使用 `keyfile` 、 `ifdownup` 还是 `ifcfg-rh` 接口的情况下(我常用 ArchLinux 、 Debian 和兼容 RHEL 二进制的社区发行版，这三个接口对应这三个发行版的默认配置)。

最近碰到个场景，涉及到多个 con 而不是维护单个 con 。具体是这样的: 需要 ***在 RHEL 系发行版下使用脚本先 clone 原 connection 作为备份文件后，再修改原 connection 修改配置，最后重新激活原 connection 。***

这个行为在一些动作后引发一个奇怪的现象：在这个行为执行了若干次以后，产生了多个备份文件时， ***当手动使用 nmcli 激活另一份绑定同一张网口的 con 时*** ，随着机器 reboot 后，nm 激活的 con 重视不会是最后一次激活的 con ，而是激活到了的旧的 con 。且重启后再使用 nmcli 手动执行一次这个动作再重启，也依旧是稳定激活旧 con ，而不是我们的目标 con 。很明显，这个不是预期的结果，简单来说：

 - 当在这个场景下，无论是 `keyfile` 还是 `ifcfg-rh` 作为 nm 配置文件的接口，当存在若干份 con 且它们的配置的网络接口相同时，激活目标 con 后无法保证下一次重启后激活的 con 是目标 con 而不是旧 con

## 尝试 reproduce 

举个例子，在使用 `ifcfg-rh` 作为配置文件的接口，当存在两个 con 且他们绑定的网络接口都为 `ens4`:
```bash
# 旧的 connection `ifcfg-b` 指定 IP 为 22.22.22.22/32
# 新的 connection `ifcfg-a` 指定 IP 为 33.33.33.33/32
# 下面首先将 22.22.22.22 这个配置文件激活后，再将 
# 33.33.33.33 激活，观察重启后所得到的 IP 地址

# 首先检查配置文件是否正确
[root@rhel-88-noSVT network-scripts]# grep 'IPADDR\|PREFIX\|NAME' ./ifcfg-*
./ifcfg-a:NAME=a
./ifcfg-a:IPADDR=33.33.33.33
./ifcfg-a:PREFIX=32
=====手动分隔符=====
./ifcfg-b:IPADDR=22.22.22.22
./ifcfg-b:PREFIX=32
./ifcfg-b:NAME=b


# 激活配置 `ifcfg-b` 22.22.22.22 并检查配置是否生效
[root@rhel-88-noSVT network-scripts]# nmcli con up b
Connection successfully activated (D-Bus active path: /org/freedesktop/NetworkManager/ActiveConnection/4)
[root@rhel-88-noSVT network-scripts]# nmcli con show
NAME    UUID                                  TYPE      DEVICE
b       bb3a696e-3c63-4d33-aa18-a421948eb493  ethernet  ens4   <-- 这里是绿色的
virbr0  7b95e7cd-24b4-48b3-8053-e394926d5d39  bridge    virbr0
a       c86226f5-8cda-4792-81c7-ea9cb1813080  ethernet  --
c       a2983dd9-d631-4225-9a32-0365b0f2c756  ethernet  --
[root@rhel-88-noSVT network-scripts]# ip a | grep ens4
2: ens4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    inet 22.22.22.22/32 scope global noprefixroute ens4        <-- 拿到目标 IP

# 激活配置 `ifcfg-a` 33.33.33.33 并检查配置是否生效
[root@rhel-88-noSVT network-scripts]# nmcli con up a
Connection successfully activated (D-Bus active path: /org/freedesktop/NetworkManager/ActiveConnection/5)
[root@rhel-88-noSVT network-scripts]# nmcli con show
NAME    UUID                                  TYPE      DEVICE
a       c86226f5-8cda-4792-81c7-ea9cb1813080  ethernet  ens4   <-- 这里是绿色的
virbr0  7b95e7cd-24b4-48b3-8053-e394926d5d39  bridge    virbr0
b       bb3a696e-3c63-4d33-aa18-a421948eb493  ethernet  --
c       a2983dd9-d631-4225-9a32-0365b0f2c756  ethernet  --
[root@rhel-88-noSVT network-scripts]# ip a | grep ens4
2: ens4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    inet 33.33.33.33/32 scope global noprefixroute ens4        <-- 拿到目标 IP
```
这里稍微做一下阶段总结，我们首先激活了配置文件 `ifcfg-b` 后又激活了配置文件 `ifcfg-a` 也就是说，如果用 IP 代表我们的配置文件的话

我们的 `ens4` 先拿到了 22.22.22.22/32 作为地址，后又激活新的 con 拿到了新的地址 33.33.33.33/32 并撤掉了原地址，接下来 `reboot` 测试一下我们到底哪个地址会保留下来:
```bash
# 可以看到激活的配置文件是 `ifcfg-b` ，也就意味着地址将是 22.22.22.22/32
[root@rhel-88-noSVT network-scripts]# nmcli con show
NAME    UUID                                  TYPE      DEVICE
b       bb3a696e-3c63-4d33-aa18-a421948eb493  ethernet  ens4   <-- 这里是绿色的
virbr0  7052d934-5cb2-44b8-875a-7d6159476252  bridge    virbr0
a       c86226f5-8cda-4792-81c7-ea9cb1813080  ethernet  --
c       a2983dd9-d631-4225-9a32-0365b0f2c756  ethernet  --
# 开箱验证可以看到 IP **不是** 我们最新激活的 33.33.33.33/32
[root@rhel-88-noSVT network-scripts]# ip a | grep ens4
2: ens4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    inet 22.22.22.22/32 scope global noprefixroute ens4       <-- 这里是绿色的
```
居然是旧配置文件被激活了，不是最近激活的配置文件，WHY ?

从 manpage 可以知道当配置文件优先级相同时，将使用 `connection.timestamp` 这个 property 作为最近激活配置文件的指标，根据这个线索，我们可以回到上一步，逐步检查 `connection.timestamp` 这个属性(下文简称 timestamp ):
```bash
[root@rhel-88-noSVT network-scripts]# nmcli con up b
Connection successfully activated (D-Bus active path: /org/freedesktop/NetworkManager/ActiveConnection/3)
[root@rhel-88-noSVT network-scripts]# nmcli con show b | grep timestamp
connection.timestamp:                   1733678430
[root@rhel-88-noSVT network-scripts]# nmcli con up a
Connection successfully activated (D-Bus active path: /org/freedesktop/NetworkManager/ActiveConnection/4)
[root@rhel-88-noSVT network-scripts]# nmcli con show a | grep timestamp
connection.timestamp:                   1733678474
[root@rhel-88-noSVT network-scripts]# nmcli con show b | grep timestamp
connection.timestamp:                   1733678474
```

从这里开始就很明显了，在 up 新 con 的同时，旧 con 的 `timestamp` 也会被刷新一次

我们再 narrow down 一下，既然 up 新 con 这个属性会被刷新，那么 down 一下当前配置这个属性会不会刷新呢:
```bash
# 很明显，属性刷新了，具体看倒数两个时间戳
[root@rhel-88-noSVT network-scripts]# bash -c '
nmcli con down b;
nmcli con show b | grep timestamp;
date +%s'
Connection 'b' successfully deactivated (D-Bus active path: /org/freedesktop/NetworkManager/ActiveConnection/5)
connection.timestamp:                   1733679276
1733679276
```

这下傻眼了，明明红帽的文档的描述是这样的:
| Key Name | Value Type | Default Value | Value Description |
| :--- | :--- | :--- | :--- |
| timestamp | uint64 | 0 | The time, in seconds since the Unix Epoch, that the connection was last _successfully_ fully activated. NetworkManager updates the connection timestamp periodically when the connection is active to ensure that an active connection has the latest timestamp. The property is only meant for reading (changes to this property will not be preserved).|

俺寻思根本不存在保持 up 的 con 会定时刷新 `timestamp` 就算了，这个 down 掉怎么也会刷新一次......上当受骗了

考虑到我用的 `ifcfg-rh` 现在已经被弃用了，换了 `RockyLinux 9.4` 做测试，问题依旧，过程略

又经过若干测试，两个 con 在 `timestamp` 相同的情况下重启后激活并非随机的，而是固定打中某一个配置文件，这可能意味着可能有什么神秘力量控制着这个优先级，这下基本可以确定三个方向：
- 配置文件的名称
- con 的 id
- con 的 uuid

第一点是因为 `network-scripts` 后端的启发，观察过系统日志可以发现当 `ifcfg` 文件的 uuid 相同且配置同一个设备时，总是会优先启用名称排序优先级大的，然后检测到其他带相同 uuid 的配置文件时，将忽略掉他们，然后接下来按照这个名称排序启用下一份带不同 uuid 的配置文件

第二点是合理的推测，但是得到的结论是不影响

第三点是高人指点下，试了一下只能说说的道理。回到 `ifcfg-b` 和 `ifcfg-c` 这个例子，可以看看它们的 uuid 首字母，确实是符合预期的

> 当然这个排序决定优先级并不严谨，谁知道 nm 的 uuid 排序 sort by what ？但是通过实验能确定 a-z 之间，a 开头优先级会比较高

## 网络后端为 nm ，且 plugins 为 `ifcfg-rh` 和 `keyfile` 下解决手段

现在看这个问题就很直观了，分为三个思路:
- 添加 priority 相关属性，这个相对正路
- 将配置文件 up 两次，保持目标的 timestamp 为最新而不是与被顶掉的旧配置相同，奇怪的做法，但啥也不用改且有用
- 修改 uuid 首字母/数字以手动排序，这个如果需要严谨推出来排序依据好麻烦...总感觉很邪门

# network-scripts 的问题

累了不想写了，大哥手动搜一下下文的位置吧:
`第一点是因为 network-scripts 后端的启发，观察过系统日志可以发现当 ifcfg 文件的 uuid 相同且配置同一个设备时，总是会优先启用名称排序优先级大的，然后检测到其他带相同 uuid 的配置文件时，将忽略掉他们，然后接下来按照这个名称排序启用下一份带不同 uuid 的配置文件`
