---
title: PXE服务部署
date: 2023-12-24 21:50:34
tags:
- linux
- pxe
categories:
- [pxe, linux]

---

装系统实在是令人熟悉，但是PXE却像个熟悉的陌生人，除了名字以外却很少接触。最近因为实验需要同时部署多台设备，单机部署实在是太麻烦了，复制虚拟机改配置治标不治本。本着偷懒的想法遂有了部署PXE服务器的想法。本篇文章记录了vbox下部署PXE服务器的经过

# 一、基本概念

## 概述

此处引用这篇文章[浅谈pxe和ipxe](https://zhuanlan.zhihu.com/p/591334237)。大体来说pxe的启动方式对应的是bios启动+mbr分区表，自检过后，计算机控制权移交给mbr，mbr找到活动分区，加载活动分区的pbr到内存，pbr将指向一个引导程序，如bootmgr，最后由引导程序启动操作系统；而ipxe(也能兼容bios模式)对应的是uefi启动+gpt分区表，uefi将直接寻找efi分区，如boot分区或者esp分区。找到efi分区中的`EFI`后缀的引导程序，由这个引导程序启动系统。

pxe和ipxe使用前必须有一台支持pxe配置选项的dhcp服务器和一台tftp服务器。dhcp服务器将在dhcp阶段可以先判断是否有boot request，再根据客户端机器架构(arch)指定option 66(TFTP Server Name\Next-To-Server)和option 67(Filename) ,从而为客户端指定tftp服务器地址和引导程序的路径

> wireshark分析看不到相关选项的字段，只能找到next-to-server和filename这两个参数

对于pxe和ipxe来说，引导程序和一个临时操作系统是必要的，这个引导程序和临时操作系统一共分为三个部分:`pxelinux.0` \ `EFI文件`、`vmlinuz`、`initrd.img`。

`pxelinux.0` \ `EFI文件`是启动这个临时操作系统的引导程序，内核`vmlinuz`和临时文件系统`initrd.img`组成一个临时操作系统。引导程序和相关配置文件先通过tftp服务器下载，完成后引导程序通过配置文件设定的方式启动，当选定选项时再通过tftp服务器下载内核和临时文件系统(如果使用pxelinux或grub的话，选项通过`pxelinux.cfg/default`或`grub.cfg`指定)

这个临时操作系统承担了找到一个操作系统并下载、加载的任务。结束这个阶段将正式进入系统安装的环节

## 结论

所以在本个过程中，需要用到的服务组件有:dhcp服务器、tftp服务器、引导程序、临时操作系统(`vmlinuz`和`initrd.img`)和一个符合pxe启动格式的系统文件

此处为了方便将搭建一个all in one的pxe系统，这些组件可以分开协同运作，环境:

```
# 由于没有进一步的实践，所以下面的实践只以almalinux9为例
# 系统镜像文件部署方式可参考关键字"inst.repo"和"inst.stage2"
# 本文镜像所在位置部署在本地http文件服务器即下面httpd(apache)
# 以下操作用户均为root
操作系统:almalinux9
dhcp服务:dhcp-server
tftp服务:tftp-server
引导程序:syslinux    #pxelinux是syslinux的程序之一
符合pxe启动格式的系统文件:放置于httpd(apache)服务器中
```

# 二、实践

## 1、准备必要程序

安装`apache`、`dhcp-server`、`tftp-server`、`syslinux`:

```
dnf install httpd dhcp-server tftp-server syslinux
```

下载需要安装的镜像并挂载:

```
mkdir -p /mnt/cdrom
mount -t iso9660 AlmaLinux9ISO/AlmaLinux-9.3-x86_64-minimal.iso /mnt/cdrom -o ro,loop
```

查看挂载点:

```
ls -a /mnt/cdrom/
.  ..  .discinfo  .treeinfo  EFI  EULA  LICENSE  Minimal  RPM-GPG-KEY-AlmaLinux-9  TRANS.TBL  extra_files.json  images  isolinux  media.repo
```

## 2、准备引导程序到tftp服务器

这个步骤将准备引导程序到tftp服务器文件夹

<br>

准备bios下的引导程序pxelinux:

```
mkdir -p /var/lib/tftpboot/pxelinux.cfg
cp /usr/share/syslinux/{pxelinux.0,menu.c32,vesamenu.c32,ldlinux.c32,libcom32.c32,libutil.c32} /var/lib/tftpboot/
```

准备uefi启动下的引导程序grub:

```
# EFI文件只有执行权限还不够，需要有读取权限
cp -rp /mnt/cdrom/EFI /var/lib/tftpboot/
chmod -R 755 /var/lib/tftpboot/EFI
```

开启并设置自启动tftp服务:

```
systemctl enable --now tftp
```

 查看当前tftp文件夹内容:

<img title="" src="https://dlink.host/1drv/aHR0cHM6Ly8xZHJ2Lm1zL2kvcyFBckVNT01Ec2ZXcEdnUVg5Z2JzdzlXdk03Y19VP2U9TVBEYjJN.jpg" alt="">

为防火墙添加规则:

```
firewall-cmd --add-service=tftp
```

此时引导程序已经在tftp文件夹中，但是配置文件还没有修改，还需要进一步与修改之后才能启动临时系统

## 3、准备dhcp服务器

如果在vbox中部署pxe，那么需要先禁用vbox中相关网卡的的dhcp服务器，然后再重启宿主机。不重启宿主机，那么vbox的dhcp服务器将不会停止工作

编辑`/etc/dhcp/dhcpd.conf`配置文件，将以下内容替换为符合自身网络环境的配置:

```
option architecture-type code 93 = unsigned integer 16;

subnet 192.168.56.0 netmask 255.255.255.0 {
  option routers 192.168.56.50;
  option domain-name-servers 192.168.1.1;
  range 192.168.56.100 192.168.56.254;
  class "pxeclients" {
    match if substring (option vendor-class-identifier, 0, 9) = "PXEClient";
    next-server 192.168.56.50;    # tftp服务器地址
          if option architecture-type = 00:07 { # x64架构 uefi固件
            filename "EFI/BOOT/grubx64.efi"; # 以tftp服务目录为根的相对路径
                      # 此路径相当于/var/lib/tftpboot/EFI/BOOT/grubx64.efi
          }
          else {
            filename "pxelinux.0";    # 同上
          }
  }
```

以上修改自[红帽的配置文件](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/9/html/performing_a_standard_rhel_9_installation/assembly_preparing-for-your-installation_installing-rhel#configuring-the-dhcpv4-server-for-http-and-pxe-boot_preparing-for-a-network-install), 个人认为能结合[fedora的配置文件](https://docs.fedoraproject.org/zh_Hans/fedora/f36/install-guide/advanced/Network_based_Installations/#pxe-dhcpd)进行参考

开启并设置自启动dhcp服务:

```
systemctl enable --now dhcpd
```

为防火墙添加规则:

```
firewall-cmd --add-service=dhcp
```

客户机网卡可以释放地址再重新申请，以验证dhcpd服务是否正常运行

## 4、准备httpd(apache)文件服务器以部署系统镜像文件

修改`/etc/httpd/conf/httpd.conf`中`Listen :80`为`Listen 0.0.0.0:80`

将挂载的镜像文件内容复制到`/var/www/html`文件夹下:

```
mkdir /var/www/html/almalinux
cp -rp /mnt/cdrom /var/www/html/almalinux
# 查看修改后内容
ls -ahl /var/www/html/almalinux/cdrom/
total 56K
dr-xr-xr-x. 6 root root 4.0K Nov 11 04:59 .
drwxr-xr-x. 3 root root   19 Dec 21 21:40 ..
-r--r--r--. 1 root root   43 Nov 11 04:59 .discinfo
-r--r--r--. 1 root root 1.4K Nov 11 04:59 .treeinfo
dr-xr-xr-x. 3 root root   18 Nov 11 04:59 EFI
-r--r--r--. 1 root root  290 Nov 11 04:59 EULA
-r--r--r--. 1 root root  18K Nov 11 04:59 LICENSE
dr-xr-xr-x. 4 root root   38 Nov 11 04:59 Minimal
-r--r--r--. 1 root root 3.1K Nov 11 04:59 RPM-GPG-KEY-AlmaLinux-9
-r--r--r--. 1 root root 1.6K Nov 11 04:59 TRANS.TBL
-r--r--r--. 1 root root  719 Nov 11 04:59 extra_files.json
dr-xr-xr-x. 3 root root   76 Nov 11 04:59 images
dr-xr-xr-x. 2 root root 4.0K Nov 11 04:59 isolinux
-r--r--r--. 1 root root   86 Nov 11 04:59 media.repo
```

开启并设置自启动httpd服务:

```
systemctl enable --now httpd
```

为防火墙添加规则:

```
firewall-cmd --add-service=http
```

检查服务是否自启动:

```
systemctl status httpd
```

也可直接浏览器访问测试:

<img title="" src="https://dlink.host/1drv/aHR0cHM6Ly8xZHJ2Lm1zL2kvcyFBckVNT01Ec2ZXcEdnUVN0X055bWNtX05BdmV6P2U9Q2VTbTNM.jpg" alt="">

<img title="" src="https://dlink.host/1drv/aHR0cHM6Ly8xZHJ2Lm1zL2kvcyFBckVNT01Ec2ZXcEdnUVlFa0ZEVHBCRkQ3Q0JsP2U9UDFaWDQ4.jpg" alt="">

> vbox下可能要重启一下虚拟机，某些配置才能生效.....
> 
> 如果不想搭建http服务器的话也能用其他方式获取系统文件
> 
> 参考:
> 
> [RHEL 安装程序的引导选项 Red Hat Enterprise Linux 9 | Red Hat Customer Portal](https://access.redhat.com/documentation/zh-cn/red_hat_enterprise_linux/9/html-single/boot_options_for_rhel_installer/index#installation-source-boot-options_kickstart-and-advanced-boot-options)
> 
> [Anaconda Boot Options &#8212; Anaconda 40.15 documentation](https://anaconda-installer.readthedocs.io/en/latest/boot-options.html)

## 5、配置引导程序

为`pxelinux.0`配置`/var/lib/tftpboot/pxelinux.cfg/default`:

```
default vesamenu.c32
prompt 1
timeout 600

label linux
menu label ^Install AlmaLinux9(Minimal) 64-bit
kernel pxeboot/vmlinuz    # 以tftp服务目录为根的相对路径
append initrd=pxeboot/initrd.img inst.repo=http://192.168.56.50/almalinux/cdrom

label local
menu label Boot from ^local drive
menu default
localboot 0xffff
```

为 `grub`配置`/var/lib/tftpboot/EFI/BOOT/grub.cfg`:

```
set default="1"

function load_video {
  insmod efi_gop
  insmod efi_uga
  insmod video_bochs
  insmod video_cirrus
  insmod all_video
}

load_video
set gfxpayload=keep
insmod gzio
insmod part_gpt
insmod ext2

set timeout=60
### END /etc/grub.d/00_header ###

search --no-floppy --set=root -l 'AlmaLinux-9-3-x86_64-dvd'

### BEGIN /etc/grub.d/10_linux ###
menuentry 'Install AlmaLinux 9.3' --class fedora --class gnu-linux --class gnu --class os {
        linuxefi pxeboot/vmlinuz inst.stage2=http://192.168.56.50/almalinux/cdrom/
        initrdefi pxeboot/initrd.img
}
#       menuentry 'Test this media & install AlmaLinux 9.3' --class fedora --class gnu-linux --class gnu --class os {
#               linuxefi /images/pxeboot/vmlinuz inst.stage2=hd:LABEL=AlmaLinux-9-3-x86_64-dvd rd.live.check quiet
#               initrdefi /images/pxeboot/initrd.img
#       }
#       submenu 'Troubleshooting -->' {
#               menuentry 'Install AlmaLinux 9.3 in text mode' --class fedora --class gnu-linux --class gnu --class os {
#                       linuxefi /images/pxeboot/vmlinuz inst.stage2=hd:LABEL=AlmaLinux-9-3-x86_64-dvd inst.text quiet
#                       initrdefi /images/pxeboot/initrd.img
#               }
#               menuentry 'Rescue a AlmaLinux system' --class fedora --class gnu-linux --class gnu --class os {
#                       linuxefi /images/pxeboot/vmlinuz inst.stage2=hd:LABEL=AlmaLinux-9-3-x86_64-dvd inst.rescue quiet
#                       initrdefi /images/pxeboot/initrd.img
#               }
#       }
```

自此，pxe和ipxe配置完成

> grub.cfg还有更简短的写法，比如红帽的例子[Chapter 2. Preparing for your RHEL installation Red Hat Enterprise Linux 9 | Red Hat Customer Portal](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/9/html/performing_a_standard_rhel_9_installation/assembly_preparing-for-your-installation_installing-rhel#configuring-a-tftp-server-for-uefi-based-clients_preparing-for-a-network-install)，也能用而且没什么区别
> 
> inst.repo和inst.stage2选项的区别可以参考:
> 
> [RHEL 安装程序的引导选项 Red Hat Enterprise Linux 9 | Red Hat Customer Portal](https://access.redhat.com/documentation/zh-cn/red_hat_enterprise_linux/9/html-single/boot_options_for_rhel_installer/index#installation-source-boot-options_kickstart-and-advanced-boot-options)
> 
> [Anaconda Boot Options &#8212; Anaconda 40.15 documentation](https://anaconda-installer.readthedocs.io/en/latest/boot-options.html)
> 
> 本人的理解是都一样，inst.stage2会忽略寻找packages这个文件夹

## 6、vbox部署的坑

- 配置完httpd服务和防火墙设置后重启服务没有立即生效，需要重启虚拟机

- 客户机的内存应该调大一点，网上说的2g不够，此次实验客户机为4g内存

- 客户机应该调整更多显存，否则大概率到安装界面后只有一个鼠标

- 防火墙设置要永久生效需要添加`--permanent`参数

- efi启动问题很多，无法吐槽

# 三、总结

pxe结束。为了偷懒，下一步就是kickstart
