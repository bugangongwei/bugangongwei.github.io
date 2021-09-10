---
title: 面试真题
date: 2021-08-16 14:10:00
tags: [interview]
categories: interview
comments: true
---

# B 站一
## GRPC 连接的四种模式
(1) 一元RPC
(2) 服务端流式RPC
(3) 客户端流式RPC
(4) 双向流式RPC

## 超时控制
context.WithTimeout(parent Context, d time.Duration)(Context, CancelFunc)
context.WithDeadline(parent Context, timeout time.Time)(Context, CancelFunc)
select + time.After(d time.Duration) <-chan time.Time

## context 的实现机制
Context interface 提供了四个方法, Deadline(), Done(), Err(), Value(key interface{}), 所有实现这四个方法的 context struct 都能完成取消, 超时和传递值这三个功能;

## 控制子goroutine退出运行
(1) goroutine panic 或者 main 函数结束, 会自动退出子 goroutine
(2) 全局变量控制
(3) channel 控制
(4) context.WithCacel(parent Context)(Context, CancelFunc)

## 页面的 UV 使用什么方式存储?
UV(Unique Visitor), 指通过互联网访问这个网页的自然人, 一般指的是一个账号;
(1) 不需要排序: set/hash
set 的底层实现是 intset 或者 hashtable
如果元素类型都是整数并且长度不超过 512 的话, set 的底层实现就是 intset, 否则就用 hashtable;

hash 的底层实现是 ziplist 或者 hashtable, 当哈希对象保存的所有键值对的键和值的字符串长度都小于64字节并且哈希对象保存的键值对数量小于512时, 哈希对象底层使用 ziplist 结构;

(2) 需要排序: zset
zset 的底层实现是 ziplist + hashtable 或者 skiplist + hashtable
ziplist 和 skiplist 的查找时间复杂度都不太理想, 所以引入了 hashtable;
hashtable 保存的数据是无序的, 所以需要 skiplist 或者 ziplist 来实现范围查询;

(3) 是否可以使用 bitmap 来存储?
可以, UV 可以用 user_id 来表示, user_id 是唯一的账户, 比如总共有 20 亿用户, 那么需要一个长度为 20 亿的 bitmap, 写入时, bitmap[user_id]=1, 查找时, bitmap[user_id], 统计时, 使用 O(N) 的负责度遍历; 这个 O(N) 的 N=20 亿, 和使用 redis 的 set, hashmap 的 O(N) 不一样;

## bitmap 的应用场景:
(1) 找出一个数是否在海量数据中存在
(2) 快速排序, 把一组数据 O(M) 遍历修改 bitmap, 然后 O(N) 遍历一遍 bitmap 就得到了有序的数组;
(3) 一个数有三种状态，00表示不存在，01表示存在1次，11表示存在1次以上。将数字用两个bit表示。最后，将状态位为01的进行统计，就得到了不重复的数字个数，时间复杂度为O(n)
(4) bloom 过滤器, 用来判断一个数不存在, 用在过滤场景中用来过滤不存在的数据

## select * from tb_user where uid = xxx for update 分析
uid not null, unique key
(1) 查到数据, 给 uid 索引和主键索引对应的数据加上行锁;
(2) 查不到数据, 给扫描到的数据间加上间隙锁;

## update 锁
update 语句自动加互斥锁;

## insert 锁
### Insert Intention Lock
假设现在有记录 10， 30， 50， 70 ；且为主键 ，需要插入记录 25 。
(1) 找到 小于等于25的记录 ，这里是 10
(2) 找到 记录10的下一条记录 ，这里是 30
(3) 判断 下一条记录30 上是否有锁
判断 30 上面如果 没有锁 ，则可以插入
判断 30 上面如果有Record Lock，则可以插入
判断 30 上面如果有Gap Lock/Next-Key Lock，则无法插入，因为锁的范围是 (10, 30) /（10, 30] ；在(10,30)上增加insert intention lock（ 此时处于waiting状态），当 Gap Lock / Next-Key Lock 释放时，等待的事物（ transaction）将被 唤醒 ，此时(10,30)上才能获得 insert intention lock ，然后再插入 记录25
注意：一个事物 insert 25 且没有提交，另一个事物 delete 25 时，记录25上会有 Record Lock

插入意向锁的作用是为了提高并发插入的性能， 多个事务 同时写入 不同数据 至同一索引范围（区间）内，并不需要等待其他事务完成，不会发生锁等待。

### 自增锁 AUTO-INC locks
(1) 自增锁 + `select max(auto_inc_col) from table` 自增锁是表锁, 性能比较差
(2) 互斥量 并发问题导致自增值可能不一定连续

## 浏览器中输入网址, 经历了哪些网络过程
https://zhuanlan.zhihu.com/p/133906695
1. 输入 url 网址
2. DNS 域名解析
解析过程: 本地 hosts 文件 -> 本地域名服务器 -> DNS 根服务器 -> .com 域名服务器 -> .llsapp.com  域名服务器 -> 返回 ip 地址 -> 缓存在本地域名服务器中;
3. 浏览器向 web 服务器发送一个 http 请求
(1) 经过重重解析, 对报文重重拨开, 会进入到内核的 TCP/IP 协议栈, 发起 TCP/IP 连接请求;
(2) 三次握手建立连接
(3) 发送一个 http 请求, 包括请求方法URI协议/版本, request header, 请求正文
4. 服务器的永久重定向响应
如果你的域名进行了更改, 不带有 www, 那么 server 可以把旧的地址重定向到新的地址去
301 和 302 重定向的区别, 301 表示旧的资源已经彻底不用了, 302 表示旧的资源还存在, 但结果都是会重定向到新的地址;
5. 浏览器跟踪重定向地址
6. 服务器处理请求
7. 服务器返回一个 http 响应
8. 浏览器显示 HTML
9. 浏览器发送请求获取嵌入在 HTML 中的资源（如图片、音频、视频、CSS、JS等等）

## 为什么 TCP 使用三次握手来建立连接
https://draveness.me/whys-the-design-tcp-three-way-handshake/
前提: TCP 连接的建立是由接收方完成的, 历史连接的判断是由发送方完成的
『两次握手』：无法避免历史错误连接的初始化，浪费接收方的资源；
『四次握手』：TCP 协议的设计可以让我们同时传递 ACK 和 SYN 两个控制信息，减少了通信次数，所以不需要使用更多的通信次数传输相同的信息；
### 两次握手为什么不行
历史连接问题, 过期的连接请求到达发送方, 发送方只能知道 [SEQ=xxx, CTL=SYN] 这两个信息, 无法判断连接是否是历史连接, 只能选择接受或者拒绝, 一旦接受, 就会造成接收方资源的浪费, 一旦拒绝, 可能拒绝到正常的连接请求;

三次握手怎么做的? 发送方一旦接受到接受方的返回信息 [SEQ=xxx, ACK=SEQ+1, CTL=SYN+ACK], 根据本地时钟或者当前的 SEQ, 可以判断这个回复是否来自历史连接, 如果是历史连接, 发送 CTL=RST 来进行请求中断, 客户端收到 RST 就不用建立历史连接了;  

### 四次握手为什么不行?
#### SEQ 的作用
(1) 接收方通过序列号去重;
(2) 接收方通过期望的序列号判断数据包是否丢失;
(3) 接收方通过序列号来对数据包进行排序;

client 作为发送方, server 作为接收方的情况下, 需要通过两次握手来进行去重, 丢失判断和排序;
client 作为接收方, server 作为发送方的情况下, 需要通过两次握手来进行去重, 丢失判断和排序;
总共需要四次握手, 但是, 在 server 端向 client 发送回复的时候, 可以携带询问信息, 把两次握手合成一次握手, 总共三次握手;
结论是, 三次握手已经能够建议一个可靠的 TCP/IP 连接, 不需要四次或者更多次的连接来浪费资源;

## 为什么 TCP 断开连接需要四次挥手
和 TCP 建立连接一样, TCP 断开连接也需要四次操作, 但是它不可以合并消息
(1) client 发送 [FIN=M], 用来关闭 client 对 server 端的数据传送, client 进入 FIN_WAIT_1 状态;
(2) server 发送 [ack=M+1] 给 client 应答确认, server 进入 CLOSE_WAIT 状态; 此时, server 端可能还有数据没有发送完, 继续发送
(3) server 发送 [FIN=N] 给 client, 用来关闭 server 端对 client 的数据传输,  server 进入 LAST_ACK 状态;
(4) client 发送 [ACK=N+1] 给 server 应答确认, 此时, client 进入 TIME_WAIT 状态, Server 收到消息后, 进入 CLOSED 状态;

(2)(3) 中间还有数据发送的可能, 所以, (2)(3) 不合并发送;

## HTTP2 
https://www.zhihu.com/question/34074946
### HTTP2 的不变
(1) 协议名不变, 用户感知不到, 平滑升级
(2) 语义层, 如请求方法、状态码、头字段等保持不变
### HTTP2 的变
(1) 头部压缩
静态字典: 高频 header, 61 组, 传输中只需要传输 index 就行, 如 `method: GET` 缩减为 2
动态字典: 静态字典之外的字段, index 从 62 开始, 动态添加
Huffman 算法: 二进制压缩
(2) 二进制帧
HEADERS frame: 头部帧
DATA frame: 数据帧
压缩算法: HPACK
(3) 并发传输/多路复用
HTTP1.x 通过 TCP 连接实现并发, 每个 stream 都有新起一个 TCP 连接;
HTTP2 通过 stream 实现并发, 一个 TCP 连接可以并发发送很多 stream, 一个 stream 携带很多的 Message, 一个 Message 包含多个 frame;
stream 内部的帧是严格有序的, 但是 stream 是并发无序的, 但是每个 stream 携带了保留顺序的 stream_id;
(4) 服务器主动推送
HTTP1.x 不支持服务器主动推送
### HTTP2 的可改进点
HTTP2 是基于 TCP 协议的, TCP 必须保证收到的数据是连续的完整的, 才会把缓冲区内的内容返回给 HTTP 应用, 如果数据没有完整到达, 缓冲区会一直等待, 这就造成了 HTTP2 队头阻塞问题;

HTTP3 针对这个问题, 准备放弃 TCP 协议, 改成使用 UDP 协议;

## 实现斐波那契数列
```
func dp(n int) int{
  if n == 0 {
    return 0
  }

  // dp[i] 表示 i 的斐波那契值
  // dp[0] = 0
  // dp[1] = 1
  // 遍历从 2 开始

  var dp = make([]int, 2)
  dp[0], dp[1] = 0, 1

  for i := 2; i<= n ;i++ {
    result := dp[0] + dp[1]
    dp[0] = dp[1]
    dp[1] = result
  }

  return dp[1]
}
```

# B 站二
## 挑有难点的业务项目
缓存自动更新组件
要解决的问题: 减轻源数据/DB压力, 提高接口响应速度, 源数据和接口响应的解耦问题;
要预防的问题: 缓存穿透, 缓存雪崩, 缓存击穿, 流量控制, key 元数据长度控制, 扩展性;
方式方法:
减轻源数据/DB压力, 提高接口响应速度: 在源数据之上增加缓存;
源数据和接口响应的解耦问题: 请求尽量都打到缓存上, 让缓存来回源, 实现业务和数据解耦;
回源触发: 请求触发, 异步回源;
并发请求, 并发回源: 分布式锁, 并发回源条件下, 只允许一个回源操作拿到锁;
请求抢锁, 锁竞争: 设置回源间隔时间, 减少抢锁次数;
缓存雪崩: key 过期时间设置, 业务差异, 回源重新设置过期时间;
缓存穿透: 不存在的 key 或者源数据被删除的 key 缓存, 设置较短的过期时间;
缓存击穿: 分布式锁;
流量控制: 对算法接口的访问, 可以采用有损的流量控制, 使用令牌桶, 漏斗, 计数器算法;
key 元数据长度控制: LRU Cache, cachegroup 组件采用 map 和 环形双向链表实现的非并发安全的 LRU Cache, 控制元数据数量, 防止内存泄漏;
扩展性: 使用了策略模式, 仅提供 Loader, Cacher, Lock 的接口, 实际上, 可以灵活换成任意的组件;

## Redis 分布式锁
### Redis 分布式锁的实现
(1) 原子加锁, `set key value nx ex t`
(2) value 唯一, 避免误解锁
(3) 超时解锁导致并发, 为获取锁的线程增加守护线程, 为将要过期的锁增加过期时间
(4) 可重入锁, 用 Redis Hash 计数
(5) 不想阻塞等待锁释放时, 可以考虑使用 Redis 发布订阅模式, 获取锁失败时, 订阅锁释放消息, 锁释放时, 发送锁释放消息;
### Redis 集群带来的分布式锁问题
(1) 主备切换: 主备切换过程中可能导致新的 master 分配新的锁, 导致两个锁同时存在;
(2) 集群脑裂: 集群脑裂会使 sentinel 失去 master 的消息, setinal 会选举新的 master, 此时可能同时存在两个锁;
### Redis 发布订阅模式
`SUBSCRIBE channel1`
`PUBLISH channel1`
### Redis 集群
Redis-Cluster: Redis 复制集的集合
Redis 复制集: master-slave 模式, 从服务器复制主服务器的数据, 并在主服务器宕机时, 担任新的主服务器, 实现高可用
分片和槽指派: 将 16384 个槽分派给集群中所有的节点, 只有在所有节点都指派了槽之后, redis 集群才正式启动;
重分片: 增加或者减少节点的时候, redis 集群会重新分配槽
去中心化: Redis 集群没有控制中心, 请求打到任意一个节点, 节点会判断该槽是否由自己负责, 有则执行命令, 没有就返回 MOVED 错误给客户端并指向正确的节点; 所以, redis 节点的数据结构不仅保存了自己的槽信息, 还保存了所有其他节点的槽信息;
gossip: 完成槽指派, 重分片, 握手等工作后, 会通过 gossip 协议把信息同步到其他节点;

## 限流策略/并发控制
### 有损的
(1) 计数器
(2) 漏斗算法
(3) 令牌桶算法
### 无损的
(1) Redis 集群, 提高 Redis 并发处理能力
(2) 多级缓存, 减轻 Redis 负载压力

## 介绍下 redis 单线程模型是如何运行的, 优缺点
redis 使用单线程 Reactor 模型 + IO 多路复用来监听客户端请求; 一旦命令请求处理事件发生, 便把事件放入队列中, 命令分配器会从队列中取出命令, 执行命令, 然后再从队列中取下一个命令, 如此循环, 命令的读取, 执行和写入都是单线程的;

优点:
(1) 内存读写, 速度很快
(2) 单线程没有上下文切换和复杂的设计, 简单而快速
(3) IO 多路复用, 不需要阻塞等待命令请求

缺点:
命令执行前后的命令读和回复写是单线程的, 使网络 IO 的压力成为了 redis 的性能瓶颈; 这也是 Redis6.0 优化的重点, 在这里做了多线程优化;

## Redis6.0 多线程如何实现的?
Redis6.0 主要针对网络 IO 瓶颈和多核计算机的特性, 允许在命令读和回复写阶段使用多线程模型, 提高多核的利用效率, 减少 IO 负荷;

## 如何提高 redis 对 cpu 的利用率
(1) 如果是 Redis6.0 的话, 可以开启多线程;
(2) 不考虑多线程的情况下, 可以在一台主机上增加 Redis 实例, 可以使用哨兵模式对 redis 进行监控;

## select * from table wher x=1 and y=2 order by z, 如何添加索引
最好是联合索引(x,y,z), 首先查询 x=1, 然后根据有序的 y, 查找 y=2, 然后 z 在创建索引时已经有序了;

## 介绍一个做过的项目

## 双端用户, 对用户和商户进行分库分表, 如果需要对数据进行聚合检索, 如何设计?

## kafka 模型
Producer:
Kafka:
Zookeeper:
ConsumerGroup: 
Consumer: 


















