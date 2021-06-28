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
![Mutex state 每位的含义](mutex_state.png)
(2) sema: 控制 goroutinue 阻塞与唤醒的信号量
