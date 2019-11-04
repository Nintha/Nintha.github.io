---
title: KMS服务器搭建并激活Office
date: 2018-04-05
---
## 前言

本教程提供一种KMS激活方案，即在本地搭建临时性KMS服务器，通过相应命令进行激活。本教程使用Office Visio 2016为例子，Windows 10和其他Office 2016组件激活同理。
<!--more-->
## 搭建KMS服务器

这里我们是使用vlmcsd搭建KMS服务器，首先下载相应压缩包，解压后进入vlmcsd\binaries文件夹里面是各个平台对应的服务端执行文件。

[下载地址](https://github.com/Nintha/gitTest/blob/master/kms%E6%9C%8D%E5%8A%A1%E5%99%A8-vlmcsd.zip)

`vlmcsd`开头的是我们所需要的文件，如win64平台就运行`vlmcsd-Windows-x64.exe`

运行后,会有一个弹出一个cmd界面，现在KMS服务器已经启动了。KMS服务器默认是使用1688端口，如果你想要把它放在公网上的话，务必开启1688端口的权限。

## 激活Visio

使用[管理员权限]进入cmd或powershell（Win+X, A）

```shell
# 64位office安装目录
cd "C:\Program Files (x86)\Microsoft Office\Office16"
# 设置kms地址，本地服务器就用127.0.0.1，端口默认是1688
cscript ospp.vbs /sethst:127.0.0.1
# 安装visio 专业版密钥，如果要激活其他组件，安装对应密钥即可
cscript ospp.vbs /inpkey:PD3PC-RHNGV-FXJ29-8JK7D-RJRJK
# 激活
cscript ospp.vbs /act
```

然后重启电脑，可以看到Visio 2016已经激活了。

## 其他情况的处理

如果出现激活失败的情况，有可能是你的系统里已经安装了不止一个密钥，导致激活失败。这个时候就需要把失败的密钥卸载，重新安装本KMS支持的密钥。
```shell
# 查看已经安装的密钥（如果安装多个密钥，激活会失败，需要进行卸载）
cscript ospp.vbs /dstatus
# 卸载密钥，XXXXX是密钥后5位，上条命令中可以获取
cscript ospp.vbs /unpkey:XXXXX
# 查看帮助
cscript ospp.vbs /?
```

## 附录
Windows10和Office2016官方KMS Client key:
```
Office Professional Plus 2016 - XQNVK-8JYDB-WJ9W3-YJ8YR-WFG99
Office Standard 2016 - JNRGM-WHDWX-FJJG3-K47QV-DRTFM
Project Professional 2016 - YG9NW-3K39V-2T3HJ-93F3Q-G83KT
Project Standard 2016 - GNFHQ-F6YQM-KQDGJ-327XX-KQBVC
Visio Professional 2016 - PD3PC-RHNGV-FXJ29-8JK7D-RJRJK
Visio Standard 2016 - 7WHWN-4T7MP-G96JF-G33KR-W8GF4
Access 2016 - GNH9Y-D2J4T-FJHGG-QRVH7-QPFDW
Excel 2016 - 9C2PK-NWTVB-JMPW8-BFT28-7FTBF
OneNote 2016 - DR92N-9HTF2-97XKM-XW2WJ-XW3J6
Outlook 2016 - R69KK-NTPKF-7M3Q4-QYBHW-6MT9B
PowerPoint 2016 - J7MQP-HNJ4Y-WJ7YM-PFYGF-BY6C6
Publisher 2016 - F47MM-N3XJP-TQXJ9-BP99D-8K837
Skype for Business 2016 - 869NQ-FJ69K-466HW-QYCP2-DDBV6
Word 2016 - WXY84-JN2Q9-RBCCQ-3Q3J3-3PFJ6
Windows 10 Home - TX9XD-98N7V-6WMQ6-BX7FG-H8Q99
Windows 10 Home N - 3KHY7-WNT83-DGQKR-F7HPR-844BM
Windows 10 Home Single Language - 7HNRX-D7KGG-3K4RQ-4WPJ4-YTDFH
Windows 10 Home Country Specific - PVMJN-6DFY6-9CCP6-7BKTT-D3WVR
Windows 10 Professional - W269N-WFGWX-YVC9B-4J6C9-T83GX
Windows 10 Professional N - MH37W-N47XK-V7XM9-C7227-GCQG9
Windows 10 Education - NW6C2-QMPVW-D7KKK-3GKT6-VCFB2
Windows 10 Education N - 2WH4N-8QGBV-H22JP-CT43Q-MDWWJ
Windows 10 Enterprise - NPPR9-FWDCX-D2C8J-H872K-2YT43
Windows 10 Enterprise N - DPH2V-TTNVB-4X9Q3-TJR4H-KHJW4
Windows 10 Enterprise 2015 LTSB - WNMTR-4C88C-JK8YV-HQ7T2-76DF9
Windows 10 Enterprise 2015 LTSB N - 2F77B-TNFGY-69QQF-B8YKP-D69TJ
```
激活windows的命令为：
```shell
slmgr /skms franklv.ddns.net
slmgr /ato
```