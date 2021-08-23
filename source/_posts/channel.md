---
title: channel
date: 2021-08-20 00:00:00
tags: [GO]
categories: GO
comments: true
---

# 概念
1. CSP(Communicating sequential process) 通信顺序进程, 实体 - channel - 实体 对应 golang goroutine - channel - goroutine

# 数据结构
````go
type hchan struct {
  qcount uint // channel 中当前元素个数
  dataqsize uint // channel 中的循环队列(缓冲区)的长度
  buf unsafe.Pointer // channel 中循环队列(缓冲区)的地址指针
  ...
  sendx uint // 下一个被发送的元素在循环队列中的下标
  recvx uint // 下一个接受的元素在循环队列中的下标
  recvq waitq // 由于缓冲区空而阻塞的 Goroutine 列表, 元素类型 runtime.sudog
  sendq waitq // 由于缓冲区满而阻塞的 Goroutine 列表, 元素类型 runtime.sudog
}
```` 

# 发送数据
````go
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
  lock(&c.lock)

  // channel 已关闭, 则 panic
  if c.closed != 0 {
    unlock(&c.lock)
    panic(plainError("send on closed channel"))
  }

  // 1. 接受阻塞队列 recv 不为空的情况下, 直接把数据发送给阻塞的第一个 Goroutine 
  if sg := c.recvq.dequeue(); sg != nil {
    send(c, sg, ep, func() { unlock(&c.lock) }, 3)
    return true
  }

  // 2. 如果循环队列没有满, 则把数据放入循环队列
  if c.qcount < c.dataqsize {
    qp := chanbuf(c, c.sendx)
    typedmemmove(c.elemtype, qp, ep)
    c.sendx++
    if c.sendx == c.dataqsiz {
      c.sendx = 0
    }
    c.qcount++
    unlock(&c.lock)
    return true
  }

  // 3. 阻塞发送
  if !block {
    // 指定必须非阻塞发送
    unlock(&c.lock)
    return false
  }

  // 把 g 包装到 sudog
  gp := getg()
  mysg := acquireSudog()
  mysg.elem = ep
  mysg.g = gp
  mysg.c = c
  gp.waiting = mysg
  // sudog 入队列 sendq
  c.sendq.enqueue(mysg)
  // 调用 gopark 挂起当前 g, atomicstatus _Running -> _Gwaiting
  goparkunlock(&c.lock, waitReasonChanSend, traceEvGoBlockSend, 3)

  gp.waiting = nil
  gp.param = nil
  mysg.c = nil
  releaseSudog(mysg)
  return true
}

// send 函数
func send(c *hchan, sg *sudog, ep unsafe.Pointer, unlockf func(), skip int) {
  // 拿到 sg 接收 channel 值的变量的地址
  if sg.elem != nil {
    // 把 channel 值 ep 直接 copy 到 sg.elem 中
    sendDirect(c.elemtype, sg, ep)
    sg.elem = nil
  }

  gp := sg.g
  unlockf()
  ...
  // gp.aotomicstatus _Gwaiting -> _Grunable
  // Grunable 加入 gp.runnext
  goready(gp, skip+1)
}

````

# 接收数据
````go
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
  // 从空 channel 里面 recv , 调用 gopark 挂起当前的 goroutine
  if c == nil {
    if !block {
      return
    }
    gopark(nil, nil, waitReasonChanReceiveNilChan, traceEvGoStop, 2)
    throw("unreachable")
  }

  lock(&c.lock)

  // channel 被关闭且 缓冲区中不存在数据, 返回 false
  if c.closed != 0 && c.qcount == 0 {
    unlock(&c.lock)
    if ep != nil {
      typedmemclr(c.elemtype, ep)
    }
    return true, false
  }

  // 1. sendq 中存在阻塞接收的 g 时, 直接塞个阻塞队列第一个 g
  if sg := c.sendq.dequeue(); sg != nil {
    recv(c, sg, ep, func() { unlock(&c.lock) }, 3)
    return true, true
  }

  // 2. 缓冲区有数据时, 直接从缓冲区拿数据
  if c.qcount > 0 {
    qp := chanbuf(c, c.recvx)
    if ep != nil {
      typedmemmove(c.elemtype, ep, qp)
    }
    typedmemclr(c.elemtype, qp)
    c.recvx++
    if c.recvx == c.dataqsiz {
      c.recvx = 0
    }
    c.qcount--
    return true, true
  }

  // 3. 阻塞接收
  if !block {
    unlock(&c.lock)
    return false, false
  }

  gp := getg()
  mysg := acquireSudog()
  mysg.elem = ep
  gp.waiting = mysg
  mysg.g = gp
  mysg.c = c
  // 入队 recvq
  c.recvq.enqueue(mysg)
  // 挂起当前 g
  goparkunlock(&c.lock, waitReasonChanReceive, traceEvGoBlockRecv, 3)

  gp.waiting = nil
  closed := gp.param == nil
  gp.param = nil
  releaseSudog(mysg)
  return true, !closed
}

// recv 函数
func recv(c *hchan, sg *sudog, ep unsafe.Pointer, unlockf func(), skip int) {
  if c.dataqsiz == 0 {
    // 不存在缓冲区的情况, 直接接收 sendq 的第一个 g 中的 elem, 复制给当前 g 的变量
    if ep != nil {
      recvDirect(c.elemtype, sg, ep)
    }
  } else {
    // 有缓冲区的情况, 把 sendq 的第一个 g 的数据写入缓冲区
    qp := chanbuf(c, c.recvx)
    if ep != nil {
      typedmemmove(c.elemtype, ep, qp)
    }
    typedmemmove(c.elemtype, qp, sg.elem)
    c.recvx++
    c.sendx = c.recvx // c.sendx = (c.sendx+1) % c.dataqsiz
  }
  gp := sg.g
  gp.param = unsafe.Pointer(sg)
  // _Gwaiting -> _Grunable
  goready(gp, skip+1)
}

````

# 关闭 channel
````go
func closechan(c *hchan) {
  if c == nil {
    panic(plainError("close of nil channel"))
  }

  lock(&c.lock)
  if c.closed != 0 {
    unlock(&c.lock)
    panic(plainError("close of closed channel"))
  }

  c.closed = 1

  // 把所有的阻塞 goroutine 塞到 glist
  var glist gList
  for {
    sg := c.recvq.dequeue()
    if sg == nil {
      break
    }
    if sg.elem != nil {
      typedmemclr(c.elemtype, sg.elem)
      sg.elem = nil
    }
    gp := sg.g
    gp.param = nil
    glist.push(gp)
  }

  for {
    sg := c.sendq.dequeue()
    ...
  }

  // 为所有被阻塞的 goroutine 调用 runtime.goready
  for !glist.empty() {
    gp := glist.pop()
    gp.schedlink = 0
    goready(gp, 3)
  }
}
````



