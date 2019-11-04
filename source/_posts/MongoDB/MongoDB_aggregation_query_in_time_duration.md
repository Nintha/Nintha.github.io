---
title: MongoDB基于时间段的聚合查询
date: 2018-05-15
---

## 前言

最近写了一个爬虫对B站的视频统计数据进行追踪，每2分钟爬取一次存在mongo里，然后用这些数据画折线图。这个时候问题来了，如果我爬取了一年的数据，进行数据展示的时候，不应该把一年的数据都从数据库里读取出来，对于年这样大粒度统计，应该以每天抽取一条记录就OK了；当想看一天内的数据变化，又要以分钟为粒度进行记录抽取。

基本的mongo查询语法已经难以解决均匀抽样查询记录了，这时就需要聚合查询这样的工具。

<!--more-->

## Mongo聚合查询

先看下存储的数据结构，其中`ctime`代表了这条记录的创建时间。

```json
{
    "_id" : ObjectId("5ae738594cb3ed1a60210042"),
    "aid" : NumberLong(22755224),
    "view" : NumberLong(18649),
    "danmaku" : NumberLong(533),
    "favorite" : NumberLong(1027),
    "reply" : NumberLong(553),
    "coin" : NumberLong(1896),
    "share" : NumberLong(172),
    "ctime" : ISODate("2018-04-30T15:38:01.120Z")
}
```

查询的目标是这样的，对于指定的时间区间，返回最多300条（这个数量大概是性能和图表显示效果的平衡吧）。

思路是这样的：

​先对把时间区间平均分成300份，对每一个份的小时间区间中按一个统一的规律取一条记录作为小时间区间的代表记录（这个统一的规律可以用最大值、最小值、平均值等，我这里使用了最大值，为了保证可以看到最新的一条追踪记录）。

时间区间均分这个需求可以用mongo聚合中的`$group`操作进行处理，不过值得注意的是`$group`是对某一列中拥有相同值的记录进行分组。那如何把一段时间内的记录归并到同一组呢？一种可行的操作是把时间转换成时间戳，就是长整形数字，把时间区间同样转换为一个数字，两者相除再抹除小数部分就可以了。

把这个思路用Mongo原生API实现：

```javascript
db.trace_video_stat.aggregate([
    {"$match":{
        "aid":22755224,
        "ctime": {
            '$gte':ISODate("2018-05-10T00:00:00Z"),
            '$lt':ISODate("2018-05-10T00:30:00Z")
        }
    }},
    {"$group": {
        "_id": { 
            "$subtract": [
                { "$subtract": [ "$ctime", new Date("1970-01-01") ] },
                { "$mod": [
                    { "$subtract": [ "$ctime", new Date("1970-01-01") ] },
                    1000 * 60 * 30  /*聚合时间段，30分钟*/
                ]}
            ]
        },
        "coin": {'$max': '$coin'},
        "view": {'$max': '$view'},
        "danmaku": {'$max': '$danmaku'},
        "favorite": {'$max': '$favorite'},
        "reply": {'$max': '$reply'},
        "coin": {'$max': '$coin'},
        "share": {'$max': '$share'},
        "ctime": {'$max': '$ctime'},
  	}},
  	{"$sort": {
        'ctime': 1
  	}}
 ])

```

这里先用`$match`去除对应时间区间的数据，再用`$group`进行分组处理，最后用`$sort`进行排序。

查询结果：

```json
[{
    "_id" : NumberLong(1525910400000),
    "coin" : NumberLong(8421),
    "view" : NumberLong(109018),
    "danmaku" : NumberLong(2859),
    "favorite" : NumberLong(4939),
    "reply" : NumberLong(1799),
    "share" : NumberLong(837),
    "ctime" : ISODate("2018-05-10T00:00:22.563Z")
},
// omitted
]
```

这里出现的字段是和`$group`中处理的字段一一对应，一些没有用到的字段就会被省略。注意这里的`_id`并不是mongo的ObjectId。

## SpringData mongoTemplate

实际开发中我们可能会用到SpringData封装mongoTemplate来操作数据库。同样的，使用mongoTemplate也可以很方便的实现一样的效果。

```
public List<VideoStat> getVideoStat(long aid, Date startTime, Date endTime) {
    long interval = (endTime.getTime() - startTime.getTime()) / 300;
    TypedAggregation<VideoStat> agg = Aggregation.newAggregation(
                VideoStat.class,
                match(Criteria.where("aid").is(aid).and("ctime").gte(startTime).lte(endTime)),
                project("ctime", "coin", "share", "view", "danmaku", "favorite", "reply")
                        .andExpression("ceil((ctime - [0]) / [1])", new Date(0), interval)
                        .as("cdate"),
                group("cdate")
                        .max("view").as("view")
                        .max("coin").as("coin")
                        .max("danmaku").as("danmakuu")
                        .max("favorite").as("favorite")
                        .max("reply").as("reply")
                        .max("share").as("share")
                        .max("ctime").as("ctime"),
                sort(Sort.Direction.ASC, "ctime"));
    return mongoTemplate.aggregate(agg, "TRACE_VIDEO_STAT", VideoStat.class).getMappedResults();
        
}
```

match方法缩小数据范围，project方法决定需要聚合的字段，group方法决定被分组的列表，后面的链时调用时对组内数据的统计处理，最后的sort是排序。

这里用到的SpEL表达式`.andExpression("ceil((ctime - [0]) / [1])", new Date(0), interval)`，`[0]`是占位符，表达式支持对时间类型的直接运算，默认会转换为时间戳进行处理，还是非常方便的。