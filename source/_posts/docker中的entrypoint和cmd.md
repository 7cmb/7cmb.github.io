---
title: docker中的entrypoint和cmd
tags:
  - docker
categories:
  - - container
    - docker
date: 2024-04-28 16:32:05
---

关于docker中的entrypoint和cmd的疑惑很久了，一直搞不明白两者的区别，参阅两篇文章后逐渐解开疑惑，在此记录

> 参考:
> 
> [docker - What is the difference between CMD and ENTRYPOINT in a Dockerfile? - Stack Overflow](https://stackoverflow.com/questions/21553353/what-is-the-difference-between-cmd-and-entrypoint-in-a-dockerfile)
> 
> [Docker 的 ENTRYPOINT 和 CMD 参数探秘 | 亚马逊AWS官方博客](https://aws.amazon.com/cn/blogs/china/demystifying-entrypoint-cmd-docker/)

# ENTRYPOINT

`entrypoint`参数我的理解是相当于一个指定一个解释器，就像shell的shebang。一切传入到容器的参数和选项都将由`entrypoint`指定的值来处理。

也就是说指定了`entrypoint`的值以后，就可以将容器本身当成一个二进制文件或者说可执行文件，在run生成容器实例时使用镜像描述的`cmd`或者命令行中覆写的`cmd`当作参数传入

如果`entrypoint`没有指定，则执行`cmd`中的第一个参数

这个参数只能出现一次

# CMD

`cmd`一般会当作`entrypoint`的参数传入。如果`entrypoint`没有指定，那么他将直接代替`entrypoint`以执行后续命令，而不是作为参数

这个参数可与出现多次，但只有最后一次指定的会生效

# 小结

也就是说，ENTRYPOINT和CMD使用是相对灵活的。对于需要处理单一任务的容器可以指定ENTRYPOINT和CMD以专注特定任务；而只使用CMD而不使用ENTRYPOINT就可以随时更改容器的使用的主程序以完成不同的任务

- ENTRYPOINT+CMD == 可执行文件+默认参数

- ENTRYPOINT == 解释器

- CMD == 直接执行命令

# 分析

## 1-拉取运行实例容器

以这个镜像[grycap/cowsay](https://hub.docker.com/layers/grycap/cowsay/latest/images/sha256-fad516b39e3a587f33ce3dbbb1e646073ef35e0b696bcf9afb1a3e399ce2ab0b?context=explore)为例，粗浅分析一下

先run两个容器，后面再对他们的配置文件进行修改:

```bash
docker run -i --name 1no_entry_point 
docker run -i --name 1entry_point 
```

有一只牛在说话，说明成功了

## 2-修改配置文件

之后再修改它们的配置文件

容器配置文件路径在`/var/lib/docker/containers/<容器id>/config.v2.json`

> 这种修改配置方法应该是比较邪道的，网上查了资料都没有对这种方法的描述

打开后发现有一坨东西缩在一行，在vim下用python工具格式化一下。在vim下输入这一小段命令:

`:%python3 -m json.tool`（会变成`:'<,'>!python3 -m json.tool`）

格式整理好了，但是每次容器启动时停止都会将其转为一行的格式且变更为原来的配置。所以需要保证这时候容器处于停止状态

`1no_entry_point`修改`Path` `Args` `Cmd` `Entrypoint`为以下值:

<img title="" src="https://dlink.host/1drv/aHR0cHM6Ly8xZHJ2Lm1zL2kvcyFBckVNT01Ec2ZXcEdnVFI1b1lpSTFuQl9DakhxP2U9TWJTRGRH.png" alt="">

`1entry_point`修改`Path` `Args` `Cmd` `Entrypoint`为以下值:

<img title="" src="https://dlink.host/1drv/aHR0cHM6Ly8xZHJ2Lm1zL2kvcyFBckVNT01Ec2ZXcEdnVE1RSUNCZWJCZWR2WERRP2U9aTVhc0dL.png" alt="">

> 对于无ENTRYPOINT只有CMD的配置，`Path`为CMD第一个字段，剩下的字段分开塞进`Args`
> 
> 对于ENTRIYPOINT和CMD都有的配置，`Path`应该也是ENTRYPOINT的第一个字段，剩下的CMD和ENTRYPOINT字段应该也要塞进`Args`里(这里尚未验证)
> 
> 另外，该配置里那段奇怪的编码字段是中文“测试”的UTF-16编码

修改完毕后运行:

```bash
➜  ~ docker start -i 1no_entry_point
 __
< 测试  >
 --
        \   ^__^
         \  (oo)\_______
            (__)\       )\/\
                ||----w |
                ||     ||
➜  ~ docker start -i 1entry_point
 __
< 测试  >
 --
        \   ^__^
         \  (oo)\_______
            (__)\       )\/\
                ||----w |
                ||     ||
```

可以看到两个效果是一样的。说到底只是一个把cowsay这个可执行文件放在`entrypoint`；一个放在`cmd`

# 总结

虽然还不是很懂，但是我认为写了ENTRYPOINT的镜像对于处理一次性文件好像比较有优势。例如这样阅后即焚的容器，处理容器内部`/etc/passwd`:

```bash
➜  ~ docker run --rm --entrypoint=/bin/cat alpine:latest /etc/passwd
root:x:0:0:root:/root:/bin/ash
bin:x:1:1:bin:/bin:/sbin/nologin
daemon:x:2:2:daemon:/sbin:/sbin/nologin
adm:x:3:4:adm:/var/adm:/sbin/nologin
lp:x:4:7:lp:/var/spool/lpd:/sbin/nologin
sync:x:5:0:sync:/sbin:/bin/sync
shutdown:x:6:0:shutdown:/sbin:/sbin/shutdown
halt:x:7:0:halt:/sbin:/sbin/halt
mail:x:8:12:mail:/var/mail:/sbin/nologin
news:x:9:13:news:/usr/lib/news:/sbin/nologin
uucp:x:10:14:uucp:/var/spool/uucppublic:/sbin/nologin
operator:x:11:0:operator:/root:/sbin/nologin
man:x:13:15:man:/usr/man:/sbin/nologin
postmaster:x:14:12:postmaster:/var/mail:/sbin/nologin
cron:x:16:16:cron:/var/spool/cron:/sbin/nologin
ftp:x:21:21::/var/lib/ftp:/sbin/nologin
sshd:x:22:22:sshd:/dev/null:/sbin/nologin
at:x:25:25:at:/var/spool/cron/atjobs:/sbin/nologin
squid:x:31:31:Squid:/var/cache/squid:/sbin/nologin
xfs:x:33:33:X Font Server:/etc/X11/fs:/sbin/nologin
games:x:35:35:games:/usr/games:/sbin/nologin
cyrus:x:85:12::/usr/cyrus:/sbin/nologin
vpopmail:x:89:89::/var/vpopmail:/sbin/nologin
ntp:x:123:123:NTP:/var/empty:/sbin/nologin
smmsp:x:209:209:smmsp:/var/spool/mqueue:/sbin/nologin
guest:x:405:100:guest:/dev/null:/sbin/nologin
nobody:x:65534:65534:nobody:/:/sbin/nologin
```
