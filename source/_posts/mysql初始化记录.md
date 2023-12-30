---
title: mysql初始化记录
date: 2023-12-09 01:34:49
tags: 
 - documentation
 - mysql 
categories:
 - [documentation, mysql]
---

每次在各种平台中安装mysql时总是会遇上问题，特别是初始化的时候....遇到问题的时候，中文资料的时效性和广度简直是一言难尽。不读一遍官方文档不是说不能安装，只是即使安装完也就很不爽，全世界的解决方案都不太一样....特别是对于初始化的处理。无奈只好读一遍官方文档以解答自己对安装过程中一些问题的疑惑

> 仅对mysql80负责，时效性很重要
> 
> 本文内容来源: [MySQL :: MySQL Installation Guide :: 9.1 Initializing the Data Directory](https://dev.mysql.com/doc/mysql-installation-excerpt/8.0/en/data-directory-initialization.html)

# 1、对初始化的概述

对于安装mysql后的初始化，大概有两种方法，一种是照着文档`MySQL Installation Guide`一步一步来，另外一种则是依靠官方程序`mysql_secure_installation`。此外的方法笔者就不清楚了

# 2、MySQL Installation Guide 方法

这种方法还是比较繁琐的，首先系统需要有`用户组:mysql`和`用户:mysql`(当然用其他用户组和用户名替代也可以，现有的也行，本人没实践过)，没有请创建用户及组，不同发行版的语法可能有不同，请注意

```bash
$> groupadd mysql
$> useradd -r -g mysql -s /bin/false mysql
```

## Step1-创建一个导入导出目录(非必须)

首先转移到mysql安装的所在目录，创建一个目录`mysql-files`这个目录由mysql系统变量[`secure_file_priv`](https://dev.mysql.com/doc/refman/8.0/en/server-system-variables.html#sysvar_secure_file_priv) 指定，用以导入导出操作

```bash
mkdir mysql-files
```

修改该文件夹权限，secure_file_priv需要指定的文件夹创建完成

```bash
chown mysql:mysql mysql-files
chmod 750 mysql-files
```

## Step2-初始化data目录

这一步就是初始化mysql的data目录，有两种方式:

```bash
bin/mysqld --initialize --user=mysql
bin/mysqld --initialize-insecure --user=mysql
```

`--initialize`和`--initialize-insecure`决定了初始化后，**mysql的root账户**是否会带有一个随机生成的密码，如果是 `--initialize-insecure` 则没有随机密码。`--user=mysql`决定了初始化data目录及文件在系统中的用户归属权，以root用户执行此命令注意带上这句参数。相反的，你可以以mysql用户登陆系统，并不用这句参数以初始化data目录。

- 对于Windows用户，命令二选一:
  bin\mysqld --initialize --console
  bin\mysqld --initialize-insecure --console

> 碎碎念:我选择gui

<br>

<br>

对于初始化data目录，还能带有更多参数。比如mysql无法识别正确的安装目录和数据目录，那么你要指定mysql的安装目录(basedir)和数据目录(datadir),例子如下:

```bash
bin/mysqld --initialize --user=mysql \
  --basedir=/opt/mysql/mysql \
  --datadir=/opt/mysql/mysql/data 
```

> 对于`basedir`，我的理解是mysql二进制文件目录`bin/`和其他程序组件所在地，对Windows来说这个目录意义好像更大，Linux一般都直接在`/usr`里找了。下文所说的`my.cnf`/`my.ini`的Windows全局路径就和这个目录相关

<br>

除了添加参数，也可指定一个配置文件，当初始化目录时读取此配置文件来指定以上文件夹安装的路径，例如有一个`/opt/mysql/mysql/etc/my.cnf`内容是:

```ini
[mysqld]
basedir=/opt/mysql/mysql
datadir=/opt/mysql/mysql/data
```

输入以下命令以读取这个配置文件以初始化:

```
bin/mysqld --defaults-file=/opt/mysql/mysql/etc/my.cnf \
  --initialize --user=mysql
```

> 初始化目录时，除了`--user`、`--basedir`和`--datadir`以外，不应该指定其他参数
> 
> - `my.cnf`/`my.ini`的位置是有讲究的，详见[option file](https://dev.mysql.com/doc/refman/8.0/en/option-files.html)。每次mysql启动都会读取这个文件的默认目录以加载配置，这个例子的文件路径显然不在option file的目录中，所以我猜这个文件是一次性的
> 
> - 另外，对于server，即使mysql运行时指定了`--defaults-file`的参数,mysql仍然会读取另外一个在datadir的配置文件`mysqld-auto.cnf`

## Step-3小问题

对于`--initialize-insecure`建议第一件事就是加密码:

```sql
ALTER USER 'root'@'localhost' IDENTIFIED [WHIT auth_plugin] BY 'yourPassword'
```

<br>

关于时区表，得手动配置；
<br>

[`init_file`](https://dev.mysql.com/doc/refman/8.0/en/server-system-variables.html#sysvar_init_file)系统变量可以指定一个文件存放sql语句，mysql启动时执行；

## Step4-自动安装非对称加密（非必要）

此文件用于自动配置支持加密连接，直接运行二进制文件安装:

```bash
bin/mysql_ssl_rsa_setup
```

安装后链接默认加密，除非指定用户为`REQUIRE NONE`

官方文档:[docForMysql8.0](https://dev.mysql.com/doc/refman/8.0/en/mysql-ssl-rsa-setup.html)

> 对于mysql8.0.34来说，这个模块已经<s>爆金币</s>过时，详见: [MySQL :: MySQL 8.0 Reference Manual :: 6.3.3.1 Creating SSL and RSA Certificates and Keys using MySQL](https://dev.mysql.com/doc/refman/8.0/en/creating-ssl-rsa-files-using-mysql.html) 笔者用的就是此版本且未安装该模块情况。实测dbeaver第一次链接数据库需要开启allowPublicKeyRetrieval选项，否则会报错

# 3、mysql_secure_installation 方法

这个就简单了，找到这个文件执行就完事了，当然也能指定参数，详情: [MySQL :: MySQL 8.0 Reference Manual :: 4.4.2 mysql_secure_installation — Improve MySQL Installation Security](https://dev.mysql.com/doc/refman/8.0/en/mysql-secure-installation.html)
