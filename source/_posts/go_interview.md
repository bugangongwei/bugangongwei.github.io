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
ctx := context.WithTimeout(parentCtx, duration

# 
(1) Goroutine 如何调度
(2) PMG 模型描述



# 运行时

# 如何启动第一个 goroutine
