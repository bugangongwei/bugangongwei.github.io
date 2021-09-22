---
title: mysql
date: 2021-07-10 14:10:00
tags: [mysql]
categories: Mysql
comments: true
---

{% asset_img mysql_structure.png mysql架构 %}

# mysql 架构
## 连接器
管理连接, 权限验证
``` mysql -h$host -P$port -u$user -p ```
(1) 在客户端与服务端之间建立 TCP 连接
(2) 客户端向服务端发送用户名和密码验证
(3) 如果用户名密码错误, 返回错误: Access denied for user
(4) 连接器查询权限表获取用户的权限, 此后这个连接里面所有的权限判断逻辑都基于此时读到的权限

查看 mysql 的所有 thread 情况(对应不同的连接)
``` mysqladmin -u root -p processlist ```

查看 mysql 长链接最大空闲时间
``` show global variables like 'wait_timeout'; ```


## 查询缓存
在频繁更新的数据库中, 每次更新数据都会清空查询缓存, 在这种情况下, 查询缓存是不推荐使用的, mysql8.0已经去掉查询缓存
显式指定需要查询缓存
select SQL_CACHE * from T where ID=10；

## 分析器
分成解析器和预处理器, 解析器分成词法分析和语法分析两个阶段, 预处理器进一步检查语法书的合理性, 比如, 数据表和数据列是否存在, 别名是否有歧义等

## 优化器
在多个索引的时候, 选择用哪个索引, 在多表 join 的时候, 选择表的连接顺序

## 执行器
(1) 判断用户是否有表的执行权限
(2) 有权限的情况下, 就通过存储引擎打开表进行查询;

# 日志系统
## redo log
WAL (Write-Ahead Logging)技术: 先写日志, 再写磁盘;
crash-safe 能力: 当数据库意外重启时, 因为有 innodDB 的 redo log 日志, 所以数据不会丢失;

在进行更新操作的时候, innoDB 引擎会先把更新日志写入 redo log, 更新内存, 这就算完成了更新操作; 在空闲的时候, innoDB 会将 redo log 里面的操作日志写入磁盘; 如果 redo log 写满的情况下, innoDB 需要先擦掉一些数据才能允许新的更新操作;

{% asset_img redo_log.png redo log %}
write pos: 当前写入的位置
check point: 当前擦出的位置
当 write pos 追上 check point 时, 表示 redo log 写满了;

## binlog
server 层的归档日志, 可以给所有的存储引擎用
binlog 和 redo log 不一样, 它是没有固定空间大小的, 只要有日志就可以追加写, 且, 二者的日志信息不一样, redo log 写的是对一条数据的修改, binlog 写的是 mysql 语句的原始逻辑;


有了对这两个日志的概念性理解，我们再来看执行器和 InnoDB 引擎在执行这个简单的 update 语句时的内部流程。
``` update T set c=c+1 where ID=2; ```
(1) 执行器先找引擎取 ID=2 这一行。ID 是主键，引擎直接用树搜索找到这一行。如果 ID=2 这一行所在的数据页本来就在内存中，就直接返回给执行器；否则，需要先从磁盘读入内存，然后再返回。
(2) 执行器拿到引擎给的行数据，把这个值加上 1，比如原来是 N，现在就是 N+1，得到新的一行数据，再调用引擎接口写入这行新数据。
(3) 引擎将这行新数据更新到内存中，同时将这个更新操作记录到 redo log 里面，此时 redo log 处于 prepare 状态。然后告知执行器执行完成了，随时可以提交事务。
(4) 执行器生成这个操作的 binlog，并把 binlog 写入磁盘。
(5) 执行器调用引擎的提交事务接口，引擎把刚刚写入的 redo log 改成提交（commit）状态，更新完成。
{% asset_img update语句执行过程.png update语句执行过程 %}


# QA 
1. 为什么 next-key 的前提条件是 RR 隔离级别?
next-key = gap lock + row lock
间隙锁的引入就是针对 RR 隔离级别下的 "幻读" 等问题而提出的;


# CDC
log_bin=ON, binlog_format=ROW, binlog_row_image=FULL
binlog_format 分为 ROW, STATEMENT 和 MIXED, ROW 记录每一条 SQL 语句; STATEMENT 记录每一行数据; MIXED 是两者的结合;
binlog_row_image 分为 FULL 和 Minimal, FULL 展示所有列的值, Minimal 只展示变化的列的值;

binlog_format=ROW, binlog_row_image=FULL:
````binlog
#200702 11:26:00 server id 1  end_log_pos 2647 CRC32 0x59777bf5     Delete_rows: table id 120 flags: STMT_END_F

BINLOG '
yFP9XhMBAAAAOQAAACcKAAAAAHgAAAAAAAEABHRlc3QAC2JpbmxvZ19kZW1vAAMDAxEBAAIaymaN
yFP9XiABAAAAMAAAAFcKAAAAAHgAAAAAAAEAAgAD//gEAAAABAAAAF7/VgD1e3dZ
'/*!*/;
### DELETE FROM `test`.`binlog_demo`
### WHERE
###   @1=4 /* INT meta=0 nullable=0 is_null=0 */
###   @2=4 /* INT meta=0 nullable=1 is_null=0 */
###   @3=1593792000 /* TIMESTAMP(0) meta=0 nullable=0 is_null=0 */
# at 2647
````

binlog_format=ROW, binlog_row_image=MINIMAL
````binlog
#200702 11:26:00 server id 1  end_log_pos 2647 CRC32 0x59777bf5     Delete_rows: table id 120 flags: STMT_END_F

BINLOG '
yFP9XhMBAAAAOQAAACcKAAAAAHgAAAAAAAEABHRlc3QAC2JpbmxvZ19kZW1vAAMDAxEBAAIaymaN
yFP9XiABAAAAMAAAAFcKAAAAAHgAAAAAAAEAAgAD//gEAAAABAAAAF7/VgD1e3dZ
'/*!*/;
### DELETE FROM `test`.`binlog_demo`
### WHERE
###   @1=4 /* INT meta=0 nullable=0 is_null=0 */
# at 2647
````

STATEMENT:
````binlog
#200702 11:58:33 server id 1  end_log_pos 2961 CRC32 0x48d934c7     Query    thread_id=10    exec_time=0    error_code=0
use `test`/*!*/;
SET TIMESTAMP=1593662313/*!*/;
insert into binlog_demo values(6,66,'2020-07-06')
/*!*/;
# at 2961
````

备注:
server id 用来唯一标识 Mysql Server 主机, 在双主模式下, 可以防止 bin_log 循环复制问题;
SET TIMESTAMP=1593662313 是 STATEMENT 模式下的上下文信息, 比如使用 now() 函数的时候, 由于主从主机时间可能不一致, 需要这个 TIMESTAMP 来一致化时间;

ROW 模式: 优势是没有歧义, 每一行都很清楚地表达; 劣势是数据量可能比较大, 比如批量删除数据, 就需要记录很多行记录;
STATEMENT 模式: 优势是只需要记录 SQL 语句, 行数相对 ROW 模式少很多; 劣势是执行入 DELETE...limit n 等语句是, 可能由于主从选择的索引不一致导致最终结果不一致;
````
mysql> select * from binlog_demo;
+----+----+---------------------+
| id | a  | t_modified          |
+----+----+---------------------+
|  1 |  1 | 2020-07-01 00:00:00 |
|  2 |  2 | 2020-07-02 00:00:00 |
|  3 |  3 | 2020-07-02 00:00:00 |
|  5 |  5 | 2020-07-05 00:00:00 |
|  6 | 66 | 2020-06-01 00:00:00 |
+----+----+---------------------+

-- 使用索引 a
mysql> select * from binlog_demo   where a>=4 and t_modified<='2020-07-10' limit 1;
+----+---+---------------------+
| id | a | t_modified          |
+----+---+---------------------+
|  5 | 5 | 2020-07-05 00:00:00 |
+----+---+---------------------+

-- 使用索引 b
mysql> select * from binlog_demo use index(t_modified)  where a>=4 and t_modified<='2020-07-10' limit 1;
+----+----+---------------------+
| id | a  | t_modified          |
+----+----+---------------------+
|  6 | 66 | 2020-06-01 00:00:00 |
+----+----+---------------------+
````
MIXED: 两种方式的结合, Mysql 自动判断使用 ROW 还是 STATEMENT;

为什么现在 Mysql 默认使用 ROW 模式?
(1) 就缺点而言, 目前的大数据背景下, 硬件已经变得成本不算太高, 所以浪费空间也变得可以接受;
(2) 就优点而言, ROW 模式对 update, delete 和 insert 都有很好的回滚策略, 因为它记录了变化前和变化后的数据, 非常精确, 不用过于关心不一致的问题;

# redolog binlog 两阶段提交
(1) 执行器想要更新记录 A
(2) 查找记录 A, 如果 A 已经在 buffer pool 中, 直接从 buffer pool 取, 否则 InnoDB 从磁盘中夹在记录 A 到 buffer pool;
(3) 将记录 A 的旧值写入 undolog, 便于回滚;
(4) 在 buffer pool 中更新记录 A, 此时该页变成脏页;
(5) 执行器写 redolog 并标记为 prepare 状态;
(6) 执行器写 binlog 文件;
(7) 执行器写 redolog 并标记为 commit 状态, 事务 commit;
如果 innodb_flush_log_at_tx_commit=1, 会在每次 commit 时持久化 redolog 到磁盘; 如果 sync_binlog=1, 会在每次 commit 时持久化 binlog 到磁盘;

如何判断 binlog 和 redolog 是否一致? 通过 redolog 和 binlog 中记录的 XID, 也就是事务 ID;




















