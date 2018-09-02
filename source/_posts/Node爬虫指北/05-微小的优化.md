---
title: 05-微小的优化
date: 2018-07-10 00:00:00
categories: Node爬虫指北
---

本章会针对前面章节中不完善的地方进行补充说明。

## 批量写入

之前的章节中，我们使用MongoDB进行存储数据，每当爬取一条记录就往数据库里写，虽然这样做也能实现我们所需要的功能，但在性能上还是会有或多或少的影响。聪明的同学肯定已经想到了，如果一条一条的写入数据库比较费力的话，那就爬取一批数据后再写入就好啦。

原来的代码：

```js
// ...other code

localdb.test.save(data, (err, res)=>{
	// do something
})
```

`save`方法的第一个参数不仅可以接受一个对象，也可以接受一个数组，类似这样：

```js
const array = []
for(let i = 0 ; i < MAX_ID; i++){
	// fetch data from website
	const data =  fetchData(i)
    array.push(data)
}

localdb.test.save(array, (err, res)=>{
	// do something
})
```

其实语法上没有很大的区别，不过批量写入数据操作可以帮你的爬虫程序更加高效。



## 同步与异步

一般情况下，循环爬虫逻辑可以分为同步写法和异步写法。

同步写法如下所示:

```js
const main = async (MAX_ID) => {
    const array = []
    for(let i = 0 ; i < MAX_ID; i++){
        // fetch data from website
        const data = await fetchData(i) //this return a Promise
        array.push(data)
    }
    saveToMongoDB(array);
}
main(1000);
```

代码使用 `async/await`进行控制逻辑流，让代码可以同步运行，代码逻辑非常清晰，也很方便debug，缺点是效率偏低，因为如果其中一次循环由于某种原因阻塞了，会影响后面逻辑的执行。

<!--more-->

异步代码如下所示：

```js
const main = async (MAX_ID) => {
    const array = []
    for(let i = 0 ; i < MAX_ID; i++){
        // fetch data from website
        fetchData(i) // it return a Promise
        	.then(data => array.push(data))
        await sleep(100) // block thread 100 ms    
    }
    // wait for async threads
    while(array.length < MAX_ID){
        await sleep(100)
    } 
    saveToMongoDB(array);
}
main(1000);
```



异步的逻辑更加的复杂，由于逻辑流的难以控制，需要更多的代码去保证每一个请求可以正确地获取到数据，还需要进行超时判断；但复杂的逻辑所换来的好处是不言而喻的，并发的请求可以让爬虫速度更进一步。



## 断点续爬

很多时候我们写了一个看起来很棒的爬虫程序，启动后可以飞快地爬取，但运行过程中由于某些不可控的因素需要中断运行（比如，万能的Win10在关键时刻需要重启一下），如果贸然地中断程序，下次想要继续爬的话又需要自己进行繁琐的数据清理，特别是大量地使用了异步的逻辑（数据大概率不是连续的）。所以在编写爬虫程序的时候，预先为断点续爬进行合理的规划是非常有必要的。

断点续爬的实现有很多种，这里提供一种比较简单的实现思路：既然要断点，那么程序随时需要保存当前的爬取进度。

最极端的情况下，是使用同步单线程的方式进行爬虫，爬取完一条数据后再爬取下一条数据，如果按ID升序爬取的话，我们只要再续爬的时候查询下数据库里最大的ID就OK了。这样做的话，我们不需要额外的逻辑记录当前进度，但是爬虫的效率就很低下，实际使用中很少会使用这种方式。

聪明的同学肯定已经想到了，只要把需要爬的数据分成很多个组，每次完成一组的内容再爬取下一组的内容，每完成一组把整组的数据一次性写入数据库，这样可以保证数据的连续性。这样的话，即使中间意外中断，最多损失一组的数据，对于整个爬虫也不会有很大的影响。

下面是参考代码

```js
// 获取waiting状态任务
const findOneWaitingPackage = async () => {
    return new Promise((resolve, reject) =>
        localdb.package
            .find({ status: { $lte: 0 } })
            .sort({
                status: -1,
                pid: 1
            })
            .limit(1, (err, doc) => {
                if (err) reject(err)
                else {
                    updatePackageToRunning(doc[0])
                    resolve(doc[0])
                }
            })
    )
}
/**
 * 重置package表
 * 创建两个索引status, pid
 */
const resetPackage = async () => {
    indexMember().catch(err => console.error(err))
    return new Promise((resolve, reject) =>
        localdb.package.drop(() => {
            localdb.package.ensureIndex({ status: 1 }, () => {
                localdb.package.ensureIndex(
                    { pid: 1 },
                    { unique: true },
                    (err, res) => (err ? reject(err) : resolve(res))
                )
            })
        })
    )
}
/**
 * 创建任务包
 * package status: 0-waiting, 1-finished，-1-running
 */
const insertPackage = async (start, end) => {
    const packs = Array(end - start + 1)
        .fill(0)
        .map((v, i) => {
            return {
                pid: i + start,
                status: 0
            }
        })
    return new Promise((resolve, reject) =>
        localdb.package.insert(
            packs,
            (err, res) => (err ? reject(err) : resolve(res && res.length))
        )
    )
}
// 更新package到执行中的状态
const updatePackageToRunning = async pid => {
    if (!pid) return
    return new Promise((resolve, reject) =>
        localdb.package.update(
            { pid },
            { $set: { status: -1 } },
            { multi: false },
            (err, doc) => (err ? reject(err) : resolve(doc))
        )
    )
}
// 更新package到完成状态
const updatePackageToFinished = async pid => {
    return new Promise((resolve, reject) =>
        localdb.package.update(
            { pid },
            { $set: { status: 1 } },
            { multi: false },
            (err, doc) => (err ? reject(err) : resolve(doc))
        )
    )
}

```

这里我们专门创建了一张package表进行任务包的分配，每个任务包有三种状态：

- 0 ：准备
- 1 ：完成
- -1：执行中

注：每个任务包包括的任务内容需要读者自行处理

`resetPackage`和`insertPackage`方法是爬虫程序第一次启动的时候需要运行的，它们会构建一张完整的package表，包括索引的创建（有效地保证程序的效率）。

`findOneWaitingPackage`可以获取到一个准备状态的任务包，当这个任务包的内容被全部完成的时候，可以调用`updatePackageToFinished`方法把任务包标记成完成状态。

当程序完成所有任务包的时候，`findOneWaitingPackage`会返回`null`。



上面提到的内容更多的时候需要根据实际的场景来实施，而且这些优化往往和语言无关，更多还是经验的积累。

