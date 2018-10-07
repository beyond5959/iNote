title: SQL成长记
date: 2015-11-19 10:45:40
tags: [数据库, SQL]
---
这一段时间在进行服务器端的开发，在开发的过程需要遍写大量的SQL语句来操作数据库，对于之前SQL还是菜鸟水平的我来说在开发的过程中遇到很多之前不太熟悉的地方，有些业务需要的SQL较为复杂，需要连接好几张表，数据呈现的要求也多种多样，最后也还算顺利的完成了业务。
<!-- more -->
下面是一些在这个过程中让我对SQL有更深入认识的SQL语句：

## 模糊查询并按匹配度进行排序

```
    select name from user where name like '%foo%' order by abs(length(name)-length('foo'));
```

这是一种按匹配度排序很简陋的做法，在很多情况下都存在问题，如果是对查询的结果按匹配度排序有着重度的需求，最好不要使用这个。

## 分组统计数量

```
    select count(1) 'count',class from user where gender='male' group by class;
```

上面的SQL语句是在user表中查询每个班男生的数量，通过group by class做到了按班分组，通过count(1)做到了统计数量。

## 随机查询指定数量的数据

```
    select id,name from user where type=5 order by rand() limit 10;
```

在这里用到了SQL的内置函数rand()用于获取随机数据，不过处于性能考虑，在SQL语句中应尽量少用内置函数。

## 在查询字段中指定一个常量

```
    select k.* from (select id,create_date,content,'homework' as 'type' from comments where isHomework=1 union select id,create_date,content,'post' as 'type' from posts) k where k.account_id='123456' order by k.create_date desc;
```

上面的SQL语句在查询comments表和posts表是分别在查询字段中增加了常量homework和post，并指定了列名为type，也就是在临时表k中，存在名为type的列，它下面的值要么是homework，要么是post。

