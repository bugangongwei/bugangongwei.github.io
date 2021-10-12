---
title: GMP structure
date: 2021-08-16 14:10:00
tags: [GO]
categories: GO
comments: true
---

# 概念
GOMAXPROCS: GMP 中 P 的最大数量, 默认值是 cpu 的逻辑核数;
CPU 上下文: CPU 寄存器和程序计数器;
进程: 操作系统资源分配和调度的基本单位;
线程: 操作系统执行运算调度的基本单位;
协程: coroutine/fiber 用户空间线程, 由应用程序自主控制的轻量级线程;

进程上下文切换: 用户态资源(进程的虚拟内存, 栈, 全局变量等资源)切换 和 内核态资源(上下文)切换;
线程上下文切换: 用户态资源(线程的栈) 和 内核态资源(上下文);

一个进程可以拥有多个线程, 这些线程共享进程的内存空间;

线程各自拥有约 1M 的内存空间, 并且需要切换上下文, 有一定的开销;
协程各自拥有约 1K 的内存空间, 因为是应用程序在调度, 所以不需要切换上下文;

# GO 调度器
## G-M 模型 (1.1 之前)
全局就一个 scheduler 和一把全局锁, scheduler 内维护全局唯一一个 Goroutine 可执行队列
(1) 锁竞争激烈
(2) 线程间互相传递可运行的 Goroutine, 引入大量的延迟和额外的开销;
例子: 比如当 G 中包含创建新协程的时候，M 创建了 G’，为了继续执行 G，需要把 G’交给 M’执行，也造成了很差的局部性，因为 G’和 G 是相关的，最好放在 M 上执行，而不是其他 M’。
(3) 系统调用(内核态访问文件,设备等)频繁阻塞和解除阻塞线程, 增加了额外的开销;
(4) 每个线程都需要处理内存缓存，导致大量的内存占用并影响数据局部性;
例子: 比如当 G 中包含创建新协程的时候，M 创建了 G’，并且对 G’ 进行内存缓存, 换到 M' 运行, M' 又要维护一份 G' 的内存缓存, 创建的 G' 多了, 额外的内存缓存就多了;

## G-M-P 模型 (1.1 至今)
1. 锁竞争问题
G-M 模型的性能开销大头就是锁竞争, 所以, 减少锁竞争是头号目标;
G-M-P 模型, 引入 P(Processor), 每个 P 维护了一个本地可运行 Goroutine 队列(本地 runq), 以及, 绑定了一个 M, 这样, 它就可以从本地 runq 中直接获取下一个 Goroutine 运行, 极大地减少了锁竞争, 缓解了上述问题(1);
同时, M 新建 G', G' 放入本地 runq, 之后大概率在 M 线程中运行, 不需要额外的给 M' 创建 G' 的内存缓存, 缓解了问题 (2)(4);

2. 堵塞问题
引入窃取式调度
(1) M 的本地 runq 中没有可运行的 Goroutine 了, 窃取其他 M 的本地 runq 或者全局 runq 中的任务, 平衡各个 M 的本地 runq 长度;
(2) M 被系统调用阻塞, 则解绑 MP, 把 P 给到别的 M', 等到解除阻塞时, 再分配新的 P, 减小阻塞带来的影响;

3. 为什么引入 P
在 GOMAXPROCS 个 M 全部被占用的状态下, 需要新建 M 来处理新的 G, 只要 M 不超过阈值就行;

(1) 为什么引入 P, 转变为要不要解耦 MP 的问题:
不解耦: M 堵塞, 则 cpu 被分配到 M‘, 由于 M 上面还有 runq, M‘, M'' 等都可以窃取 M 的 Goroutine;
解耦: M 堵塞, 则 P 被分配到 M‘, M‘ 运行完当前 Goroutine, 直接从 P 中取 Goroutine, 减少窃取的压力;
(2) M 是操作系统线程, 是由操作系统调度和管理的;

## 抢占式调度器 (1.2 至今)
抢占式调度解决的问题
(1) 大的 Goroutine 可能占用线程太长时间, 造成其他 Goroutine 的饥饿;
(2) 垃圾回收 STW, 会造成整个程序的停摆, 需要抢占式调度中断这个过程;

### 基于协作的抢占式调度(1.2 ~ 1.13)
基于协作的抢占式调度, 在编译阶段, 会给函数插入抢占函数, 在运行时, 调用函数的时候, 会调用抢占函数, 判断函数是否被抢占, 如果是, 则释放 M;
因为是在函数被调用时判断是否被抢占, 所以, 没有办法及时中断和释放资源, 如果遇到很长的 for 循环或者垃圾回收长时间占用线程, 则是不能及时释放资源的;

### 基于信号的抢占式调度(1.14 至今)
在程序启动时, 注册信号 SIGURG 和信号处理函数, 在特定时候会触发抢占, 发送 SIGURG 信号; 收到信号的 Goroutine 会被挂起然后退出线程;
触发信号发送时机:
(1) go 后台监控 runtime.sysmon 检测超时发送抢占信号;
(2) go 垃圾回收栈扫描时发送抢占信号;
(3) go 垃圾回收 STW 时调用 preemptall 发送抢占信号, 暂停所有的 P;

# 数据结构
1. runtime.g 结构体
重点介绍 runtime.g 中的 atomicstatus 字段, 表示 Goroutine 的当前状态
(1) 第一类 waiting
```_Gsyscall``` Goroutine 正在进行系统调用
```_Gwaiting``` Goroutine 运行时堵塞
```_Gpreempted``` Goroutine 因抢占而被堵塞
(2) 第二类 runnable ```_Grunnable```
(2) 第二类 running ```_Grunning```

2. runtime.m 结构体
重点介绍
(1) g0: 持有调度栈的特殊 g
(2) curg: 当前线程上正在运行的 g

3. runtime.p 结构体

# 调度器初始化
runtime.schedinit()
(1) 设置 maxmcount = 10000, procs = ncpu | gogetenv("GOMAXPROCS")
(2) runtime.procresize(procs)
  (2.1) 根据 procs 来初始化全局变量 allp 切片的容量
  (2.2) 创建并初始化(runtime.p.init)新的 runtime.p 结构体
  (2.3) 绑定 m[0] 与 allp[0]
  (2.4) 将 allp[0] 之外的所有 p 丢到全局空闲队列中, 状态为 ```_Pidle```

# 创建 goroutine
1. 获取或创建新的结构体 runtime.newproc1
(1) 从当前 goroutine 获取对应的 _p_;
(2) gfget() 从 _p_.gFree 或者 sched.gFree 列表中获取 runtime.g 结构体, 获取到的 runtime.g.atomicstatus 应该是 ```_Gdead```;
(3) 如果(2)为空, 调用 runtime.malg 并添加到全局的 allg 列表中, runtime.g.atomicstatus 从 ```_Gidle``` 变成 ```_Gdead```;
goroutine 运行完之后, g0 会修改它的状态为 ```_Gdead```, 清除 runtime.g 所有字段的值, 解绑 g 和 p, 然后把空的 g 结构体丢到 gFree 里面去, 所以, 创建 goroutine 的时候从 gFree 获取 g 结构体其实是复用结构体;

2. 将传入的参数移到 goroutine 栈上
runtime.memmove 将 fn 函数的所有参数拷贝到 goroutine 的栈空间

3. 更新 goroutine 调度相关的属性
runtime.g.atomicstatus = ```_Grunnable```
设置栈指针, 程序计数器

4. 将 goroutine 加入到可运行队列 runtime.runqput
(1) next == true, 处理器下一个执行的就是当前 goroutine
(2) next == false, 加入本地队列, 或者全局队列

# 触发调度的时机
(1) 主动挂起
(curg) runtime.gopark -> (g0) runtime.gopark_m
curg: 暂停当前 g
g0: runtime.g.atomicstatus ```_Grunnable``` 变成 ```_Gwaiting```, 解绑 gp, 调用 runtime.schedule 触发新一轮调度;
(2) 系统调用
(3) 协作式调度
(4) 系统监控

# 第一个 goroutine 的创建
https://cbsheng.github.io/posts/%E6%8E%A2%E7%B4%A2golang%E7%A8%8B%E5%BA%8F%E5%90%AF%E5%8A%A8%E8%BF%87%E7%A8%8B/
用 gdb 跟踪查看 go 程序的启动过程
````
$ go build main.go
$ gdb main

GNU gdb (GDB) 7.12.1
Copyright (C) 2017 Free Software Foundation, Inc.
(gdb) info files
Symbols from "/Users/cbsheng/goproject/src/test/main".
Local exec file:
  `/Users/cbsheng/goproject/src/test/main', file type mach-o-x86-64.
  Entry point: 0x1051540  # 入口点
  0x0000000001001000 - 0x000000000109352b is .text
  0x0000000001093540 - 0x00000000010d6e52 is __TEXT.__rodata
  0x00000000010d6e52 - 0x00000000010d6e52 is __TEXT.__symbol_stub1
  ...

(gdb) b *0x1051540
Breakpoint 1 at 0x1051540: file /usr/local/go/src/runtime/rt0_darwin_amd64.s, line 8.
(gdb) b runtime.rt0_go
Breakpoint 2 at 0x104dd30: file /usr/local/go/src/runtime/asm_amd64.s, line 12.
````

````
$GOPATH/src/runtime/asm_amd6.s

TEXT runtime·rt0_go<ABIInternal>(SB),NOSPLIT,$0
...
  // 初始化全局 g0 的栈数据
  // create istack out of the given (operating system) stack.
  // _cgo_init may update stackguard.
  MOVQ  $runtime·g0(SB), DI
  LEAQ  (-64*1024+104)(SP), BX
  MOVQ  BX, g_stackguard0(DI)
  MOVQ  BX, g_stackguard1(DI)
  MOVQ  BX, (g_stack+stack_lo)(DI)
  MOVQ  SP, (g_stack+stack_hi)(DI)

  // 执行文件的绝对路径初始化
  CALL  runtime·args(SB)
  // cpu个数和内存页大小初始化
  CALL  runtime·osinit(SB)
  // 命令行参数、环境变量、gc、栈空间、内存管理、所有P实例、HASH算法等初始化
  CALL  runtime·schedinit(SB)

  // 新建一个goroutine，该goroutine绑定runtime.main，放在P的本地队列，等待调度
  CALL  runtime·newproc(SB)

  // start this M
  // 启动M，开始调度goroutine
  CALL  runtime·mstart(SB)
````

实际上, 第一个 goroutine 应该是全局第一个第一个 g0
第二个 goroutine 应该是除 P0 外第一个 P 的 g0
非 g0 的第一个 goroutine 就是绑定了 runtime.main 的 g


总结:
Golang 为什么要使用 Goroutine ?
与多进程/多线程对比:
(1) 进程/线程需要占据不少的内存空间, 协程占据很小的内存空间, 所以协程的数量可以很多
(2) 基于时间片轮询的抢占式调度, 使得线程存在大量的上下文切换场景, 上下文切换需要进行内核和用户态的状态切换和数据存储, 浪费不少 cpu, 而协程自始至终都是用户态的结构体, 切换也只是在用户态;

基于协程提出了 G-M 模型, 为什么放弃了 GM 模型?
(1) 全局 G 队列, 锁竞争激烈;
(2) 局部性差, 新建的 G, 肯能要使用其他的 M 运行, 需要另外复制一份数据, 加大内存开销;
(3) 频繁的阻塞和接触阻塞, 需要消耗系统资源;

GMP 模型的优势
(1) 锁竞争: 使用本地队列
(2) 局部性差: 本地队列优先级最高
(3) 阻塞问题: hand off 机制 + 窃取式调度 + 抢占式调度

为什么引入 p ?
(1) M 是操作系统调度的, 如果用 M 来调度并运行 G, 那么需要频繁地进行系统调用, 需要进行用户态和内核态切换, 引入 p 的话, 调度只在用户态, 运行时才需要内核态的支持;
(2) GM 模型, G 和 M 高度耦和, 在发生阻塞的时候, M 的本地队列剩下的 G 需要长时间等待阻塞释放, 而引入 p 之后, GM 解耦, 可以通过 p 把 G 绑定到其他的 M 上, 避免阻塞队后面的 G 造成影响;

Go 语言为什么要引入抢占式调度?
防止 Goroutine 长时间占用 cpu, 造成其他 Goroutine 的饥饿, 典型的如垃圾回收 G, STW 的时候甚至可能需要几分钟;

基于协作的抢占式调度
在每个函数调用前插入代码来判断 Goroutine 是否需要被挂起来, 如果需要, 就执行 gopark 执行挂起函数;
如何实现? runtime.G 结构体中引入 stackguard0 字段, 这个字段被设置为 stackpreempt 时, 表示 Goroutine 需要被挂起;
优点: 对多函数调用的 Goroutine 能够及时通过超时运行, 判断 Goroutine 是否需要被挂起;
缺点: 如果 Goroutine 只有一个大的函数, 函数运行时间很长, 那么一旦判断完认为不需要挂起, 之后很长的执行时间内都不会有第二次判断, 那么这个 Goroutine 就会长时间占用 cpu;
比如 for 循环和 STW 都是需要很长的执行时间, 根本无法被协作式抢占调度挂起;

基于信号的抢占式调度
基于信号的抢占式调度在接受到信号的瞬间, 就可以调用 gopark 挂起 G, 即使是正在运行中的函数;
如何解决了基于协作的抢占式调度的问题?
(1) for 循环: sysmon 还会一如既往地监视所有 Goroutine, 发现有的 Goroutine 的运行时间超过 10ms, 那么发送抢占信号;
(2) STW: STW 或者栈扫描(写屏障的时候的STW)会对当前所有正在运行的 G 发送抢占信号;

非均匀内存访问调度器: 核心思想, 减少全局变量, 增加本地变量, 减少资源和锁的竞争;

触发调度的时机:
(1) 主动挂起: Goroutine 调用 gopark 进行主动挂起, 发生在运行超时, 垃圾回收时, sync.Cond.Wait();
(2) 阻塞的系统调用: 系统调用根据类型分为阻塞和非阻塞的, 非阻塞的调用完直接回调运行时函数, 阻塞的在调用准备阶段就会把 p 和 m 解绑, 触发重新调度;



















