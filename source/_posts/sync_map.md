title: sync.Map
date: 2021-08-16 14:10:00
tags: [GO]
categories: GO
comments: true
---

# 源码剖析
https://colobu.com/2017/07/11/dive-into-sync-Map/
源码位于 sync 包的 map.go 文件

## 非并发安全的 Map + 读写锁
````go
counter.RLock()
n := counter.m["some_key"]
counter.RUnlock()
````

````go
counter.Lock()
counter.m["some_key"]++
counter.Unlock()
````
在 Map 很长的情况下, 只有一把锁, 锁竞争太激烈, 对性能的影响是很大的


## 数据结构
````go
type Map struct {
  // 当涉及到dirty数据的操作的时候，需要使用这个锁
  mu Mutex
  

  // 一个只读的数据结构，因为只读，所以不会有读写冲突。
  // 所以从这个数据中读取总是安全的。
  // 实际上，实际也会更新这个数据的entries,如果entry是未删除的(unexpunged), 并不需要加锁。如果entry已经被删除了，需要加锁，以便更新dirty数据。
  read atomic.Value // readOnly


  // dirty数据包含当前的map包含的entries,它包含最新的entries(包括read中未删除的数据,虽有冗余，但是提升dirty字段为read的时候非常快，不用一个一个的复制，而是直接将这个数据结构作为read字段的一部分),有些数据还可能没有移动到read字段中。
  // 对于dirty的操作需要加锁，因为对它的操作可能会有读写竞争。
  // 当dirty为空的时候， 比如初始化或者刚提升完，下一次的写操作会复制read字段中未删除的数据到这个数据中。
  dirty map[interface{}]*entry


  // 当从Map中读取entry的时候，如果read中不包含这个entry,会尝试从dirty中读取，这个时候会将misses加一，
  // 当misses累积到 dirty的长度的时候， 就会将dirty提升为read,避免从dirty中miss太多次。因为操作dirty需要加锁。
  misses int
}

````