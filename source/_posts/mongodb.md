---
title: mongoDB
date: 2021-08-16 14:10:00
tags: [mongoDB]
categories: 数据库
comments: true
---

# 概念 
mongoDB 是一个以 json 为数据模型的文档型数据库; 非常方便程序员开发;
在业务设计中, 选择 mongoDB 要考虑 lost data 的风险;

# 基本操作
启动 mongod
````
bin/mongod --dbpath /usr/local/mongodb/data/db/ --logpath /usr/local/mongodb/logs/mongodb.log --logappend --port 27017 --bind_ip 0.0.0.0 --fork

--dbpath: db 数据存放的目录, 对表 mysql 数据库, redis RDB;
--logpath: db 日志目录;
27017: 默认端口
--bind_ip 0.0.0.0: 开放权限, 任何客户端都可以访问
--fork: 启动为后台进程
````

配置启动文件
``` 
bin/mongodb.conf 

bin/mongod -f bin/mongodb.conf

bin/mongod -f bin/mongodb.conf --shutdown

```

客户端命令
```
# 如果 db 里面没有文档, 就不限制那个 db
show databases

show collections

help

db.help()

db.version()

use db_name

db.shutdownServer
```

创建 admin 用户
````
# use 既可以切换数据库也可以创建数据库
use admin

db.createUser({user:"user_name",pass:"pa.."...})

bin/mongod -f bin/mongodb.conf --auth

use admin

db.auth("user_name", "pa..")
````

创建普通用户
````
# auth 管理员用户
db.auth("user_name", "pa..")

use test

db.createUser({user:"user_name",pass:"pa.."...})
...
````

集合
````
db.createCollection()
````

集群
````
rs.init()
rs.add(ip:port)
````

# 复制集

## 数据是如何恢复的
(1) 当一个修改操作到达主节点, 这个操作会被记录在 oplog 中
(2) 从节点通过在主节点上打开一个 tailable 游标不断地获取新进入节点的 oplog, 并在自己的数据上进行回放, 以此保持跟主节点的数据一致;

## 通过选举完成故障恢复
(1) 具有投票权的节点之间两两互相发送心跳检测;
(2) 当 5 次心跳未收到时, 判断节点失联;
(3) 如果失联的是主节点, 从节点会发起选举, 重新选择主节点;
(4) 选举基于 RAFT 一致性算法, 前提是大部分的投票节点存活;
(5) 复制集中最多有 50 多个节点, 但具有投票权的最多 7 个;

## 被选举为主节点的节点需要具备以下优势
(1) 能够与较多节点建立连接
(2) 具有较新的 oplog
(3) 具有较高的优先级(设置)

## 节点的可选配置
vote: 是否具有投票权
priority: 优先级, 优先级越高越容易被选举成主节点, 优先级为 0 的节点不可作为主节点;
hidden: 应用不可见的节点, 可以有投票权, 优先级必须为0;
slaveDelay: 延迟时间, 复制 n 秒之前的数据, 保持与主节点的时间差; 防止复制节点马上同步主节点数据, 在某些场景下, 其实可以防止误操作瞬间蔓延整个集群;

# 文档模型

## 模型设计元素
实体 Entity: 描述业务的主要数据集合
属性 Attribute: 描述实体中的单个信息
关系 Relationship: 描述实体与实体间的规则

## Json 文档模型
实体 Entity: 集合
属性 Attribute: 字段
关系 Relationship: 内嵌数组, 引用字段

## 什么时候使用引用方式
内嵌: Json 内嵌 Json 或者 Json 数组
引用: Json 内引用其他 Json 对象
(1) 内嵌文档太大, 数 MB 或者超过 16MB
(2) 内嵌文档或数组会频繁修改
(3) 内嵌数组元素会持续增长并且没有封顶

## 设计模式一: 分桶
比如在时序数据中, 通过把一小时数据内嵌一个 60 个文档的 events, 完成对数据的分桶, 减少数据冗余和内存使用;


# 索引

## b- 树
mongodb 的索引模型是 B- 树;
B树/B-树: 二叉变多叉
B+树: 仅叶子节点存储数据
B+ 树相对于 B- 树的优势:
(1) b+树的中间节点不保存数据，所以磁盘页能容纳更多节点元素，更“矮胖”；
(2) b+树查询必须查找到叶子节点，b树只要匹配到即可不用管元素位置，因此b+树查找更稳定（并不慢）；
(3) 对于范围查找来说，b+树只需遍历叶子节点链表即可，b树却需要重复地中序遍历，如下两图：

``` db.cl.find({name:"lsp"}).explain(true) ```
IXSCAN: 索引扫描
COLLSCAN: collection 扫描

## ESR 原则: 组合索引的最佳使用原则
精确(Equeal): 匹配查询字段放前面
排序(Sort): 排序条件放中间
范围(Range): 范围匹配放最后

## 其他有意思的索引
(1) 地理索引: 可查左上角到右下角这个矩形内的所有坐标点
(2) text索引/全文索引: 模糊查询, 不及 ES 强大但是可以满足一些不太大的需求
(3) 部分索引: 支队字段满足某些条件的值建立索引, 这种可以省略对历史记录的索引

















