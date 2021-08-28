---
title: 面试题
date: 2021-08-16 14:10:00
tags: [interview]
categories: interview
comments: true
---

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













