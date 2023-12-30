---
title: 利用python抓取分析数据包并存入mysql
date: 2023-11-30 17:06:36
tags:
 - network
 - scapy 
 - python
categories:
 - [python, scapy]
 - [network, scapy]
---

# 1、先决条件

本机配置好python及mysql运行环境，并且安装好python相关模块`scapy` `pymysql` 

# 2、抓包及分析

抓包功能使用的是scapy中的sniff()函数，使用方法已经记录在前篇: [scapy小记pt.4-嗅探](https://7cmb.com/scapy%E5%B0%8F%E8%AE%B0pt-4/)，本文一定程度上算后篇

# 3、建立数据库及表

登陆数据库后，建立并指定相关数据库后，建立一张表，提前规划好数据格式

```sql
CREATE DATABASE testsql;

USE testsql;

CREATE TABLE `sniffTable` (

`index` int unsigned AUTO_INCREMENT,

`pktType` char(5),

`capTime` datetime,

`srcIP` char(16),

`dstIP` char(16),

`domain` varchar(128),

PRIMARY KEY(`index`)

)DEFAULT CHARSET=utf8mb4;
```

| name    | dataType                 |
|:-------:|:------------------------:|
| index   | u_int()   AUTO_INCREMENT |
| pktType | char(5)                  |
| capTime | datetime                 |
| srcIP   | char(16)                 |
| dstIP   | char(16)                 |
| domain  | varchar(128)             |

# 4、整体代码

database.py

```python
import pymysql.cursors
import sys
class linkSql:
    flagConnected=False
    def __init__(self):
        self.__conn=None
        conf={
            "host":"localhost",
            "user":"root",
            "database":"testsql",
            "password":"yourPassWord",
            "charset":"utf8mb4"
        }

        for i in ["host","user","database","password","password","charset"]:
            if i not in conf.keys():
                exit("MISSING ARGUMENT \"database.linkSql.conf[%s]\""%(i))

        try:
            self.__conn=pymysql.connect(
                host=conf["host"],
                user=conf["user"],
                database=conf["database"],
                password=conf["password"],
                charset=conf["charset"],
                cursorclass=pymysql.cursors.DictCursor
            )
            self.cursors=self.__conn.cursor()
            self.flagConnected=True
            print("database \"%s\" conneted"%(conf["database"]))
        except:
            sys.exit("failed to conneted \"%s\""%(conf["database"]))

    # table为 数据库内的表名 type:字符串；        dataList为 包含所有数据列表的列表 type:list
    def assembleSqlText(self,table,dataList):
        sqlHead="INSERT INTO " + table + " (pktType,capTime,srcIP,dstIP,`domain`) VALUES ("
        sqlTail="),("
        workingStr=sqlHead
        for i in dataList:
            #print(i)
            for j in i:
                workingStr=workingStr + "\"" + j + "\"" + ","
            workingStr=workingStr[:-1]
            workingStr+=sqlTail
        fullSqlText=workingStr[:-2]
        return fullSqlText
    def exec(self,sqlCmd):
        self.cursors.execute(sqlCmd)
        self.__conn.commit()

    # 销毁对象时关闭数据库连接
    def __del__(self):
        try:
            self.__conn.close()
        except pymysql.Error as e:
            pass

    # 关闭数据库连接
    def close(self):
        self.__del__()
```

<br>
<br>

sniffResults.py

```python
from scapy.all import *
from scapy.layers.http import *
import time
load_layer("tls")
scapy.config.Conf.sniff_promisc=True         # 网卡工作模式设置为promisc，如果该主机网卡为局域网转发接口，理论上能截取经过的数据包

# 嗅探函数，得到目标数据包

def getDataList():
    dataList=[]
    def sniffMain():
        argv_bpf = "tcp dst port 80 or 443"  # 伯克利包过滤器，如果需要监听别的web端口，即使加载了http模块，http fields也将被raw
                                             # 取代，但是raw里的内容将被自动解析为人类可读的bytes类型
        plist = sniff(filter=argv_bpf,
                      lfilter=lambda x: ((x.haslayer(TLS_Ext_ServerName) or x.haslayer(HTTPRequest)) == True))
        return plist

    # 分析函数，返回一个分析数据后得到的dataList        dataList为 包含所有数据列表的列表 type:list
    def analyzeMain(plist):
        # analyze tlsPkt, fetch the fields, and push them into the dataList
        tlsPList = plist.filter(lambda x: ((x.haslayer(TLS_Ext_ServerName) == True)))
        for i in tlsPList:
            tempTime = time.localtime(int(i.time))
            pktTime = time.strftime("%Y%m%d%H%M%S", tempTime)
            domain = i[TLS_Ext_ServerName].servernames[0].servername.decode()
            row=["TLS",pktTime,i[IP].src,i[IP].dst,domain]  # queue of the row MUST BE [pktType,capTime,srcIP,dstIP,`domain`]
            dataList.append(row)

        # analyze httpPkt, fetch the fields, and push them into the dataList
        httpPList = plist.filter(lambda x: ((x.haslayer(HTTPRequest) == True)))
        for i in httpPList:
            tempTime = time.localtime(int(i.time))
            pktTime = time.strftime("%Y%m%d%H%M%S", tempTime)
            domain = i[HTTPRequest].Host.decode()
            row = ["HTTP", pktTime, i[IP].src, i[IP].dst,domain]  # queue of the row MUST BE [pktType,capTime,srcIP,dstIP,`domain`]
            dataList.append(row)
    analyzeMain(sniffMain())
    return dataList

"""

# for testing

if __name__ == "__main__":
    dateList=getDataList()
    for i in dateList:
        print(i)
"""
```

<br>
<br>

main.py

```
import database
import sniffResults

if __name__ == "__main__":
    print("中断程序以获取结果，并将结果存往数据库")
    sqlObj=database.linkSql()
    dataList=sniffResults.getDataList()
    sqlObj.exec("SELECT MAX(`index`) FROM sniffTable")
    lastRecordIndex=sqlObj.cursors.fetchone()["MAX(`index`)"]   # store the lasest index in the table

    # insert the row of dataList in table
    commandInsert=sqlObj.assembleSqlText("sniffTable",dataList)
    #print(commandInsert)
    sqlObj.exec(commandInsert)

    # show the added results
    commandSearchNew="SELECT * FROM sniffTable WHERE `index`> " + str(lastRecordIndex)
    #print (commandSearchNew)
    sqlObj.exec(commandSearchNew)
    print("NEW RECORDS:")
    for i in sqlObj.cursors.fetchall():
        print (i["index"],"\t",i["pktType"],"\t",i["capTime"],"\t",i["srcIP"],"--->",i["dstIP"],"doamin:",i["domain"])
```

# 参考

[使用python抓包并分析后存入数据库，或直接分析tcpdump和wireshark抓到的包，并存入数据库 - 花柒博客-原创分享](https://www.vuar.cn/index.php/archives/117/)
