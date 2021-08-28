---
title: GMP structure
date: 2021-08-16 14:10:00
tags: [GO, interview]
categories: GO
comments: true
---


# context
context 作用:
(1) Deadline()(deadline time.Time, ok bool): 用来传递请求的截止时间, DeadlineExceeded
(2) Done()<-chan struct: 用来传递取消信号, Canceled
(3) Value(key interface())interface{}: 用来传递请求参数, 比如 trace_id, user_id

context 的原理:
拥有同一个上下文或拥有父子上下文的 Goroutine 会订阅同一个 ctx.Done() channel, 一旦接收到取消信号, 就会立即停止手中的工作;

context 超时控制:
ctx := context.WithTimeout(parentCtx, duration)

# GMP
(1) Goroutine 如何调度
通过 GMP 模型调度
(2) PMG 模型描述
G-Goroutine 
M-线程
P-处理器
每个 P 绑定一个 M, 拥有一个本地待运行的 Goroutine 队列, 和一个用于调度的 G0
GMP 模型由一组 P 和可以自由绑定在 P 上的 M, 一组用于调度的 P0 和 M0, 以及一个全局的待运行 Goroutine 队列;
程序启动时, 会先调用 initSched 来初始化调度器, 这个函数除了各种资源的准备之外, 会生成 GOMAXPROCS 个 P 和 M, 其中包括 P0 和 M0;
然后初始化全局的 G0 的栈信息, goexist 入栈;
通过 newproc 创建第一个运行代码的 G, 即 main 函数所在的 G;
然后调用 runtime.startm 初始化线程然后调用 runtime.schedule 进入调度循环;
调度循环的关键是: 当一个 Goroutine 运行完成, 会出栈它的 goexist, 然后会交给 G0 的 goexist0 进行处理, G0 不仅回收了运行完毕的 G 的资源, 还会调用 runtime.schedule 重新调度;

# struct{} 为什么不占用内存
内存在分配 struct 类型的内存时, 会判断 size, 如果 size == 0 , 则返回一个全局 struct{} (runtime.zerobase, 8 字节) 的指针地址, 所以使用 struct{}{} 的时候, 其实不会再分配内存;

# sync.Map 原理
sync.Map: 并发安全的 map
https://colobu.com/2017/07/11/dive-into-sync-Map/
(1) read 用来读, 更新和删除; dirty 用来写, 空间换时间, 避免读写冲突;
(2) 动态调整, 在 miss == len(dirty) 的时候, 把 dirty 提升为 read;
(3) 延迟删除, 删除数据只打标记, 只有在 dirty 提升的时候才真正删除数据;

# sync.Map 的操作
entry 中的 p 指向用户值, p 有三种类型的值:
(1) nil: 实际删除, m.dirty 为 nil;
(2) expunged: 标记删除, m.dirty 不为空, entry 不存在于 m.dirty 中;
(3) 正常值

Load: 
(1) 优先从 read 读, 如果 read 中没有且 read.amended == true, 则去 dirty 找; 
(2) m.missed++: 如果 missed == len(dirty), 触发 dirty 提升, 然后清空 dirty 和 missed;
(3) 过滤掉 nil 和 expunged 状态的 entry;

Store: 
(1) m.read 有 key, 则只要不是 expunged 的状态, 就用 CAS 原子更新;
(2) 如果 entry 是 expunged 状态, 先删除, 再更新 m.dirty;
(3) 如果 m.ready 没有, 但 m.dirty 有的, 则直接更新 m.dirty;
(4) 剩下的情况, 直接插入 m.dirty;

Delete:
(1) m.dirty 有, m.read 没有的 entry 直接删除;
(2) m.read 有胆 m.dirty 没有的, 标记删除;

Range:
(1) 如果 m.read 和 m.dirty 数据不一致, 则提升 dirty;
(2) for 循环遍历元素;


# channel
## channel
channel 阻塞:
 (1) 阻塞发送: recvq 为空, 缓冲区已满的条件下, 会阻塞发送; 把 g 包装成 runtime.sudog, 然后把 sudog 入队列 sendq, 然后挂起当前 g, 让出 m;
 (2) 阻塞接收: sendq 为空, 缓冲区为空的条件下, 会阻塞接收; 把 g 包装成 runtime.sudog, 然后把 sudog 入队列 recvq, 然后挂起当前 g, 让出 m;
chanel 关闭: 
channel.closed = 1, 把所有的 recvq 和 sendq 中的 g 加入到新结构 glist 中, 然后遍历把所有的 g 都调用 runtime.goready 唤醒;

## channle 的读写
读: (1) 从阻塞队列读 (2) 从缓冲区读 (3) 阻塞读
写: (1) 直接写阻塞队列 (2) 写缓冲区 (3) 阻塞写

## 给已经关闭的 channel 进行读写会发生什么
写: 直接 panic
读: 如果缓冲区没有值了, 就直接返回; 如果缓冲区还有值, chanrecv 继续往下走, 先取阻塞队列(channel 关闭会清空阻塞队列, 所以这一步基本跳过), 再取缓冲区, 读到数据;

# 内存分配
go 语言的内存分配使用了多级缓存+空闲链表分配+二维稀疏矩阵
多级缓存: mcache + mcentral + mheap;
空闲链表分配: 内存中需维护一个存储内存地址空间地址的链表, 内存分配时需要遍历链表找到合适的内存块进行分配; 为了减少链表遍历的性能损耗, go 语言的空闲链表适应采用隔离适应策略, 按照对象大小堆链表进行隔离, 不同大小的对象去对应的链表查询可分配的空闲内存, 互不干扰;
二维稀疏矩阵: 
golang 的 mheap 是内存管理的重要结构, 其中维护了一个二维数组, 数组长度因操作系统的差异而不同, 数组元素为 heapArena, 每个 heapArena 都对应了一个 page 列表, page 和操作系统的页相似, 只不过是 golang 自己对内存的一个页定义, 每页大小为 8K;

golang 为对象分配内存时:
获取对象大小级别, 比如 4B, 8B, 16B, 32K..., (0, 16B)微对象, [16B,32K]小对象, (32K, ...)大对象
(1) 微对象: 微对象分配器, 从空闲内存块中线性地分配内存, 不够时从 mcache, mcentral 中获取空闲内存块;
(2) 小对象:
    mcache 有空间: mcache.alloc 数组根据 spanClass 获取 mspan 链表头节点, 然后遍历链表, 定位到某个 mspan, 然后根据 allocBits 找到空闲地址空间, 然后找到大于当前对象真实 size 的内存空间并分配出去, 更新 mspan.allocBits 和 mspan.allocCache;
    mcache 已经满了: 此时只能拿着 spanClass 去找 mheap.central 数组中对应的 mcentral, mcentral 这样做:
      a. 从已经清理过的, 有空闲地址空间的 spanSet 中尝试获取空闲 mspan, 如果没有, next;
      b. 从还没清理过的, 有空闲地址空间的 spanSet 中尝试获取空闲 mspan, 如果没有, next;
      c. 从还没清理过的, 没有空闲地址空间的 spanSet 获取 mspan 并清理这个 spanSet;
      d. 从 mheap 中申请新的 mspan;
      e: 把找到的 mspan 的 allocBits 和 allocCache 更新一下;
(3) 直接去 mheap 中申请: mheap 会计算该对象需要的页数, 分配 8KB 的倍数的内存;


# 编译阶段
## 栈逃逸
什么是栈逃逸: 一个子程序分配一个对象并返回这个对象的指针, 该对象可以在程序中被访问的地方无法确定, 这个指针就逃逸了; 如果指针存储在全局变量中或者其他堆中的变量中, 这个指针可以被子程序之外的其他程序访问到, 这个指针也发生了逃逸;
总结: 栈逃逸是指子程序的指针变量不再栈内, 而在堆中, 可以被其他的子程序访问到;
逃逸的指针被称作<悬挂指针>, 指针的值在堆中, 而指针指向的对象在栈中, 栈随着程序或者函数的结束而清空, 此时指针指向一个没有对象的地址, 所以叫做悬挂指针;

## 逃逸分析
在编译器优化中, 逃逸分析是指分析指针的动态范围的方法;

## golang 逃逸分析
逃逸分析的两个不变性:
(1) 指向栈对象的指针不能存在于堆中；
(2) 指向栈对象的指针不能在栈对象回收后存活；

静态分析: 根据抽象语法数分析静态的数据流, 找到违反两个不变性的地方
静态分析过程:
(1) 构建带权重的有向图-对象分配图, 节点为被分配的变量, 边表示变量之间的分配关系, 权重表示寻址和取址的次数;
(2) 遍历对象分配图并查找违反两条不变性的变量分配关系, 如果堆上的变量指向了栈上的变量, 则该变量需要分配在堆上;
(3) 记录从函数的调用参数到堆以及返回值的数据流，增强函数参数的逃逸分析；

## go run -gcflags "-m -l"
-gcflags 表示后面带的参数可以传入编译器
-m 打印逃逸分析信息
-l 禁止内联编译(就是把一个函数展开来)

## 什么情况下会发生逃逸?
(1) 函数内新建的指针作为返回值, 该指针一定会发生逃逸
(2) 被已经逃逸的变量引用的指针, 一定会发生逃逸
(3) 被指针类型的slice、map 和 chan 引用的指针一定发生逃逸

## golang 逃逸分析举例
````go
func main() {
  a := S(0)
  b := make([]*S, 2)
  b[0] = &a
  c := new(S)
  b[1] = c
}

# command-line-arguments
main/main.go:160:2: moved to heap: a
main/main.go:161:11: make([]*S, 2) does not escape
main/main.go:163:10: new(S) escapes to heap
main/main.go:165:11: make([]*S, 2) does not escape
````

## 为什么指针 channel 比值 channel 慢了 30% ?
根据逃逸发生情况(3), 指针 channel 会发生逃逸, 指针在堆中, 增加 gc 的负担, 而值 channel 在栈中回收, 速度更快;

## 内联编译
在编译过程中把一些小的函数(AST 树节点小于 80 的函数)展开来, 而不使用函数调用
(1) 优点: 节省一部分函数调用的固定开销, 如栈分配, 栈溢出检查, 寄存器等资源准备;
(2) 对于可以被多个地方访问的函数, 不建议使用内联编译;
(3) go build -gcflags="-m -m" main.go
(4) 有一些特定的情况, 不允许内联, 比如存在 for 循环, select, defer, go 等关键词等...
(5) 内联树(inline tree), go build -gcflags "-d pctab=pctoinline" main.go 可以查看内联树, 内联树保存了函数调用的堆栈信息;






