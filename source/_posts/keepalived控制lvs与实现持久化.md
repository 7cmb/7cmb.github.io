---
title: keepalived控制lvs与实现持久化
date: 2024-02-21 22:41:35
tags:
categories:
---
用`keepalived`来控制lvs的行为而不通过`ipvsadm`可以大大简化配置流程。用ipvsadm配置还得通过一些[手段](https://www.cnblogs.com/alexlv/p/14789862.html#:~:text=%2Fetc%2Fsysconfig%2Fipvsadm,4%E3%80%81%E9%80%9A%E8%BF%87%E9%87%8D%E5%AE%9A%E5%90%91%E5%B0%86%E5%BD%93%E5%89%8D%E8%A7%84%E5%88%99%E9%87%8D%E5%AE%9A%E5%90%91%E5%88%B0%E7%B3%BB%E7%BB%9F%E9%BB%98%E8%AE%A4%E7%9A%84%E8%A7%84%E5%88%99%E5%AD%98%E6%94%BE%E4%BD%8D%E7%BD%AE%EF%BC%8C%E5%B0%86%E8%A7%84%E5%88%99%E5%AD%98%E6%94%BE%E5%9C%A8%E8%BF%99%E4%B8%AA%E6%96%87%E4%BB%B6%E9%87%8C%EF%BC%8C%E9%87%8D%E5%90%AF%E6%9C%8D%E5%8A%A1%E4%BC%9A%E8%87%AA%E5%8A%A8%E6%81%A2%E5%A4%8D%E9%87%8C%E9%9D%A2%E7%9A%84%E8%A7%84%E5%88%99)把配置保留下来。keepalived一站式解决lvs、vrrp，实在是太爽啦(就是有些地方UserGuide和Manual会打架)


对于配置keepalived的vrrp实例还是很简单的，就是注意配置文件`/etc/keepalived/keepalive.conf`权限不能为执行，否则会报错<del>(远程配置文件的时候还没有搞定su下x转发的问题)</del>


## 基本配置

***部分条目注释将放在备份调度器配置文件***
***部分条目注释将放在备份调度器配置文件***
***部分条目注释将放在备份调度器配置文件***

主调度器配置
路径`/etc/keepalived/keepalived.conf`:
```bash
global_defs {
  router_id lvs1
# lvs_sync_daemon enp0s3
# lvs_timeouts tcp 30 tcpfin 90 udp 90
}
vrrp_instance VI_1 {
   state BACKUP
   interface enp0s3
   virtual_router_id 51
   priority 101
   advert_int 10
   authentication {
       auth_type PASS
       auth_pass 1111
   }
   virtual_ipaddress {
      192.168.56.49/24
   }
}
virtual_server 192.168.56.49 80 {
   delay_loop 10
   lb_algo rr
   lb_kind DR
   persistence_timeout 
   protocol TCP
   real_server 192.168.56.56 80 {
       weight 100
       TCP_CHECK {
	        connect_timeout 3
                retry 3
                delay_before_retry 3
                connect_port 80
       }
   }

   real_server 192.168.56.57 80 {
       weight 100
       TCP_CHECK {
	        connect_timeout 3
                retry 3
                delay_before_retry 3
                connect_port 80
       }
   }
}

```

备份调度器配置
路径同上:
```bash
global_defs {
  router_id lvs2

  # 同步lvs的配置，此步不开下行配置可能不生效 
# lvs_sync_daemon enp0s3

  # 设置lvs超时时间，为空则保持之前的设置不变，此行和持久化强相关
# lvs_timeouts tcp 30 tcpfin 90 udp 90
}
vrrp_instance VI_1 {
   state BACKUP
   interface enp0s3
   
   # 同一组vrrp服务虚拟路由id相同，同一接口可以跑多组服务
   virtual_router_id 51
   
   # 优先级以大为优 
   priority 100
   
   advert_int 10
   
   # 同一组认证需相同
   authentication {
       auth_type PASS
       auth_pass 1111
   }
   
   # vip和主调度器配置不相同的话，包括掩码前缀有无，将导致vrrp分组失败
   virtual_ipaddress {
      192.168.56.49/24
   }
}
virtual_server 192.168.56.49 80 {
   delay_loop 10
   lb_algo rr
   lb_kind DR
   
   # 开启持久化，持久化时间默认360，单位为秒
   persistence_timeout 
   
   protocol TCP
   real_server 192.168.56.56 80 {
       weight 100
       TCP_CHECK {
	        connect_timeout 3
                retry 3
                delay_before_retry 3
                connect_port 80
       }
   }

   real_server 192.168.56.57 80 {
       weight 100
       TCP_CHECK {
	        connect_timeout 3
                retry 3
                delay_before_retry 3
                connect_port 80
       }
   }
}

```

重启keepalived验证配置是否成功:
```bash
[baka@lvs1 ~]$ sudo ipvsadm -L
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  lvs1:http rr persistent 360
  -> 192.168.56.56:http           Route   100    12         0
  -> 192.168.56.57:http           Route   100    19         0
```

可以看到成功配置，符合期望

本配置如下:
Master LVS:192.168.56.51
Backup LVS:192.168.56.52
Real Server1:192.168.56.56
Real Server2:192.168.56.57

## 关于持久化
以上配置成功后，发现轮询不生效了，即使持久化时间即上述配置文件的`persistence_timeout`设置得很短，也无法在此时间后切换rs。百思不得其解，遂发现该行为还和`lvs_timeouts`控制的时间有关。如果是DR模式下使用curl直接访问上述配置的vip，在主调度器使用`ipvsadm -lnc查询lvs连接表时，一共有两种条目，则分别为`ASSURED`和`ESTABLISHED`

先下结论:
**对于同一个客户端(相同ip)，当这个表中还有它的条目时，那么使用该ip的客户端下次将还是连接相同的rs**

persistence_timeout设定的时间对应lvs表的条目是`ASSURED`  
tcp连接建立成功后对应的条目是`ESTABLISHED`:
<img title="" src="https://dlink.host/1drv/aHR0cHM6Ly8xZHJ2Lm1zL2kvcyFBckVNT01Ec2ZXcEdnU2Z5NGpJc3prdmZEU1VKP2U9STI0OWhx.png" alt="借图一用" >

lvs_timeouts默认值是tcp 900 tcpfin 120 udp 300

持久化选项开启后，lvs将根据这个表来定义持久化行为。对于同一客户端(用ip区分不同客户端)，当这个表对应的条目全部消失了，才会根据负载均衡指定的方式为这个客户端分配下一台rs

一般来说，`persistence_time`设定的值要比lvs_timeouts的值要大（如果这个值小于lvs_timeouts那它根本没有意义），否则最后一个连接条目消失前，它会在超时的时候重新刷新倒计时为一分钟
也就是说如果`persistence_time`小于lvs_timeouts时，它最长持续时间是表中最后一个连接条目消失后加一分钟
> 以ip区分不同客户端   
> 
> 每次有新连接刷新或建立时，`persistence_time`将同步刷新
> 
> lvs_timemouts控制每个连接条目保持的时间，条目保持的时间和persistence_time共同控制客户端保持访问同一台rs

### 例子
这里有一个超短的`lvs_timeouts`和一个比它更短`persistence_time`:
<img title="" src="https://dlink.host/1drv/aHR0cHM6Ly8xZHJ2Lm1zL2kvcyFBckVNT01Ec2ZXcEdnU2hwS0ZHS3NhRG1pODZWP2U9WlNkaDFs.png" alt="" >
建立连接的条目消失后，理应更早一秒消失的`persistence_time`居然带着更长的时间回来了:
<img title="" src="https://dlink.host/1drv/aHR0cHM6Ly8xZHJ2Lm1zL2kvcyFBckVNT01Ec2ZXcEdnU2Z5NGpJc3prdmZEU1VKP2U9STI0OWhx.png" alt="我在这呢" >

## 结论
持久化不但需要看持久化时间(persistence_time)，更要看lvs连接条目的保持时间(lvs_timeouts)。真是人不可貌相


> 本帖参考:
> [lvs 两个timeout解析](https://www.jianshu.com/p/6b3202599682)
>
> [LVS负载均衡之持久性连接介绍](https://icloudnative.io/posts/lvs-persistent-connection/#%E5%89%8D%E8%A8%80)
>
> [Persistence Handling in LVS](http://www.linuxvirtualserver.org/docs/persistence.html) 