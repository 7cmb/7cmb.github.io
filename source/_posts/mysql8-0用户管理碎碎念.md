---
title: mysql8.0用户管理碎碎念
date: 2023-12-09 01:35:20
tags: 
 - document
 - mysql 
categories:
 - [document, mysql]
---

首先从mysql8.04版本开始，mysql更改了默认身份插件从 mysql_native_password​  改为 caching_sha2_password​ 。老客户端可能会出问题。<s>但是这不是距离现在挺久了吗</s>

但是以上和本文正题没关

# 1、创建远程用户相关

```sql
CREATE USER 'finley'@'localhost'
  IDENTIFIED BY 'password';
GRANT ALL
  ON *.*
  TO 'finley'@'localhost'
  WITH GRANT OPTION;

CREATE USER 'finley'@'%.example.com'
  IDENTIFIED BY 'password';
GRANT ALL
  ON *.*
  TO 'finley'@'%.example.com'
  WITH GRANT OPTION;

CREATE USER 'admin'@'localhost'
  IDENTIFIED BY 'password';
GRANT RELOAD,PROCESS
  ON *.*
  TO 'admin'@'localhost';

CREATE USER 'dummy'@'localhost';
```

在这个例子里，用户名为finley的两个账号都是superuser，拥有能做所有事情的权限。`'finley'@'hostlocal'`是一个本地账户，仅限在本地机器链接；`'finley'@'%.example.com'`是一个远程账户，其中符号`%`类似于反掩码，匹配该域名的子域，如只有`%`，它将匹配本机的任何地址

<br>

<br>

如果user table有**匿名帐户**就要特别注意，如需创建一个远程账户，必须有一个与其用户名对应的本地账户。如上述用户名为finley的账户，一共两个。以上述用户为例，如果只有远程账户finley，那么当finley在本地登录数据库时将会被认为是匿名帐户而不是finley，原因是匿名帐户的host值更为具体而在用户表的排序中更靠前。因此创建远程账户时为了保险，需要为其额外创建一个本地账户

`CREATE USER 'dummy'@'localhost'`账户没有任何权限，但是不推荐这样做

# 2、基本权限管理

```sql
CREATE USER 'custom'@'localhost'
  IDENTIFIED BY 'password';
GRANT ALL
  ON bankaccount.*
  TO 'custom'@'localhost';

CREATE USER 'custom'@'host47.example.com'
  IDENTIFIED BY 'password';
GRANT SELECT,INSERT,UPDATE,DELETE,CREATE,DROP
  ON expenses.*
  TO 'custom'@'host47.example.com';

CREATE USER 'custom'@'%.example.com'
  IDENTIFIED BY 'password';
GRANT SELECT,INSERT,UPDATE,DELETE,CREATE,DROP
  ON customer.addresses
  TO 'custom'@'%.example.com';
```

正如上述语句，`GRANT...ON <DATABASE>.[TABLE]`可以用户权限限定在某个数据库/表里

GRANT更多用法: [MySQL :: MySQL 8.0 Reference Manual :: 13.7.1.6 GRANT Statement](https://dev.mysql.com/doc/refman/8.0/en/grant.html)

# 3、关于 [Password Validation Component](https://dev.mysql.com/doc/refman/8.0/en/validate-password.html)

密码可用性组件，有时候并不会自动安装

如需安装则需要确保设备的相关目录有此文件:

```
$> cd /path/to/mysql/lib/plugin/
$> ls component_v*
component_validate_password.so
```

再确认数据库全局变量`plugin_dir`数值是否指向插件目录:

```sql
mysql> SELECT @@plugin_dir;
+--------------------------------------------+
| @@plugin_dir                               |
+--------------------------------------------+
| /path/to/mysql/lib/plugin/                 |
+--------------------------------------------+
```

接着安装:

```sql
mysql> INSTALL COMPONENT 'file://component_validate_password';
```

安装后组件将会立刻开启。这个语句会立即加载安装模块，然后将其注册在`mysql.component`这张系统表里，因此它能在后续中当服务启动时被加载

> **使用 `INSTALL`语句需要有`INSERT`权限**，因为它注册过程需要添加一行语句到上述系统表中

<br>

<br>

安装过后就可以对与这个组件相关的系统变量进行配置，无论是写在[option file](https://dev.mysql.com/doc/refman/8.0/en/option-files.html)当中还是用`SET`语句。**当然，因为这是个全局系统变量，使用`SET`语句配置需要用户有[`SYSTEM_VARIABLES_ADMIN`](https://dev.mysql.com/doc/refman/8.0/en/server-system-variables.html)这个权限**。以下是默认配置也是可供option_file使用的选项:

```ini
validate_password.policy=1
validate_password.length=8
validate_password.number_count=1
validate_password.mixed_case_count=1
validate_password.special_char_count=1
validate_password.check_user_name=1
```

这里记录了validate_password.policy参数对该组件的影响，不同的值将改变检查的性能:

| Policy          | Tests Performed                                                                   |
| --------------- | --------------------------------------------------------------------------------- |
| `0` or `LOW`    | Length                                                                            |
| `1` or `MEDIUM` | Length; numeric, lowercase/uppercase, and special characters                      |
| `2` or `STRONG` | Length; numeric, lowercase/uppercase, and special characters; dictionary<br> file |

<br>

<br>

如需检查当前组件的相关变量情况:

```sql
SHOW VARIABLES LIKE 'validate_password.%';
```

# 4、检查用户权限和属性

检查用户权限可用以下命令:

```sql
mysql> SHOW GRANTS FOR 'admin'@'localhost';
+-----------------------------------------------------+
| Grants for admin@localhost                          |
+-----------------------------------------------------+
| GRANT RELOAD, PROCESS ON *.* TO `admin`@`localhost` |
+-----------------------------------------------------+
```

**此命令需要登录用户对`mysql`具有`SELECT`权限**

<br>

<br>

检查用户属性可用以下命令:

```sql
mysql> SET print_identified_with_as_hex = ON;
mysql> SHOW CREATE USER 'admin'@'localhost'\G
*************************** 1. row ***************************
CREATE USER for admin@localhost: CREATE USER `admin`@`localhost`
IDENTIFIED WITH 'caching_sha2_password'AS 0x24412430303524301D0E17054E2241362B1419313C3E44326F294133734B30792F436E77764270373039612E32445250786D43594F45354532324B6169794F47457852796E32
REQUIRE NONE PASSWORD EXPIRE DEFAULT ACCOUNT UNLOCK
PASSWORD HISTORY DEFAULT
PASSWORD REUSE INTERVAL DEFAULT
PASSWORD REQUIRE CURRENT DEFAULT
```

`SET print_identified_with_as_hex = ON;`(available as of MySQL 8.0.17)这个系统变量的设置是因为密码字段存储的哈希值会将不可打印字符打印为十六进制字符串而不是常规字符串

**此命令需要登录用户对`mysql`具有`SELECT`权限**

# 5、撤销权限

撤销权限语句`revoke`和`GRANT`类似，可控制用户对数据库或者表的权限，特别的一点就是TO变成了FROM:

```sql
//撤销全局权限
REVOKE ALL
  ON *.*
  FROM 'finley'@'%.example.com';

REVOKE RELOAD
  ON *.*
  FROM 'admin'@'localhost';

//撤销expenses这个数据库拥有的相关权限
REVOKE CREATE,DROP
  ON expenses.*
  FROM 'custom'@'host47.example.com';

//撤销customer.addresses这张表的相关权限
REVOKE INSERT,UPDATE,DELETE
  ON customer.addresses
  FROM 'custom'@'%.example.com';
```

# 6、删除用户

这个简单，示例代码:

```sql
DROP USER 'finley'@'localhost';
DROP USER 'finley'@'%.example.com';
DROP USER 'admin'@'localhost';
DROP USER 'dummy'@'localhost';
```
