---
title: selinux基本使用
date: 2024-03-22 16:54:10
tags:
 - selinux
 - linux
categories:
 - [linux, selinux]
---

部署web服务的时候遇到关于读写的问题，debug半天在selinux发现了端倪。本来应该更早记录的，现在属于是回忆＋再看一遍文档了

> PS:又得回忆一遍细节，还得开机验证有没有记错，好痛苦
> 
> <del>中间的时间作者玩洋垃圾去了</del>

# 概述

遇到该问题的场景是在nginx部署php应用，DAC配置为777的情况下依旧无法读写。首先排除是php-fpm的问题。如果php-fpm配置错误，nginx直接访问php文件会404，而遇到的场景是php读写时才会404

虽然将selinux设置为宽松模式或者干脆关闭selinux将不用思考这个问题，但是还是想不关闭selinux的情况下实现目标

## 目的

需要在selinux在`enforcing`模式下修改上下文以实现php后端对文件的修改

## 先决条件

nginx和php-fpm配置正确，DAC权限配置正确。且所有进程均以正确身份运行，如果有NFS服务则需要配置匿名客户映射到共享目录的所有者

## 流程

查看nginx或者说web服务目录需要配置的selinux上下文，为web应用实例所在目录配置该上下文，可按需临时更改或者永久更改。之后配置web服务布尔值（selinux策略）

# 实操

## 1-修改web实例目录上下文

首先查看目标上下文是什么，以nginx的示例文件夹作为参考:

```bash
[baka@test1-web-sql lib]$ sudo semanage fcontext --list |grep /usr/share/nginx/html
/usr/share/nginx/html(/.*)?                        all files          system_u:object_r:httpd_sys_content_t:s0
```

可以看到，web实例需要`type`的上下文为`httpd_sys_content_t`

假设我需要一个discuz应用，实例文件夹在`/usr/share/discuz`:

```bash
[baka@test1-web-sql lib]$ ls -Z /usr/share/nginx
              system_u:object_r:usr_t:s0 discuz                system_u:object_r:usr_t:s0 modules
system_u:object_r:httpd_sys_content_t:s0 html
```

可以看到默认`type`上下文默认是`usr_t`

使用`chcon`临时修改上下文:

```bash
sudo chcon -R -t httpd_sys_content_t /usr/share/nginx/discuz
```

临时指的是`restorecon`能够将这个方法修改的上下文给恢复过来，将其变为默认状态

如果需要永久记录则使用`semanage fcontext`命令并配合`restorecon`刷新上下文:

```bash
sudo semanage fcontext -a -t httpd_sys_content_t '/usr/share/nginx/discuz(/.*)?'
sudo restorecon -R /usr/share/nginx/discuz/
```

以上两种方法二选一皆能达到下面效果:

```bash
[baka@test1-web-sql lib]$ ls -Z /usr/share/nginx
system_u:object_r:httpd_sys_content_t:s0 discuz                system_u:object_r:usr_t:s0 modules
system_u:object_r:httpd_sys_content_t:s0 html
```

可以看到目标上下文已经应用

## 2-修改selinux布尔值(策略)

这时候按道理来说依旧无法使用cgi(该例子为php-fpm)读写该目录，所以需要修改策略

查看并筛选当前`httpd`策略:

```bash
sudo semanage boolean -l | grep httpd
```

这时候需要找到`httpd_unified`和`httpd_enable_cgi`并永久设置为1:

```bash
sudo setsebool -P httpd_unified on
sudo setsebool -P httpd_enable_cgi 1
```

> httpd_unified启用后，此布尔值允许 `httpd_t` 完全访问所有 `httpd` 类型（即执行、读取或写入 sys_content_t）。禁用后，将区分只读、可写入或可执行的 Web 内容。禁用此布尔值可确保额外的安全级别，但增加了管理开销，即必须根据每个布尔值应具有的文件访问权限单独标记脚本和其他 Web 内容。
> 
> 
> 
> 引用自——[13.3. 布尔值 Red Hat Enterprise Linux 7 | Red Hat Customer Portal](https://access.redhat.com/documentation/zh-cn/red_hat_enterprise_linux/7/html/selinux_users_and_administrators_guide/sect-managing_confined_services-the_apache_http_server-booleans)

这时web实例就能正常使用了

参考:

> [使用 SELinux Red Hat Enterprise Linux 8 | Red Hat Customer Portal](https://access.redhat.com/documentation/zh-cn/red_hat_enterprise_linux/8/html-single/using_selinux/index)
> 
> [部分 I. SELinux Red Hat Enterprise Linux 7 | Red Hat Customer Portal](https://access.redhat.com/documentation/zh-cn/red_hat_enterprise_linux/7/html/selinux_users_and_administrators_guide/part_i-selinux)
