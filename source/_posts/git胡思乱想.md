---
title: git胡思乱想
date: 2024-03-22 14:35:40
tags:
 - git
 - 折腾
categories:
 - [折腾, git]
---

因为不怎么研究code,所以git本人来说其实工具的属性十分强的，属于是能用就行。但是在写文章时遇到两个问题，稍微搜索了一下，并顺便记录感想。

# 1-关于Windows生成新密钥后ssh克隆远程仓库

Windows机器很少用作轻度作业<del>环境太杂了，分心因素太多</del>。最近在生成了一对新密钥，但是在github上传后却无法在远程拉取拉取自己的仓库，无论Windows Terminal还是Git Bash都显示类似提示:

```bash
git clone git@github.com:某仓库
Cloning into '某仓库'...
Permission denied (publickey).
fatal: Could not read from remote repository.

Please make sure you have the correct access rights
and the repository exists.
```

可是使用`ssh -T git@github.com`提示的却是正常信息，参考了两篇文章

> https://stackoverflow.com/questions/17846529/could-not-open-a-connection-to-your-authentication-agent
> 
> [bash - Github permission denied: ssh add agent has no identities - Stack Overflow](https://stackoverflow.com/questions/26505980/github-permission-denied-ssh-add-agent-has-no-identities)

在git bash里输入:

```bash
eval `ssh-agent -s`
```

其实我认为不加`-s`选项也可以，毕竟引号命令的默认基本输出是一样适用于bash的

然后再使用:

```bash
ssh-add `私钥路径`
```

这下就解决问题了，具体原因不清楚。再生成一对新密钥后再指定其拉取仓库能直接拉，原因晚点得搞清楚

# 2-关于子模块

hexo的静态页是使用git管理的，主题仓库在remote显示是作为子模块的，但是拉取到本地使用子模块命令却看不到主题仓库，本地子模块文件夹也是空的。remote端显示子模块的文件夹也是空的。使用`git ls-tree <branch> [ dir/ | file ]`或者用`git ls-files --stage`却可以看到这个文件夹确实是以create mode 160000上传的。暂时搞不明白，先记录
