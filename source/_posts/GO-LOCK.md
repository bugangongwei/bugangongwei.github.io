---
title: GO 互斥锁
date: 2021-06-26 16:36:15
tags: [GO源码解析]
categories: GO
comments: true
---

数据结构
````
type Mutex struct {
  state int32
  sema  uint32
}
````
(1) state: 表示当前互斥锁的状态信息
{% asset_img mutex_state.png mutex state 的各个位的含义 %}
(2) sema: 控制 goroutinue 阻塞与唤醒的信号量


Lock()
{% asset_img lock_sch.png 加锁过程 %}


满足自旋的条件
(1) 当前互斥锁处于正常状态 (饥饿状态下, goroutinue 全部都按照 FIFO 的顺序进入队列, 不需要自旋)
(2) 当前运行运行的机器是多核 CPU (单核 cpu 下, 本身处理速度慢, 自旋是空转 cpu, 所以更加损耗性能)
{% asset_img 1_core_cpu.png 单核 cpu 下自旋的情况 %}
(3) 至少存在一个其他正在运行的处理器 P (GOMAXPROCS > 1, GOMAXPROCS 的默认值其实就是 cpu 核数, 前面已经讲过必须是多核的)，并且它的本地运行队列(local runq)为空
(4) 当前goroutine进行自旋的次数小于4

所谓自旋, 其实就是执行 30 次 PAUSE 指令, 消耗着 cpu 进行忙等待;
