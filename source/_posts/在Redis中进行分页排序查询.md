title: 在Redis中进行分页排序查询
date: 2015-11-17 15:35:01
tags: [数据库, NoSQL, Redis]
---
Redis是一个高效的内存数据库，它支持包括String、List、Set、SortedSet和Hash等数据类型的存储，在Redis中通常根据数据的key查询其value值，Redis没有条件查询，在面对一些需要分页或排序的场景时（如评论，时间线），Redis就不太好不处理了。
<!-- more -->
前段时间在项目中需要将每个主题下的用户的评论组装好写入Redis中，每个主题会有一个topicId，每一条评论会和topicId关联起来，得到大致的数据模型如下：

```
{
    topicId: 'xxxxxxxx',
    comments: [
        {
            username: 'niuniu',
            createDate: 1447747334791,
            content: '在Redis中分页',
            commentId: 'xxxxxxx',
            reply: [
                {
                    content: 'yyyyyy'
                    username: 'niuniu'
                },
                ...
            ]
        },
        ...
    ]
}
     
```

将评论数据从MySQL查询出来组装好存到Redis后，以后每次就可以从Redis获取组装好的评论数据，从上面的数据模型可以看出数据都是key-value型数据，无疑要采用hash进行存储，但是每次拿取评论数据时需要分页而且还要按createDate字段进行排序，hash肯定是不能做到分页和排序的。

那么，就挨个看一下Redis所支持的数据类型：
> **String: **主要用于存储字符串，显然不支持分页和排序。
> **Hash: **主要用于存储key-value型数据，评论模型中全是key-value型数据，所以在这里Hash无疑会用到。
> **List: **主要用于存储一个列表，列表中的每一个元素按元素的插入时的顺序进行保存，如果我们将评论模型按createDate排好序后再插入List中，似乎就能做到排序了，而且再利用List中的LRANGE key start stop指令还能做到分页。嗯，到这里List似乎满足了我们分页和排序的要求，但是评论还会被删除，就需要更新Redis中的数据，如果每次删除评论后都将Redis中的数据全部重新写入一次，显然不够优雅，效率也会大打折扣，如果能删除指定的数据无疑会更好，而List中涉及到删除数据的就只有LPOP和RPOP这两条指令，但LPOP和RPOP只能删除列表头和列表尾的数据，不能删除指定位置的数据，所以List也不太适合。
> **Set: **主要存储无序集合，无序！排除。
> **SortedSet: **主要存储有序集合，SortedSet的添加元素指令*ZADD key score member [[score,member]...]*会给每个添加的元素member绑定一个用于排序的值score，SortedSet就会根据score值的大小对元素进行排序，在这里就可以将createDate当作score用于排序，SortedSet中的指令*ZREVRANGE key start stop*又可以返回指定区间内的成员，可以用来做分页，SortedSet的指令ZREM key member可以根据key移除指定的成员，能满足删评论的要求，所以，SortedSet在这里是最适合的。

所以，我需要用到的数据类型有SortSet和Hash，SortSet用于做分页排序，Hash用于存储具体的键值对数据，我画出了如下的结构图：

![结构图](http://nnblog-storage.b0.upaiyun.com/img/redis.png!watermark1.0)

在上图的SortSet结构中将每个主题的topicId作为set的key，将与该主题关联的评论的createDate和commentId分别作为set的score和member，commentId的顺序就根据createDate的大小进行排列。
当需要查询某个主题某一页的评论时，就可主题的topicId通过指令*zrevrange topicId (page-1)×10 (page-1)×10+perPage*这样就能找出某个主题下某一页的按时间排好顺序的所有评论的commintId。page为查询第几页的页码，perPage为每页显示的条数。
当找到所有评论的commentId后，就可以把这些commentId作为key去Hash结构中去查询该条评论对应的内容。
这样就利用SortSet和Hash两种结构在Redis中达到了分页和排序的目的。
