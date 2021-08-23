---
title: GMP structure
date: 2021-08-16 14:10:00
tags: [GO]
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


# channel
channel 阻塞:
 (1) 阻塞发送: recvq 为空, 缓冲区已满的条件下, 会阻塞发送; 把 g 包装成 runtime.sudog, 然后把 sudog 入队列 sendq, 然后挂起当前 g, 让出 m;
 (2) 阻塞接收: sendq 为空, 缓冲区为空的条件下, 会阻塞接收; 把 g 包装成 runtime.sudog, 然后把 sudog 入队列 recvq, 然后挂起当前 g, 让出 m;
chanel 关闭: 
channel.closed = 1, 把所有的 recvq 和 sendq 中的 g 加入到新结构 glist 中, 然后遍历把所有的 g 都调用 runtime.goready 唤醒;

# 内存分配