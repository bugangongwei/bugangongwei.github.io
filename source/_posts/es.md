---
title: Elasticsearch
date: 2021-08-18 00:00:00
tags: [Elasticsearch]
categories: 搜索引擎
comments: true
---

# ElasticSearch 的使用场景和优势, 劣势
## 使用场景
垂直搜索
## 优势
(1) 横向扩展性: 添加机器时, 重启即可并入集群;
(2) 分布式集群和分片机制: 通过 HashMap 进行分片, 把不同的文档路由的不同的分片;
(3) 高可用: 复制机制;
## 劣势
(1) 没有精细化的权限管理;
(2) 集群脑裂问题, 节点不一致问题仍然存在;
(3) 不支持事务;

# ElasticSearch Sniff
sniff, 嗅探机制, 只需要给客户端一个或者多个节点的 ip 就可以嗅探到整个集群的状态, 把集群中其他机器的 ip 地址加入到客户端中; 好处是不用手动把集群中所有 ip 地址都添加到客户端;
sniff 只嗅探内网 ip, 对于外网的 ES 访问, 会返回负载均衡器或反向代理服务器给到的虚假的 ip, 这样的情况下建议把 client.transport.sniff 设置为 false;

# 运作机制
## 集群部署
新建 index 的时候, 默认 node 个数为 2, primary_shard 个数为 5, replication 个数为 1;

## 添加文档
Client 随机指向一个 Node 节点, 这个 Node 节点被当作为 Coordinate Node, 作为协调节点, 处理分发和聚集等工作, Coordinate Node 根据文档 id, 通过 `hash(_doc_id)%num_of_primary_shards` 来计算文档指向的切片, 寻找到切片所在的 Node, 被称作 Data Node, 把请求转发到这个 Node 的响应分片去;
文档写入切片的缓冲区 MemoryBuffer, 同时写入 TransLog, 等到缓冲区文档数量到达一定的限制或者缓冲区留存时间到达一定的阈值的时候, 进行 Refresh 操作;
Refresh 的具体操作是: 新建一个 segment, 把文档从缓冲区中写入 segment 的内存中, 然后把 segment 记录到 commit points 中, 这个 commit point 只记录可查询的 segment, 让 segment 内存中的数据可以查询;
因为 Refresh 的间隔时间默认是 1s, 所以经常说道的 ElasticSearch 的 NRT(Near RealTime) 说的就是, 在一秒的时间后, 新插入的文档就能被查询到, 这是一个近乎实时的查询;
当 translog 足够大或者到达一定的时间的时候, 就该把 segment 数据写入磁盘中了, 确保写入成功以后, 会把 translog 清空, 这个过程叫做 flush;

## 查询文档
Client 随机指向一个 Node 节点, 这个 Node 节点被当作为 Coordinate Node, 作为协调节点, 处理分发和聚集等工作, Coordinate Node 把请求分发到所有的切片中, 到达某个切片后, 从 commit points 查找可查询的 segment, 然后过滤掉 .del 文件中的数据, 返回 from + size 大小的数据, 并且按照词频排序, 所有的切片都把数据返还给 Coordinate Node, Coordinate Node 会将数据进行聚集, 按照单词频率等因素进行排序, 根据 from and size 取一部分数据, 然后去各个切片取对应的文档数据, 组装后返还给客户端;

## 删除文档
`commit poin` 维护了一个 `.del` 文件, 删除文档就把文档添加到 `.del` 文件中, 查询完毕后, 通过 `.del` 过滤被删除文档然后返回给客户端;

## 更新文档
更新分为删除和新增两个动作;
旧版本的数据被加入到 .del 文件, 新版本的数据添加到新的 segment 中;

## segment 合并
segment 合并过程会过滤掉 .del 文件, 合并完会删除 .del 文件, 缩小了 segment 的最终体积;

## 倒排索引和优点
倒排索引是可以根据 value 定位到 key, 在 ES 中一般是通过分词后的单词来定位关联的文档 id;
倒排索引是不可更新的, 以为每次更新都要更新大量的数据, 每个单词涉及到的文本可能很多, 更新一个文档可能涉及到很多单词;
不更新的好处:
(1) 只读, 没有资源竞争和锁竞争
(2) 索引维护在内存中, 没有磁盘 io 开销;
(3) 可以进行数据压缩, 减轻内存和 IO 压力; 

## 倒排索引原理
Term Index + Term Dictinary
Term Index 是一个以前缀为索引, 提供一个可能的 Term 定位的数据结构, 例如新华字典的目录索引, 使用的数据结构是具有 Trie 前缀树改进的 FST(Finite-state Transducer) 有限状态机转换器, 和 Trie 树形结构不同, FST 会将后缀相同的 Term 的后缀合起来, 这样就会减少一些内存占用;
https://www.shenyanchao.cn/blog/2018/12/04/lucene-fst/

所以倒排索引为什么快?
(1) Term Index : Trie 比 HashMap 更省内存, 因为只保存了前缀, FST 比 Trie 更省内存, 因为后缀相同的被合并保存了;
(2) Term Index 是放在内存中的, 通过前缀匹配找到 Term 可能在的 Block 块, 再去磁盘 Term Dictinary 上找 Term, 大大减少了磁盘随机 IO 的次数;









