---
title: vcenter6.7搭配esxi6.7踩坑记录
tags:
  - vmware
categories:
  - - vmware
    - vcenter
date: 2024-11-12 19:59:09
---


借着工作遇到的坑水一篇文章，没有有营养的内容，只做个记录

在没有可用的内部 dns 服务器下，用 VCSA GUI 在 esxi6.7 安装 vcenter 时，不但需要超长时间，还会在 60% 处出现类似于 `firstboot` `vpxd` 的错误。各种排查看 KB 咕噜咕噜无果的情况下，最后被高人点拨，在 vcenter 的 VM 里添加一条host:
```bash
<vcenter VM ip> local.local
```

之后事情就解决了......感到震惊无力又无语，希望帮到有困难的人

UPDATE 2024.11.13
好像找到说明了，不确定，先贴上来:
<img alt='VMdoc' title='VMdoc' src='https://s2.loli.net/2024/11/13/xyUXbo25IFeVTQJ.png'>
