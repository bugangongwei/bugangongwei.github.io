---
title: kafka
date: 2021-08-16 14:10:00
tags: [kafka]
categories: kafka
comments: true
---

# 概念

# kafka 要解决的问题
## 消息不丢失
(1) 消息持久化, 使用文件存储对消息进行持久化
(2) 消息备份, leader 服务器宕机后, 采用选举机制, 把其中一个 follower 选举成 leader, 还可以继续使用
(3) 既然(2)需要 leader-follower, 就需要集群, 集群需要对服务器进行注册和发现, 就需要一个注册中心 zookeeper
(4) 既然(3)进行了服务注册和发现, 那么用户就需要负载均衡, 负载均衡是根据 HashMap 来的,根据 HashMap 的结果选择不同的分区写消息
HashMap 的原理: hashCode & (slotLength - 1), hashCode 是根据 key 生成的一个整数, slotLength 表示分区/服务器数量
redis 底层数据存储也适用的 HashMap 原理

## 高吞吐率
(1) 顺序写磁盘: producer 采用 push 的模式将消息发送到 brokers, 每条消息都被 append 到 partition 中, 通过顺序写磁盘(磁盘寻址很慢, 但是顺序追加写不需要寻址所以还算快)保障了生产消息的高吞吐率;
(2) kafka 零复制: kafka 的文件内容拷贝, 使用了 OS 层的 sendFile 函数, 直接讲 pageCache 的内容复制到目标文件磁盘位置, 减少了大量的复制操作;
(3) kafka 分段日志: kafka 的每个 partition 的日志, 以 offset start 值为文件名, 在查找指定位置的 offset 的时候, 可以直接根据文件名定位 offset 所在的文件;
(4) kafka 预读(Read Ahead)后写(Write Behind), 预读会在读取数据的时候, 把相邻的数据预先读到缓存中, 后写会直接在应用层把数据写到系统层的 pageCache 里面, 后续由系统层把 pageCache 写入到磁盘, 省略了一些复制操作, 同时系统层对系统层写磁盘比应用层对系统层写磁盘要快;
(5) 批处理: 批量写, 批量读
(6) 压缩消息: cfg.Producer.Compression 消息压缩算法

## kafka 的 offset 存放在哪里
(1) 0.9 版本前, 存放在 zookeeper 中, 但是因为 offset 经常变动, zookeeper 职责应该是协调调度, 所以其实不合适
(2) 0.9 版本后, 存放在 kafka cluster 中
(3) 也可以设置 consumer 的消费回馈机制为异步提交或同步提交, 决定是否要自定义存储 offset 避免重复消费

## kafka 消费数据使用的推还是拉的方式
kafka 消费数据使用的是拉的方式, 让消费的速率由消费者来掌控

## kafka 如何防止丢数据
(1) producer 同步发送数据
(2) producer 应答机制 ACK = -1, 保证所有副本都同步完成才反馈

## kafka 如何避免重复消费
自己维护 offset, 可以存放在 redis 里面用 zset 去重

## kafka 如何保证不同的订阅源消费到同样的数据
HW, LEO, 木桶理论, follower 同步 leader 新数据的时候, 如果拿到数据并成功反馈到 leader, 则 follower 的 LEO 更新成和 leader 一样, HW 也会更新成 LEO, 并且通知 follower 也把 HW 更新, 如果 follower 拿到数据之前就发现 leader 崩溃了, leader 成为新的 follower 的 LEO 和老的 leader 的 LEO 不一样, 没关系, 用户读取数据, 关心的是 HW 的值, 读到的数据还是 HW, 而丢失的 LEO, producer 可以重发;

## kafka 的 leader 选举机制
(1) /controller: controller 负责 leader 选举和 rebalance, 是集群中的其中一个担任重要任务的 broker, controller 是如何选举出来的呢? 是在创建 broker 的时候, 谁先创建出来, 发送消息到 zookeeper, 谁就被 zookeeper 设置为 controller;
(2) controller 通过 ISR(正在同步的副本) 通过顺序轮询来确定下一个 leader, 如果 follower 挂了, 就选下一个 follower, 如果 followers 全部挂了, 就只能阻塞等待新的副本起来;

## ISR 的 brokerid 什么时候会消失?
(1) broker down 掉
(2) 网络堵塞
(3) replica.lag.max.ms, 超过这个时间, follower 没有发送 fetch 请求, leader 会把它踢除;
(4) replica.lag.max.messages(版本 0.9 之前有这个参数)

## 如何提高 kafka 消费速度
(1) 增加分区和消费者数量
(2) 增加批量拉取数据的大小
(3) 增大批处理消息的大小

## 使用消息队列的好处
(1) 解耦了生产者, 消费者和数据存储, 耦合性越低, 扩展性越强
(2) 数据冗余在多台机器中, 一台服务宕机, 还有其他服务提供相同的数据, 提高了程序的健壮性和数据的安全性
(3) 扩展性, 在分布式的架构中, 服务器不够用可以灵活添加
(4) 保证了峰值处理能力, 生产者和消费者可以按照自己的节奏来写和读数据, 就算速率变快或变慢, 都不影响消息存储, 消息短时间内不会消失
(5) 可恢复性, 某个服务器宕机, 其他副本可以提供服务
(7) 保证顺序, 同一个 partition 里面的数据顺序能够保证 (如何保证不同分区的数据的顺序???)
(8) 缓冲, 提高效率
(9) 异步通信, 只需要保证消息发送出去了, 不需要保证被消费了

## kafka 生产者的应答机制(ACK)
(1) ACK = 0: 生产者发送完数据, 不关心数据是否送达, 直接发送下一条数据
(2) ACK = 1: 生产者发送完数据, 需要等 leader 应答, 应答完才发送下一条数据
(3) ACK = -1: 生产者发送完数据, 需要等到所有副本(leader+followers)应答, 才发送下一条数据

## 为什么使用 partition
(1) 可根据数据大小进行扩展和收缩, 提高负载能力, partition 可以根据机器的性能调整自身, 而通过 partition 的扩大和缩小可以扩大和缩小整个 topic 的可承载数据量
(2) 可以提高并发, 因为可以以 partition 为单位并发读写

## partition 分区的原则
(1) 指定了 partition 就直接用
(2) 未指定 partition 但是指定了一个 key, 通过 key 的 hash 值计算 partition
(3) partition 和 key 都没指定, 通过轮询选出一个 partition

## kafka 如何保证写入顺序
kafka producer 的批量数据使用 DoubleQueue 进行缓存, 如果某次发送失败, 那数据会重新写回到 DoubleQueue 里面, 一个 DoubleQueue 对应一个分区

## kafka 数据一致性
HW: High Waterline
LEO: Log End Offset

## kafka 删除旧数据的策略
(1) log.retention.hours = 168 // 时间 7 天
(2) log.retention.bytes = 1073741824 // 数据量 1 G

## kafka zookeeper 里面存储的数据
(1) cluster: 集群元信息, 集群 id
(2) controller: 集群控制器信息, controller 负责 rebalance 等操作
(3) controller-epoch: 当前选举的版本号
(4) brokers: 集群中机器的信息, 包括 broker 的 host, port 等
(5) consumers: 消费者组, 消费者

## kafka 是如何存储数据的
(1) topic-partition 目录
(2) xxxxxxxxxxxxxxxx.log: 存储实际的消息, 二分查找 offset 所在的 log 文件
(3) xxxxxxxxxxxxxxxx.index: 存储 offset 的下标, 索引

## sarama 的 kafka 配置参数
### 消费者
``` kafka-console-consumer.sh --zookeeper [zookeeper地址] --topic [topic那么] --from-begining```
#### config.Net.SASL.Mechanism(常用款)
(1) plain_text: 用户名密码都以 base64 字符串格式通过网络传输给 server 端进行验证;
(2) digest-md5: Client/Server 各自保存一个约定好的隐形密码, 服务器提出 challenge 给到客户端, 客户端根据 challenge 与隐形密码计算得到 response, 返回给 Server 端, 如果与 Server 端计算得到的结果一样, 则验证通过;
#### config.Consumer.Offsets.Initial
(1) OffsetNewest: 启动消费者时, 从最近一条开始消费
(2) OffsetNewest: 启动消费时, 从最早一条开始消费, `from-begining`
### 生产者
#### cfg.Producer.RequiredAck
(1) NoResponse - ACK = 0
(2) WaitForLocal - ACK = 1
(3) WaitForAll - ACK = -1
#### cfg.Producer.Compression 消息压缩算法
#### cfg.Producer.Flush.Frequency 消息批量发送的频率
#### cfg.Producer.Partitioner 分区规则

## 其他
(1) kafka 日志文件目录命名: topic-partition
(2) kafka .index 文件保存了所有 offset 在同名 .log 文件中的偏移量, 也就是在文件中第几个字节处; 如 0000.index 中的 `1 0` 表示偏移量 1 在 0000.log 文件的第 0 个字节处;


