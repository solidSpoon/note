---
slug: exploring-mongodb
title: MongoDB 初探
authors: [solidSpoon]
tags: [MongoDB]
---

# MongoDB 初探

对于已经熟悉 MySQL 的同学来说，初次接触 MongoDB 可能会不习惯它的语法，本篇文章将通过一个简单的示例带你入门 MongoDB。

## 准备

对于 MongoDB 新手，可以借助 DataGrip 来学习MongoDB 语法。在 MongoDB 中实现准备好两个表 "old" 和 "new"，并随意插入一些数据

```js
db.createCollection("new")  
db.createCollection("old")

db.old.insertOne({  
id: 1,  
name: old  
})  
db.new.insertOne({  
id: 2,  
name: 13,  
goods: 1  
})
// 随意插入数据
```

在 DataGrip 中输入如下的 SQL 语句

```sql
select * from "new" as aleft left join "old" as bright on aleft.id = bright.id;
```

然后在这条语句上「右键」，选择 「Show JS Script」，会发现 DataGrip 会帮助我们将 SQL 语句转为 MongoDB 语句，接下来我们通过研究这个语句来体会 MongoDB 的基本思想

```js
db.getSiblingDB("test").getCollection("new").aggregate([  
  {  
    $project: {"aleft": "$$ROOT", "_id": 0}  
  },  
  {  
    $lookup: {  
      localField: "aleft.id",  
      from: "old",  
      foreignField: "id",  
      as: "bright"  
    }  
  },  
  {  
    $unwind: {  
      path: "$bright",  
      preserveNullAndEmptyArrays: true  
    }  
  },  
  {  
    $replaceRoot: {  
      newRoot: {$mergeObjects: ["$aleft", "$bright", "$$ROOT"]}  
    }  
  },  
  {  
    $project: {"aleft": 0, "bright": 0}  
  }  
])
```

## 分析

### aggregate

`db.collection.aggregate(管道，选项)` 方法参数接收一个包含了若干操作的数组，类似于 Linux 中的管道一样，对集合依次进行操作。

### project

第一个操作为 `$project` ，这是一个映射操作

```json
  {  
    $project: {"aleft": "$$ROOT", "_id": 0}  
  }
```

其中 `$$ROOT` 即引用顶级文档，效果是将当前文档（行）的所有数据放到 `aleft` 字段下。`"_id": 0` 代表隐藏 `_id` 行，当设定为 `"_id": 1` 时会展示 `_id` 行，（`_id` 由 MongoDB 自动生成）结果示意如下：

```json
[  
  {  
    "aleft": {  
      "_id": {"$oid": "6247f1e253a1be11c3b88d8b"},  
      "id": 1,  
      "name": 12  
    }  
  },  
  {  
    "_id": {"$oid": "624ce445b67f62529d94a83e"}, // 当 _id: 1 时会展示 id 
    "aleft": {  
      "_id": {"$oid": "624ce445b67f62529d94a83e"},  
      "id": 2,  
      "name": 13,  
      "goods": 1  
    }  
  }  
]
```

### lookup

第二个操作为

```json
  {  
    $lookup: {  
      localField: "aleft.id",  
      from: "old",  
      foreignField: "id",  
      as: "bright" 
    } 
  }
```

顾名思义，这是一个查找操作，它根据第一步结果中的 `aleft.id` 字段，在 `old` 表中查找 `id` 与之相等的文档（行），并将所有匹配的结果以数组方式放在 `bright` 字段下，结果示意如下：

```json
[  
  {  
    "aleft": {  
      "_id": {"$oid": "6247f1e253a1be11c3b88d8b"},  
      "id": 1,  
      "name": 12  
     },  
    "bright": [  
      {  
        "_id": {"$oid": "624ce82ab67f62529d94a84c"},  
        "id": 1,  
        "name": "haha"  
      },  
      {  
        "_id": {"$oid": "624ce867b67f62529d94a84e"},  
        "id": 1,  
        "vbsss": "haha"  
      }  
    ]  
  }
]
```

### unwind

第三个操作为

```json
  {  
    $unwind: {  
      path: "$bright",  
      preserveNullAndEmptyArrays: true
    } 
  } 
```

这个操作指明了使用 `$bright` 字段，这个字段是一个数组，`$unwind` 操作会将 `$bright` 中的每一个元素与 `aleft` 组合

```json
[  
  {  
    "aleft": {  
      "_id": {"$oid": "6247f1e253a1be11c3b88d8b"},  
      "id": 1,  
      "name": 12  
    },  
    "bright": {  
      "_id": {"$oid": "624ce82ab67f62529d94a84c"},  
      "id": 1,  
      "name": "haha"  
     }  
  },  
  {  
    "aleft": {  
      "_id": {"$oid": "6247f1e253a1be11c3b88d8b"},  
      "id": 1,  
      "name": 12  
     },  
    "bright": {  
      "_id": {"$oid": "624ce867b67f62529d94a84e"},  
      "id": 1,  
      "vbsss": "haha"  
     }  
  }
]
```

至此，我们已经将两个表中关联的行组合在了一起，接下来需要将这个结构简化一下

### replaceRoot

第四个操作为

```json
  {  
    $replaceRoot: {  
      newRoot: {$mergeObjects: ["$aleft", "$bright", "$$ROOT"]}  
    }  
  },
```

### mergeObjects

`$mergeObjects` 操作会将参数中元素的内容进行合并，如果有重复，后面的值会覆盖前面的值，比如下面的这个文档

```json
  {  
    "aleft": {  
      "_id": {"$oid": "6247f1e253a1be11c3b88d8b"},  
      "id": 1,  
      "name": 12  
    },  
    "bright": {  
      "_id": {"$oid": "624ce82ab67f62529d94a84c"},  
      "id": 1,  
      "name": "haha"  
     }  
  },
```

执行 `$mergeObjects: ["$aleft", "$bright", "$$ROOT"]` 操作后结果如下

```json
  {  
    "_id": {"$oid": "6247f1e253a1be11c3b88d8b"},  
    "aleft": {  
      "_id": {"$oid": "6247f1e253a1be11c3b88d8b"},  
      "id": 1,  
      "name": 12  
    },  
    "bright": {  
      "_id": {"$oid": "624ce82ab67f62529d94a84c"},  
      "id": 1,  
      "name": "haha"  
    },  
    "id": 1,  
    "name": "haha"  
  }
```

### replaceRoot

`$replaceRoot` 将指定的文档提升到顶层，并丢弃顶层所有其他字段。

```json
[  
  {  
    "_id": {"$oid": "6247f1e253a1be11c3b88d8b"},  
    "aleft": {  
      "_id": {"$oid": "6247f1e253a1be11c3b88d8b"},  
      "id": 1,  
      "name": 12  
    },  
    "bright": {  
      "_id": {"$oid": "624ce82ab67f62529d94a84c"},  
      "id": 1,  
      "name": "haha"  
    },  
    "id": 1,  
    "name": "haha"  
  }
]

```

至此，只要将 `aleft` 和 `bright` 两个参数删掉就得到了最后的结果，操作为

```json
  {  
    $project: {"aleft": 0, "bright": 0}  
  }
```

结果示意如下

```json
[  
  {  
    "_id": {"$oid": "6247f1e253a1be11c3b88d8b"},  
    "id": 1,  
    "name": "haha"  
  }
]
```

## 其他

MongoDB 中的 JOIN 操作还是十分复杂的，与 MySQL 不同， MongoDB 中的文档结构并没有限制，所以可以采用[嵌套文档](https://www.mongodb.com/docs/v4.2/tutorial/query-embedded-documents/)的方式将本来需要关联的数据保存在一起，从而避免 JOIN 操作