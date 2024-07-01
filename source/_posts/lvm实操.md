---
title: lvm实操
tags:
  - linux
  - lvm
categories:
  - - linux
    - lvm
date: 2024-03-25 23:54:20
---

lvm一直没找到使用场景，直到我的小黑从朋友那转了回来 。里面插的是两张小容量固态硬盘，加起来350G左右。放到今天分开来用可以说食之无味弃之可惜，合在一起那就爽了。这下除了RAID，那就是LVM的战场了

接下来就是livecd启动，换源sshd一气呵成接着就是分区

# 操作

## 1-查看当前存储设备

`lsblk`稍微看一下，很好，已经分好区了，等等全给格了:

<img title="" src="https://telegraph.7cmb.com/file/7e9d33c18d91da872f95f.png" alt="lsblk">

## 2-分区

本设备已经设置了UEFI启动，遂`fdisk`把所有硬盘的`label`设置为GPT。除了EFI分区留1G放引导外其他通通做成`Linux LVM`格式:

<img title="" src="https://telegraph.7cmb.com/file/a63bb31607761bb689ebb.png" alt="partition1">

<img title="" src="https://telegraph.7cmb.com/file/8df2cc9e1e4dad2769de0.png" alt="partition2">

## 3-LVM基本命令

LVM初始化到使用基本有三个步骤:

- 给上述分好的LVM分区设置为PV(物理卷)

- PV(物理卷)加入到VG(卷组)

- VG(卷组)仿佛一块新硬盘一样，但是由多个PV(物理卷)组成，其空间可以呗其自由划分为LVM的LV(逻辑卷)

一旦第三步完成，有了LV，那么这块硬盘就和传统磁盘一样格式化文件系统就能正常使用了

LVM操作无非三种命令:

- `pvcreate` `pvremove` `pvmove`

- `vgcreate` `vgremove` `vgmove`

- `lvcreate` `lvremove` `lvmove`

有了这三种命令就能创建、删除、迁移对应空间

<br>

<br>

下列命令对应查看LVM信息

- `pvs` `vgs` `lvs`    # 简述

- `pvdisplay` `vgdisplay` `lvdisplay`    # 详细

这两种命令LVM信息的状态略详

接着就是实操

## 4-实操

首先给刚刚的LVM分区打上`PV`标头:

```bash
root@debian:~# pvcreate /dev/sda2 /dev/sdb1
  Physical volume "/dev/sda2" successfully created.
  Physical volume "/dev/sdb1" successfully created.
```

接着给上述`PV`加入到一个`VG`:

```bash
root@debian:~# vgcreate vg_p1  /dev/sda2
  Volume group "vg_p1" successfully created 
root@debian:~# vgextend vg_p1 /dev/sdb1
  Volume group "vg_p1" successfully extended
```

检查容量:

```bash
root@debian:~# vgs
  VG    #PV #LV #SN Attr   VSize   VFree
  vg_p1   2   0   0 wz--n- 356.71g 356.71g
```

在`VG`中划分`LV`，比较像传统硬盘分区:

```bash
root@debian:~# lvcreate -l 100%FREE vg_p1 -n lv_p1
WARNING: ext4 signature detected on /dev/vg_p1/lv_p1 at offset 1080. Wipe it? [y/n]: y
  Wiping ext4 signature on /dev/vg_p1/lv_p1.
  Logical volume "lv_p1" created.
```

现在`lv_p1`就是一个逻辑分区，离可用只差格式化，下面检查LV是否正常划分:

```bash
root@debian:~# ls /dev/vg_p1/
lv_p1 
root@debian:~# lvs
  LV    VG    Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  lv_p1 vg_p1 -wi-a----- 356.71g
root@debian:~# lsblk
NAME            MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
loop0             7:0    0   2.5G  1 loop /usr/lib/live/mount/rootfs/filesystem.squashfs
                                          /run/live/rootfs/filesystem.squashfs
sda               8:0    0 238.5G  0 disk
├─sda1            8:1    0     1G  0 part
└─sda2            8:2    0 237.5G  0 part
  └─vg_p1-lv_p1 254:0    0 356.7G  0 lvm
sdb               8:16   0 119.2G  0 disk
└─sdb1            8:17   0 119.2G  0 part
  └─vg_p1-lv_p1 254:0    0 356.7G  0 lvm
sdc               8:32   1  14.9G  0 disk
├─sdc1            8:33   1     3G  0 part /usr/lib/live/mount/medium
│                                         /run/live/medium
└─sdc2            8:34   1     5M  0 part
```

可以看到各大工具都能看到`vg_p1`卷组下的`lv_p1`(vg_p1/lv_p1)

最后分区挂载:

```bash
root@debian:~# mkfs.ext4 /dev/vg_p1/lv_p1
mke2fs 1.47.0 (5-Feb-2023)
Discarding device blocks: done
Creating filesystem with 93509632 4k blocks and 23379968 inodes
Filesystem UUID: 637bb599-9c46-458a-9937-b0721bd8c8b5
Superblock backups stored on blocks:
        32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632, 2654208,
        4096000, 7962624, 11239424, 20480000, 23887872, 71663616, 78675968

Allocating group tables: done
Writing inode tables: done
Creating journal (262144 blocks): done
Writing superblocks and filesystem accounting information: done
```

完成
