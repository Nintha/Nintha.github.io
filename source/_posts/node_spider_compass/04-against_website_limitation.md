---
title: 04-应对网站的限制
date: 2018-07-08 00:00:04
categories: Node爬虫指北
---
在完成前面章节后，的确可以去爬取很多数据了，但很多网站会对爬虫了流量进行限制，我们该如何去应对这些限制呢？

## 请求频率

对于某些API对请求频率有一定的限制，我们可以手动控制请求的时间间隔，比如使用`setInterval`函数可以进行定时执行代码，示例代码：

```
// ...require something

const fetch = (aid) => superagent
		.get('https://api.bilibili.com/x/web-interface/archive/stat')
		.query({ aid:aid })
		.then(res => {
			const data = res.body.data // 取出需要保存的数据
			dao.saveData(data, (err,res)=>{
				console.log('saved data, aid=' + aid)
			})
		})
		.catch(err => console.error(err))
// 每隔2000 ms执行一次fetch函数, 期间aid会自增，达到定时执行的目的
let aid = 26186448
setInterval(() => fetch(aid++), 2000)
```

`setInterval`详细语法可以参考[这里](http://www.runoob.com/jsref/met-win-setinterval.html)

如果你对Nodejs比较熟悉的话，也可以尝试去使用ES8里的新语法`async/await`，下面代码是一个简单的demo

```js
const sleep = (ms) => new Promise((suc,fail) => setTimeout(suc, ms));
(async ()=>{
	for(;;){
		await sleep(2000)
		console.log(new Date())
	}
})()
```

输出如下:

```
$ node index
2018-07-08T10:09:43.830Z
2018-07-08T10:09:45.833Z
2018-07-08T10:09:47.835Z
2018-07-08T10:09:49.838Z
2018-07-08T10:09:51.839Z
```

其中`async`关键字放在函数前面声明这时一个异步函数，`await`关键字会对`Promise`进行阻塞，从而达到定时执行的效果; `sleep`是我们自行封装的休眠函数，用`Promise`对setTimeout进行封装，方便调用。



## UA验证

UA指的是HTTP请求头部的`User-Agent`字段，是用来给服务器分辨请求来源浏览器型号用到的。很多服务器会对未知UA的请求进行限制，这个时候我们需要对UA进行伪装。

<!--more-->

幸运的是，`superagent`提供了很方便的API在请求中设置UA，如下所示:

```js
 superagent
		.get('https://api.bilibili.com/x/web-interface/archive/stat')
		.set({ 'User-Agent': 'Opera/9.80 balabala...' }) //设置User-Agent
		.query({ aid:aid })
		.then(res => {
			// do something
		})
		.catch(err => console.error(err))
```

为了使伪装的效果更好，我们通常会不停的变换UA，以下是一个可用的UA列表:

```js
const USER_AGENTS = [
    'Mozilla/5.0 (X11; U; Linux i686; en-US; rv:1.8.0.12) Gecko/20070731 Ubuntu/dapper-security Firefox/1.5.0.12',
    'Mozilla/4.0 (compatible; MSIE 7.0; Windows NT 6.0; Acoo Browser; SLCC1; .NET CLR 2.0.50727; Media Center PC 5.0; .NET CLR 3.0.04506)',
    'Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/535.11 (KHTML, like Gecko) Chrome/17.0.963.56 Safari/535.11',
    'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_7_3) AppleWebKit/535.20 (KHTML, like Gecko) Chrome/19.0.1036.7 Safari/535.20',
    'Mozilla/5.0 (X11; U; Linux i686; en-US; rv:1.9.0.8) Gecko Fedora/1.9.0.8-1.fc10 Kazehakase/0.5.6',
    'Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/537.1 (KHTML, like Gecko) Chrome/21.0.1180.71 Safari/537.1 LBBROWSER',
    'Mozilla/5.0 (compatible; MSIE 9.0; Windows NT 6.1; Win64; x64; Trident/5.0; .NET CLR 3.5.30729; .NET CLR 3.0.30729; .NET CLR 2.0.50727; Media Center PC 6.0) ,Lynx/2.8.5rel.1 libwww-FM/2.14 SSL-MM/1.4.1 GNUTLS/1.2.9',
    'Mozilla/4.0 (compatible; MSIE 6.0; Windows NT 5.1; SV1; .NET CLR 1.1.4322; .NET CLR 2.0.50727)',
    'Mozilla/5.0 (compatible; MSIE 9.0; Windows NT 6.1; WOW64; Trident/5.0; SLCC2; .NET CLR 2.0.50727; .NET CLR 3.5.30729; .NET CLR 3.0.30729; Media Center PC 6.0; .NET4.0C; .NET4.0E; QQBrowser/7.0.3698.400)',
    'Mozilla/4.0 (compatible; MSIE 6.0; Windows NT 5.1; SV1; QQDownload 732; .NET4.0C; .NET4.0E)',
    'Mozilla/5.0 (Windows NT 6.1; Win64; x64; rv:2.0b13pre) Gecko/20110307 Firefox/4.0b13pre',
    'Opera/9.80 (Macintosh; Intel Mac OS X 10.6.8; U; fr) Presto/2.9.168 Version/11.52',
    'Mozilla/5.0 (X11; U; Linux i686; en-US; rv:1.8.0.12) Gecko/20070731 Ubuntu/dapper-security Firefox/1.5.0.12',
    'Mozilla/5.0 (compatible; MSIE 9.0; Windows NT 6.1; WOW64; Trident/5.0; SLCC2; .NET CLR 2.0.50727; .NET CLR 3.5.30729; .NET CLR 3.0.30729; Media Center PC 6.0; .NET4.0C; .NET4.0E; LBBROWSER)',
    'Mozilla/5.0 (X11; U; Linux i686; en-US; rv:1.9.0.8) Gecko Fedora/1.9.0.8-1.fc10 Kazehakase/0.5.6',
    'Mozilla/5.0 (X11; U; Linux; en-US) AppleWebKit/527+ (KHTML, like Gecko, Safari/419.3) Arora/0.6',
    'Mozilla/5.0 (compatible; MSIE 9.0; Windows NT 6.1; WOW64; Trident/5.0; SLCC2; .NET CLR 2.0.50727; .NET CLR 3.5.30729; .NET CLR 3.0.30729; Media Center PC 6.0; .NET4.0C; .NET4.0E; QQBrowser/7.0.3698.400)',
    'Opera/9.25 (Windows NT 5.1; U; en), Lynx/2.8.5rel.1 libwww-FM/2.14 SSL-MM/1.4.1 GNUTLS/1.2.9',
    'Mozilla/5.0 (Windows NT 10.0; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/61.0.3163.100 Safari/537.36'
];

```



