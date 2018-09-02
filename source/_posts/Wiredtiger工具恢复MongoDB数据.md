---
title: Wiredtiger工具恢复MongoDB数据
date: 2018-05-06
---
## 前言

前几日在VPS折腾MongoDB，由于机器内存实在是太小了（1G，又跑了别的程序），进行重建索引操作时，内存不足被系统给kill了。强制kill的Mongo无法在`repair`模式下恢复，只能直接用Wiredtiger工具读取二进制数据文件进行恢复了。

<!--more-->
## 准备
官方文档对于这种情况并没有给予更多的提示了，这个时候只能拿出强大的Google了（百度已经拯救不了）。
经过一番资料查找，得出以下结论：
- 在`repair`模式失败的情况下，恢复MongoDB是比较困难了
- MongoDB的数据使用WiredTiger引擎进行存储，可以直接使用相关工具读取二进制文件数据
- 网上有一篇成功恢复数据的文章可以参考 [Link](http://www.alexbevi.com/blog/2016/02/10/recovering-a-wiredtiger-collection-from-a-corrupt-mongodb-installation/)

按照资料，首先要找到你需要恢复的Collection所对应的`wt`文件，它们一般在Mongo安装目录的`data`文件夹下面，文件的格式应该类似这样
```
collection-0--282010455938071573.wt
```
然后把数据文件和`data`下面其他一些存有必要信息的文件一起复制到一个新的目录`mongo-bak`（为啥不在原来目录操作？又失败了怎么办），如下：
```
collection-0--282010455938071573.wt
_mdb_catalog.wt
sizeStorer.wt
storage.bson
WiredTiger
WiredTiger.basecfg
WiredTiger.lock
WiredTiger.turtle
WiredTiger.wt
```

数据准备就绪，我们需要一个工具 wt utility，
```
wget http://source.wiredtiger.com/releases/wiredtiger-3.0.0.tar.bz2
tar xvf wiredtiger-3.0.0.tar.bz2
cd wiredtiger-3.0.0
sudo yum install snappy-devel -y
./configure --enable-snappy
make
```
注意：我的系统是CentOS，如果你是Ubuntu的话，要把上面命令中的
`sudo yum install snappy-devel -y`
换成
`sudo apt-get install libsnappy-dev build-essential`

snappy是必须要安装的插件，不然无法正常解码二进制数据问题（就算你使用zlib模式进行压缩，还是需要它）

## 打捞数据

装好工具后，执行
```
./wt -v -h ../mongo-bak -C "extensions=[./ext/compressors/snappy/.libs/libwiredtiger_snappy.so]" -R salvage collection-0--282010455938071573.wt
```
其中`../mongo-bak`是我们刚刚放恢复文件的目录，`collection-0--282010455938071573.wt`就是需要恢复的wt文件

顺利的话，可以看到这样的信息
```
WT_SESSION.salvage 77982300
```
里面的数字应该是可以恢复的记录，有时候你会发现这个数字比预想的要大，原因是mongo把删除操作也会缓存在内存中，意外关闭时来不及写入磁盘，导致数据量增多。

另外，刚刚那个操作会重写wt文件里面的内容，所以尽量不要用源文件进行操作。修复好的wt文件并不能被mongo直接使用，我们需要使用wt工具里面的dump命令把里面的数据进行导出， 输入以下命令：
```
./wt -v -h ../mongo-bak -C "extensions=[./ext/compressors/snappy/.libs/libwiredtiger_snappy.so]" -R dump -f ../collection.dump collection-0--282010455938071573
```
这个命令会跑比较长的时间，可以用另一个命令行窗口查看`collection.dump`文件的生成情况，值得注意的事，根据集合内容的不同，dump文件的大小也有比较大的差异，我的一个3G大小zlib模式的wt文件，输出的dump有27G大小，一定要准备充足的磁盘空间，另外内存大概也会占用和wt文件近似的大小。

## 重建数据
原来的Mongo已经不可用了，我们需要一个新的Mongo来承载恢复出来的数据，首先在新的Mongo中建立一个collection
```
use anydb
db.back.insert({test: 1})
db.back.remove({})
db.back.stats()
```
这里在`anydb`数据库下建立了`back`集合，stats命令为了查看这个集合对应的wt文件,在输出的信息里，能找到这样一行
```
"uri" : "statistics:table:collection-7895--1435676552983097781" 
```
其中的`collection-7895--1435676552983097781`就是wt文件的名字了

先把新的mongo停了，我们需要直接对数据文件进行操作
```
./wt -v -h ../data -C "extensions=[./ext/compressors/snappy/.libs/libwiredtiger_snappy.so]" -R load -f ../collection.dump -r collection-7895--1435676552983097781
```
这里的`../data`是新mongo的存放数据文件的地址，wt工具会把dump直接写入对应的集合文件里，完成后会看到
```
table:collection-7895--1435676552983097781: 77982300
```
好了，我们再次启动mongo
```
root> show dbs
anydb → 3.087GB
local → 0.000GB
root> use anydb
switched to db anydb
anydb> show collections
back → 0.000GB / 3.087GB
anydb> db.back.count()
0
```
诡异的事情出现了，数据库大小看起来是没有问题的，但查不到记录总数
```
anydb> db.back.find({}, {_id: 1})
{
  "_id": ObjectId("55e07f3b2e967329c888ac74")
}
{
  "_id": ObjectId("55e07f3b2e967329c888ac76")
}
...
{
  "_id": ObjectId("55e07f402e967329c888ac85")
}
Fetched 20 record(s) in 29ms -- More[true]
```
可见数据已经写入了，但数据库的状态还是有点问题的,我们直接执行
```
db.repairDatabase()
```
进行修复,然后就可以正常查看记录总数了
```
anydb> db.back.count()
7789230
```
太好了，现在数据恢复了，但里面的内容肯定有不同程度的缺失，需要进一步的处理。

## 其他
根据网上的资料，本文中的操作仅对 >=3.2版本的mongo有效。

正确mongo流程可以按照官网提示，使用 `--repair` 命令进入修复模式，mongo会对所有数据进行一次扫描，并且重建索引，等到修复完毕就可以正常启动了。
但这个操作是有一个前提的，那就是文件数据必须要完整。问题是我的Mongo是被强制kill的，文件数据难免已经混乱，所以每次修复程序在重建索引这一步都会报错。

[本文谢绝转载]