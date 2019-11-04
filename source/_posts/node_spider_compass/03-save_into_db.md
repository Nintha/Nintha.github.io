---
title: 03-保存数据到数据库
date: 2018-07-08 00:00:02
categories: Node爬虫指北
---
很多情况下，为了方便使用，我们习惯把数据存储在数据库中，本章将在前一章的基础上，加入保存数据到MongoDB的逻辑。

在开始前需要确保已经成功安装了MongoDB（localhost:27017）和完成前面的章节内容

## Coding

为了方便Nodejs调用MongoDB，我们使用`mongojs`这个第三方包，它可以让我们使用近似原生的MongoDB语法来处理数据，详细语法可以查看[相关文档](https://www.npmjs.com/package/mongojs)。

安装`mongojs`：

```shell
npm i mongojs --save
```

我们再新建一个名为`dao.js`的文件，里面将存放关于和MongoDB相关的逻辑，并键入以下代码:

```js
const mongojs = require('mongojs') // 引入mongojs包
// 连接本地mongodb，关联名为'simple_spider'的库中名为'test'的集合(表)
const localdb = mongojs('simple_spider', ['test'])

//保存数据
const saveData = (data,cb) => {
	localdb.test.save(data, (err, res)=>{
	 	cb && cb(err, res)
	})
}

// 打印所有数据
const printAllData = (cb) => {
	localdb.test.find({},(err, docs)=>{
		console.log(docs)
		cb && cb(err, docs)
	})
}

// 关闭mongodb连接，在程序结束时调用，可以防止内存泄漏
const closeMongo = () => localdb.close()

// 将方法暴露出去
module.exports = {
	saveData,printAllData,closeMongo
}
```

`dao.js`提供了三个方法供`index.js`进行调用，同时我们需要修改`index.js`的代码：

```js
const superagent = require('superagent')
const fs = require('fs')
const dao = require('./dao') 

superagent
	.get('https://api.bilibili.com/x/web-interface/archive/stat')
	.query({ aid:26186448 })
	.then(res => {
		const data = res.body.data // 取出需要保存的数据
		dao.saveData(data, (err,res)=>{
			console.log('saved data')
			//打印数据库里保存的数据，然后关闭数据库连接
			dao.printAllData( () => dao.closeMongo() ) 
		})
	})
	.catch(err => console.error(err))
```

运行代码，顺利的话可以看到这样的输出：

```shell
$ node index
saved data
[ { _id: 5b41d634f3df89032c834d5b,
    aid: 26186448,
    view: 619751,
    danmaku: 12053,
    reply: 3500,
    favorite: 14961,
    coin: 40699,
    share: 1676,
    like: 20751,
    now_rank: 0,
    his_rank: 1,
    no_reprint: 1,
    copyright: 1 } ]

```







