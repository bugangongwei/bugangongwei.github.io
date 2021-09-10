---
title: 面试题
date: 2021-08-16 14:10:00
tags: [interview]
categories: interview
comments: true
---


# 业务
## 简历
负责口语课模块的维护和重构
口语课因为普及度高, 受众范围广, 成为公司的核心课程, 面向 80% 的活跃用户, 所以对用户体验有很高的要求;
配合数据组完成了 mongodb 定时备份;
对用户学习记录单表进行分库分表消除大表隐患;
给口语课 DB 增加缓存, 提高响应速度;

负责精细化运营系统资源管理模块的迭代
资源管理负责对广告, 弹窗, 推荐课等资源的后台管理和 C 端内容提供, 使用 mysql 进行资源存储, redis 进行缓存;
通过缓存自动更新策略预防缓存中可能会发生的缓存雪崩, 缓存穿透和缓存击穿等问题;
通过 pprof 分析定位内存泄漏问题并通过 LRU map 解决;

负责创新项目第一期的业务迭代
完成了用户注册登陆模块, 后台内容管理系统, 以及课程内容录入模块的开发;

## 技能回顾
TODO:
golang 面试题练习
并发安全相关
mysql 慢查询定位和优化
redis + mongodb
k8s 组件
微服务服务治理(服务发现, 配置管理, 流量监控, 日志监控, 容错容灾)
ES 的使用和原理

## 简历回顾
### 配合数据组完成了 mongodb 定时备份
(1) gocraft 中的定时任务, gocraft 组件的原理(看过, 但是讲不出来, 需要总结)
(2) mongodb dump (其实就是拷贝 json 文档, 但是需要总结一下)
(3) OSS 的应用场景及优势

### 对用户学习记录单表进行分库分表消除大表隐患
1. 分库分表数: 查询表目前的数据量, 依据 RDS 性能优化建议, 设计分库分表数;
RDS 性能优化建议:
(1) 实例中表的数量不超过 1w 张;
(2) table_rows 最好在 500w 以内;
(3) table_size 最好在 20G 以内;

2. 锚点: 找到最近创建的一个 id 做为锚点
````
select id from user_courses order by created_at desc limit 1;
````

3. 历史数据迁移
锚点前的数据全部作为历史数据, 需要读原表数据再插入新表, 对原表的读造成影响, 对目标表的写造成影响(但是一开始目标表是空的, 没关系), 所以还是需要选择在流量低峰期运行, 大概迁移了 10 个小时左右;

迁移方法:
定义一个初始的 startID 和 endID, 以及一个 step 变量, 和一个 concurrency;
我们初始化 concurrency 个 go routinue 用来处理查询旧表和插入新表;
定义一个 chan, 将 startID 和 endID 按照 step 大小分段, 插入到 chan 中, 等待 goroutinue 接收并处理;
go routine 监听 chan, 一旦读到数据, 就进行处理;


4. CDC 增量式迁移 + 客户端调用
锚点后的数据, 通过 CDC 消费 binlog 数据, 写入新的表里, CDC 只保留三天的数据, 所以历史数据迁移和 CDC 消费开启控制好时间点;
同步追平之后, 把服务的 DB 连接从老表切换到新表, 此时, 数据写入都写入了新表, 老表可以停用;
我们业务因为老表被老服务使用, 老版本的客户端用户还有请求, 所以 CDC 并没有马上断开, 而是在线上跑着, 直到下线老服务;

迁移方法:
接收 kafka 数据, 分析消息类型, before 为空, after 不为空, 表示插入, before 和 after 都不为空表示更新, before 不为空, after 为空表示删除;
按照不同的消息类型, 对数据进行插入, 更新和删除操作 (业务中是软删除, 所以删除也是更新);

#### 通过慢接口分析定位到口语课的加载速度慢, 于是给口语课 BD 增加缓存, 提高响应速度
hunter 定位慢接口


# 数据库
### 对 SQL 的优化方式
1. 子查询: 如果查询的两个表大小相当，那么用 in 和 exists 差别不大；如果两个表中一个较小一个较大，则子查询表大的用 exists，子查询表小的用 in;
````sql
-- 例如：表A(小表)，表B(大表)
-- A 表使用 cc 索引, B 表全面扫描, 不划算
select * from A where cc in (select cc from B);
-- A 全表扫描, B 使用 cc 索引, 划算
select * from A where exist(select cc from B where cc=A.cc);

-- 相反的
-- A 全表扫描, B 使用 cc 索引, 划算
select * from B where cc in (select cc fA 表使用 cc 索引, B 表全面扫描, 不划算
-- A 全表扫描, B 使用 cc 索引, 划算
select * from B where exist(select cc from A where cc=B.cc);
````

2. 子查询: 无论哪个表大，用 not exists 都比 not in 要快
````sql
-- 内外都要全表扫描
select * from t1 where c2 not in (select c2 from t2);
-- 内查询可以使用 c2 索引
select * from t1 where not exists (select c2 from t2 where c2=t1.c2);
````
bug: 如果子查询返回的值中有任意一条记录含有 null 值, 则整个查询不返回任何记录, 这个规则极易引起失误; 所以我们尽量不使用 not in;

3. 避免在索引列上使用 is null 和 is not null
注意⚠️事项: 单个索引不存储 null 值, 组合索引不存储全为 null 的值, 原因是索引应该是有序的, 但是 null 是破坏有序性的, 所以干脆不存;
替代方案: 
a. 把 NULL 转换成特定值, 使用特定值查找, a is not null 改为 a>0 等...
b. 建立复合索引;
效率分析:

````sql
create table student
(
    id int primary key not null ,
    sid   int
);
CREATE INDEX STU_SID ON STUDENT( SID    ASC);

-- 两种选择
-- (1) 索引扫描全部非 null 数据, 根据索引扫描扫描主键, 遇到索引扫描中的 id 就跳过
-- (2) 全表整块读取, 找出 is null 的数据
-- (1) 更快, 优化器会选择(1), 扫描索引 STU_SID, 找到所有非 null 的节点的 id (索引不保留 null 值)
-- 全表扫描, 一旦遇到上面这些数据, 不读取数据, 否则, 读取数据
select * from student where sid is null;

-- 两种选择
-- (1) 索引扫描全部非 null 数据, 根据索引扫描读取全部数据
-- (2) 全表扫描, 整块读取全部数据
-- (2) 更快, 优化器会选择(2), 因为(1)扫描完索引之后, 需要去主键一个一个读取数据; 而(2)全表扫描会一次读取一整个 block, 所以效率更高;
select * from student where sid is not null;
````

4. 考虑在 where 和 order by 后面的列建立索引
5. 如果数据是连续的, 能用 between 就不用 in
6. or 查询如果是相同字段, 可以用 in 替代

### 索引失效的情况
(1) 重复值太多的列 (如: TYPE 有 5 个值, 查找 TYPE = 1 时, 索引失效)
(2) 前导模糊查询 ("%XX", "%XX%"...)
(3) 函数导致的索引失效 (DATE())
(4) 类型不一致导致的索引失效 (其实类型转换是函数调用, 函数调用会使索引失效)
(5) 表达式导致的索引失效 (... where age-1=18)
(6) or 引起的索引失效 (or )
(7) not in, not exist 导致索引失效
(8) is null, is not null 导致索引失效
(9) 复合索引, 最左匹配原则
(10) 复合索引, 如果使用了 != 或 <> 会导致后面的索引全部失效

### mysql explain
1. id, id 越大, 查询优先级越高
2. select_type
(1) SIMPLE: 简单查询, 不包括子查询和 UNION 查询
(2) PRIMARY + SUBQUERY: 外部查询和子查询, 一般是一起的
(3) PRIMARY + DERIVED: 在from列表中包含的子查询会被标记为DERIVED(衍生)，MySQL会递归执行这些子查询，将结果放在临时表中。
(4) UNION: 若第二个select出现在union后，则被标记为UNION，若union包含在from子句的子查询中，外层select将被标记为DERIVED。
(5) UNION RESULT: 从union表获取结果的select。
3. sql 语句操作的表
4. partitions: 表所在的分区
5. type: 性能 system>const>eq_ref>ref>range>index>ALL 表示查询所使用的访问类型
(1) system: 表只有一行记录（等于系统表），是const的特例类型;
(2) const: 表示通过一次索引就找到了结果，常出现于 primary key 或 unique 索引, 如 ... where id = x ;
(3) eq_ref: 唯一索引扫描，对于每个索引键，表中只有一条记录与之匹配。常见主键或唯一索引扫描。
(4) ref: 非唯一性索引扫描，返回匹配某个单独值的所有行。本质上也是一种索引访问，返回匹配某值（某条件）的多行值，属于查找和扫描的混合体。
(5) range: 只检索给定范围的行，使用一个索引来检索行，可以在key列中查看使用的索引，一般出现在where语句的条件中，如使用between、>、<、in等查询。
(6) index: 全索引扫描，index和ALL的区别：index只遍历索引树，通常比ALL快，因为索引文件通常比数据文件小。虽说index和ALL都是全表扫描，但是index是从索引中读取，ALL是从磁盘中读取。
(7) ALL: 全表扫描
6. possible_keys: 显示可能应用在表中的索引，可能一个或多个
7. key: 实际中使用的索引，如为NULL，则表示未使用索引。若查询中使用了覆盖索引，则该索引和查询的select字段重叠。
8. key_len: 表示索引中所使用的字节数，可通过该列计算查询中使用的索引长度。在不损失精确性的情况下，长度越短越好。key_len显示的值为索引字段的最大可能长度，并非实际使用长度，即key_len是根据表定义计算而得，并不是通过表内检索出的。
9. rows: 根据表统计信息及索引选用情况大致估算出找到所需记录所要读取的行数。当然该值越小越好
10. filtered: 百分比值，表示存储引擎返回的数据经过滤后，剩下多少满足查询条件记录数量的比例
11. Extra: 额外信息
(1) Using filesort: 表明mysql会对数据使用一个外部的索引排序，而不是按照表内的索引顺序进行读取; 归并排序+快排
(2) Using temporary: 使用了临时表保存中间结果，常见于排序order by和分组查询group by
(3) Using index: 表明相应的select操作中使用了覆盖索引，避免访问表的额外数据行，效率不错。
如果同时出现了 Using where，表明索引被用来执行索引键值的查找
如果没有同时出现 Using where，表明索引用来读取数据而非执行查找动作

### B- 树和 B+ 树的区别
B- 树: 每个节点都存储数据
B+ 树: 只有叶子节点存储数据, 叶子节点间有相邻指针

### 为什么 mongodb 使用 B- 树而 mysql 使用 B+ 树呢?
1. mysql 结构型数据库
结构型数据库的设计遵循三范式, 即“尽可能地减少冗余”, 这就导致在设计实体关系模型时, mysql 通常会把数据拆分成多个表, 在需要查询更多信息时, 难免要进行关联查询;
而关联查询则需要把一个表的一组结构放到另一个表进行查询, 一般都要遍历的;

例如学生表 t_student(sid, sname, cid), 班级表 t_class(cid, cname)
````sql
SELECT * FROM t_student t1, (
    SELECT cid
    FROM t_class
    WHERE cname = '1班'
  ) t2
WHERE t1.cid = t2.cid
```` 
比如这个语句就难免要在 t_student 表中遍历 cid = 1 的情况(通过索引找到 cid = 1, 也要继续向右遍历知道找到第一个不等于 1 的 cid)

2. mongodb 聚合型数据库
mongo 也可以使用 lookup 进行类似 union 的联合查询, 但是一个好的非关系型数据库设计, 是反三范式的, 更好的情况是把数据都聚合在一起;

````json
{
  "class_1": {
    "_id": 1,
    "name": "class 1",
    "students": [
      {
        "_id": 1,
        "name": "a"
      },
      {
        "_id": 2,
        "name": "b"
      }
    ]
  },
  "class_2": {
    "_id": 2,
    "name": "class 2",
    "students": [
      {
        "_id": 3,
        "name": "c"
      }
    ]
  }
}
````
非关系型数据库的数据更聚合, 通常一次查询就查到足够丰富的信息, 范围查询的场景比较少;

3. 各自优势
mongodb 不注重范围查询, 但是注重等值查询, 所以使用 B- 树, 最好时间复杂度为 O(1), 最差为 O(logn), 平均查询复杂度比较好;
mysql 很注重范围查询, 所以使用 B+ 树, 稳定的时间复杂度为 O(lonn);

### 事务隔离级别
(1) 读未提交
(2) 读提交
(3) 可重复读
(5) 串行化, 顾名思义是对于同一行记录，“写”会加“写锁”，“读”会加“读锁”。当出现读写锁冲突的时候，后访问的事务必须等前一个事务执行完成，才能继续执行

### 脏读, 幻读
脏读: 当前事务读到了别的事务修改后未提交的数据;
幻读: 当前事务读到了别的事务插入的满足当前事务查询条件的数据;

### IO 多路复用
1. select
select的几大缺点：
（1）每次调用select，都需要把fd集合从用户态拷贝到内核态，这个开销在fd很多时会很大
（2）同时每次调用select都需要在内核遍历传递进来的所有fd，这个开销在fd很多时也很大
（3）select支持的文件描述符数量太小了，默认是1024

2. poll
poll 基于 select 的改进是, 把 fd_set 换成 pollfd 结构, 从限定长度的数组换成不限制长度的链表, 解决了问题 (3), 即 poll 没有最大文件描述符数量的限制;

3. epoll
原理: 红黑树+双向链表+回调机制

描述:
当某一进程调用epoll_create方法时，Linux内核会创建一个eventpoll结构体;
通过epoll_ctl方法向epoll对象中添加进来的事件, 这些事件都会挂载在红黑树中，如此，重复添加的事件就可以通过红黑树而高效的识别出来(红黑树的插入时间效率是lgn，其中n为树的高度);
所有添加到epoll中的事件都会与设备(网卡)驱动程序建立回调关系，也就是说，当相应的事件发生时会调用这个回调方法;
这个回调方法在内核中叫ep_poll_callback,它会将发生的事件添加到rdlist双链表中;
当调用epoll_wait检查是否有事件发生时，只需要检查eventpoll对象中的rdlist双链表中是否有epitem元素即可。如果rdlist不为空，则把发生的事件复制到用户态，同时将事件数量返回给用户

使用:
第一步：epoll_create()系统调用。此调用返回一个句柄，之后所有的使用都依靠这个句柄来标识。
第二步：epoll_ctl()系统调用。通过此调用向epoll对象中添加、删除、修改感兴趣的事件，返回0标识成功，返回-1表示失败。
第三部：epoll_wait()系统调用。通过此调用收集收集在epoll监控中已经发生的事件。


### mvcc
#### 快照读/可重复读
InnoDB 为每个事务构造了一个数组，用来保存这个事务启动瞬间，当前正在“活跃”的所有事务 ID。“活跃”指的就是，启动了但还没提交;
数组里面事务 ID 的最小值记为低水位，当前系统里面已经创建过的事务 ID 的最大值加 1 记为高水位。这个视图数组和高水位，就组成了当前事务的一致性视图（read-view）;

undo log 只针对 UPDATE 和 DELETE
可重复读的时候, 去查找当前事务维护的 undo log, 找到低水位以下的数据;

#### update
当前读
写 undo log

### 主键
自增 id 的优点
(1) 大小来看, 主键长度越小, 主键索引节点占据空间越小, 普通索引叶子节点占据空间越小;
(2) 自增, 插入过程中 ID 是有序插入, 那么, 可以不用考虑页分裂问题, 提升性能;


### redis 五种数据结构和底层数据结构
1. string: set/get/incr/decr/incrby/decrby
2. hash: hset/hget/hmget
3. list: 先入先出, lpush/lpop/rpush/rpop/llen
4. set: 无重复元素, sadd/scard(元素个数)/sismember(元素是否存在)/srem(删除某个元素)
5. zset: sorted set, zadd/zcard/zrange





