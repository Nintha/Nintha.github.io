---
title: 02-实现一个简单的爬虫
date: 2018-07-08 00:00:01
categories: Node爬虫指北
---
先检查一下我们需要准备的东西：

- Nodejs >= 8.9
- B站视频统计信息API：`https://api.bilibili.com/x/web-interface/archive/stat?aid=26186448`

如果你已经准备好了，在这一章我们会实现一个非常简单的爬虫。

## 思路

爬虫的简化流程是这样的：

发送一个HTTP请求到目标服务器，服务器返回相应的数据

```
+----------+                +-----------+
|          |  HTTP Request  |           |
|          +---------------->           |
|  Nodejs  |                |  Servers  |
|          <----------------+           |
|          |  HTTP Response |           |
+----------+                +-----------+

```

## Coding 

首先新建一个名为`node-simple-spider`的文件夹，“发送一个HTTP请求”这个功能需要`superagent`第三方库来实现，在文件夹`node-simple-spider`下打开你的终端窗口，键入以下命令来安装`superagent`的相关依赖：

```shell
npm i superagent --save
```

在文件夹里面创建一个名为`index.js`的文件，用你最喜欢的编辑器打开它，键入以下内容

```javascript
const superagent = require('superagent') // 引入superagent包
superagent
	.get('https://api.bilibili.com/x/web-interface/archive/stat') // Get请求url
	.query({ aid:26186448 }) // 请求参数，26186448是一个视频的ID
	.then(res => res && console.log(res.body)) // 请求成功的回调函数
	.catch(err => console.error(err)) // 请求失败的回调函数
```

然后在刚刚的终端窗口中键入以下命令运行这个程序:

```shell
node index.js
```

成功的话可以在终端看到以下输出

```shell
$ node index.js
{ code: 0,
  message: '0',
  ttl: 1,
  data:
   { aid: 26186448,
     view: 605255,
     danmaku: 11831,
     reply: 3476,
     favorite: 14744,
     coin: 40242,
     share: 1645,
     like: 20596,
     now_rank: 0,
     his_rank: 1,
     no_reprint: 1,
     copyright: 1 } }
```

这个Json就是我们所爬到的数据，里面包含了对应视频数据的播放数、弹幕数、回复、收藏等很多统计信息。

<!--more-->

#### 输出到文件

仅仅把爬取的内容打印到控制台显然是不够了，为了可以重复利用这些数据，我们可以把这些数据写入磁盘文件里面；代码如下:

```javascript
const superagent = require('superagent')
const fs = require('fs') // fs是node自带操作文件包
superagent
	.get('https://api.bilibili.com/x/web-interface/archive/stat')
	.query({ aid:26186448 })
	.then(res => {
		// 把返回数据写入data.json文件
		try{
			fs.writeFile('data.json', res.text, err => {
  				console.log('The data was appended to file!');
			});
		} catch(err) {
			console.error(err)
		}
	})
	.catch(err => console.error(err))

```

再次运行代码，就会发现在文件夹里多了个`data.json`文件，里面是我们爬到的数据。

注：`fs.writeFile`每次调用重新生成文件，同名的旧文件会被覆盖；如果希望保留旧文件里的数据可以使用`fs.appendFile `代替。

`superagent`更加详细的语法可以参考[这里](https://www.npmjs.com/package/superagent)



