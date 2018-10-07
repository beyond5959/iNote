title: MongoDB中的mapReduce
date: 2015-12-21 15:46:09
tags: [数据库, NoSQL, MongoDB]
---
MongoDB中的mapReduce一般用于处理大数据集。
<!-- more -->
## 基本用法
以下是基本的mapReduce命令的语法：
```
>db.collection.mapReduce(
   function() {emit(key,value);},  //map function
   function(key,values) {return reduceFunction},   //reduce function
   {
      out: collection,
      query: document,
      sort: document,
      limit: number
   }
)
```
使用mapReduce要实现连个函数Map函数和Reduce函数。
Map函数调用emit(key, value)，遍历collection中所有的记录，表示集合会按照指定的key进行映射分组，类似group by，分组的结果为value。
Reduce函数将对Map函数传递的key与value进行处理。
参数说明：
- **map:** 映射函数（生成键值对序列，作为reduce函数参数）。
- **reduce:** 统计函数，reduce函数的任务就是将key-values编程key-value，也就是把values数组变成一个单一的值value。
- **out:** 统计结果存放集合，不指定则使用临时集合，在客户端断开后自动删除。
- **query:** 一个筛选条件，只有满足条件的文档才会调用Map函数。（query，limit，sort可随意组合）
- **sort:** 排序参数，也就是在发往Map函数前给集合排序，可以优化分组机制。
- **limit:** 发往Map函数的文档数量的上限，要是没有limit，单独使用sort的用处不大。

## 演示
插入数据：
```
> db.books.find()
{ "_id" : ObjectId("533ee1e8634249165a819cd0"), "name" : "apue", "pagenum" : 1023 }
{ "_id" : ObjectId("533ee273634249165a819cd1"), "name" : "clrs", "pagenum" : 2000 }
{ "_id" : ObjectId("533ee2ab634249165a819cd2"), "name" : "python book", "pagenum" : 600 }
{ "_id" : ObjectId("533ee2b7634249165a819cd3"), "name" : "golang book", "pagenum" : 400 }
{ "_id" : ObjectId("533ee2ca634249165a819cd4"), "name" : "linux book", "pagenum" : 1500 }
```

Map函数：
```
var map = function(){
    var category;
    if ( this.pageNum >= 1000 ){
        category = 'Big Books';
    }else{
        category = 'Small Books';
    }
    emit(category, {name: this.name});
}
```

Map函数里面会调用emit(key, value)，集合会按照指定的key进行映射分组, 类似group by。this指向每一条被迭代的数据。

上面执行Map函数后的结果为: (按照category分组, 分组结果是{name: this.name}的list)
```
{"big books",[{name: "apue"}, {name : "linux book"}, {name : "clrs"}]]);
{"small books",[{name: "python book"}, {name : "golang book"}]);
```

Reduce函数：
```
var reduce = function(key, values) {
    var sum = 0;
    values.forEach(function(doc) {
    sum += 1;
    });
    return {books: sum};
};
```
reduce函数会对Map分组后的数据进行分组简化，注意：在reduce(key,value)中的key就是emit中的key，value为emit分组后的emit(value)的集合, 这里是{name: this.name}的list。

mapReduce函数：
```
> var count  = db.books.mapReduce(map, reduce, {out: "book_results"});
> db[count.result].find()
{ "_id" : "big books", "value" : { "books" : 3 } }
{ "_id" : "small books", "value" : { "books" : 2 } }
```

```
> db.books.mapReduce(map, reduce, {out: "book_results"});
{
        "result" : "book_results",
        "timeMillis" : 107,
        "counts" : {
                "input" : 5,
                "emit" : 5,
                "reduce" : 2,
                "output" : 2
        },
        "ok" : 1,
}
```

具体参数说明：
- **result:** 存储结果的collection的名字，这是个临时集合，mapReduce的连接关闭后制动就被删除了。
- **timeMillis:** 执行花费的时间，毫秒为单位。
- **input:** 满足条件被发送到Map函数的个数。
- **emit:** 在Map函数中emit被调用的次数，也就是所有集合中的数据总量。
- **output:** 结果集合中的个数。

查询结果集中的数据：
```
> db.book_results.find()
{ "_id" : "big books", "value" : { "books" : 3 } }
{ "_id" : "small books", "value" : { "books" : 2 } }
```
