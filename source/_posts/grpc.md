---
title: grpc
date: 2021-10-01 00:00:00
tags: [grpc]
categories: grpc
comments: true
---

# 服务端
## grpc.Server 配置初始化
lis：监听地址列表。
opts：服务选项，这块包含 Credentials、Interceptor 以及一些基础配置。
conns：客户端连接句柄列表。
m：服务信息映射。
quit：退出信号。
done：完成信号。
czData：用于存储 ClientConn，addrConn 和 Server 的 channelz 相关数据。
cv：当优雅退出时，会等待这个信号量，直到所有 RPC 请求都处理并断开才会继续处理。
serverWorkerChannels: 消息缓冲区;

````go
func NewServer(opt ...ServerOption) *Server {
  // ....
  s := &Server{
    lis:    make(map[net.Listener]bool),
    opts:   opts,
    conns:  make(map[transport.ServerTransport]bool),
    m:      make(map[string]*service),
    quit:   grpcsync.NewEvent(),
    done:   grpcsync.NewEvent(),
    czData: new(channelzData),
  }
  // ....
  s.cv = sync.NewCond(&s.mu)
  // ....
  // 初始化 serverWorkerChannels, 开启 numServerWorkers 个 channel, 每个 channel 收到消息之后进行真正的方法调用
  if s.opts.numServerWorkers > 0 {
    s.initServerWorkers()
  }
  // ....
  return s
}
````

真正的方法调用: 当监听到数据之后, 服务器会把数据写入 serverWorkerChannels 中, 服务器监听 channel, 一旦接受到值, 就进行如下的处理
````
func (s *Server) handleStream(t transport.ServerTransport, stream *transport.Stream, trInfo *traceInfo) {
  // ....
  // 通过 service name 获取 service 的具体信息
  srv, knownService := s.m[service]
  if knownService {
    // 调用对应的 unary 方法
    if md, ok := srv.md[method]; ok {
      s.processUnaryRPC(t, stream, srv, md, trInfo)
      return
    }
    // 调用对应的 stream 方法
    if sd, ok := srv.sd[method]; ok {
      s.processStreamingRPC(t, stream, srv, sd, trInfo)
      return
    }
  }
  // ....
}
````

(2) 服务注册
ServiceName：服务名称
HandlerType：服务接口，用于检查用户提供的实现是否满足接口要求
Methods：一元方法集，注意结构内的 Handler 方法，其对应最终的 RPC 处理方法，在执行 RPC 方法的阶段会使用。
Streams：流式方法集
Metadata：元数据，是一个描述数据属性的东西。在这里主要是描述 SearchServiceServer 服务
m：服务信息映射。

````go
func (s *Server) register(sd *ServiceDesc, ss interface{}) {
  // ....
  srv := &service{
    server: ss,
    md:     make(map[string]*MethodDesc),
    sd:     make(map[string]*StreamDesc),
    mdata:  sd.Metadata,
  }
  // unary 方法
  for i := range sd.Methods {
    d := &sd.Methods[i]
    srv.md[d.MethodName] = d
  }
  // stream 方法
  for i := range sd.Streams {
    d := &sd.Streams[i]
    srv.sd[d.StreamName] = d
  }
  // 注册
  s.m[sd.ServiceName] = srv
}
````


(3) 监听
lis.Accept 是 net 包的方法, 从 Socket 中监听 accept 类型的事件, 一旦有连接到达, 为这个连接建立相应的 fn 句柄, 监听句柄然后读取读缓冲区中的内容;
如果 err 不为空, 那么在[5ms, 1s]区间内使用指数退避方法, 在达到 1s 之后不变动的方式, 不断地重试;
如果 err 为空, 那么开启一个 goroutine 对获取的数据进行处理, 处理过程如下:
首先给处理过程设置 deadline;
然后把数据包装成 HTTP2Transport;
新建一个 goroutine 异步设置 HTTP2 流处理函数, 处理完之后断开连接, 移除这个句柄;
继续监听下一个请求;

HTTP2 流处理:
通过 roundRobin(轮询方式), 把数据放入 serverWorkerChannels 中;

一个监听器, 监听这个机器的某个端口下的 TCP 连接
````go
listen, err := net.Listen("tcp", s.cfg.ServerHost)

````

````

````

````
err := s.server.Serve(listen)
````

````go
func (s *Server) Serve(lis net.Listener) error {
  // 阻塞睡眠时间, 当 Socket 里面没有连接进来的时候, 服务器睡眠等待连接进来
  var tempDelay time.Duration // how long to sleep on accept failure

  for {
    rawConn, err := lis.Accept()
    if err != nil {
      // 没有连接进来
      if ne, ok := err.(interface {
        Temporary() bool
      }); ok && ne.Temporary() {
        // [5ms, 1s] 区间内使用指数退避算法来计算下一轮休眠时间
        if tempDelay == 0 {
          tempDelay = 5 * time.Millisecond
        } else {
          tempDelay *= 2
        }
        if max := 1 * time.Second; tempDelay > max {
          tempDelay = max
        }
        // 进入休眠等待
        timer := time.NewTimer(tempDelay)
        select {
        case <-timer.C:
        case <-s.quit.Done():
          timer.Stop()
          return nil
        }
        continue
      }
      // ....
    }
    // 有连接进来的时候, 首先重置休眠时间
    tempDelay = 0
    s.serveWG.Add(1)
    go func() {
      // 处理连接
      s.handleRawConn(rawConn)
      s.serveWG.Done()
    }()
  }
}
````

处理每一个具体的连接
````go
func (s *Server) handleRawConn(rawConn net.Conn) {
  // ....
  // Finish handshaking (HTTP2)
  st := s.newHTTP2Transport(conn, authInfo)
  if st == nil {
    return
  }

  rawConn.SetDeadline(time.Time{})
  if !s.addConn(st) {
    return
  }
  go func() {
    // 开始处理 HTTP2 的数据
    s.serveStreams(st)
    s.removeConn(st)
  }()
}
````

````go
func (s *Server) serveStreams(st transport.ServerTransport) {
  defer st.Close()
  var wg sync.WaitGroup

  var roundRobinCounter uint32
  st.HandleStreams(func(stream *transport.Stream) {
    wg.Add(1)
    if s.opts.numServerWorkers > 0 {
      data := &serverWorkerData{st: st, wg: &wg, stream: stream}
      select {
      // 把待处理数据放入 channel 中
      case s.serverWorkerChannels[atomic.AddUint32(&roundRobinCounter, 1)%s.opts.numServerWorkers] <- data:
      default:
        // If all stream workers are busy, fallback to the default code path.
        go func() {
          s.handleStream(st, stream, s.traceInfo(st, stream))
          wg.Done()
        }()
      }
    } else {
      go func() {
        defer wg.Done()
        s.handleStream(st, stream, s.traceInfo(st, stream))
      }()
    }
  }, func(ctx context.Context, method string) context.Context {
    if !EnableTracing {
      return ctx
    }
    tr := trace.New("grpc.Recv."+methodFamily(method), method)
    return trace.NewContext(ctx, tr)
  })
  wg.Wait()
}
````


# 客户端
(1) 初始化客户端
````go
func DialContext(ctx context.Context, target string, opts ...DialOption) (conn *ClientConn, err error) {
  cc := &ClientConn{
    target:            target,
    csMgr:             &connectivityStateManager{},
    conns:             make(map[*addrConn]struct{}),
    dopts:             defaultDialOptions(),
    blockingpicker:    newPickerWrapper(),
    czData:            new(channelzData),
    firstResolveEvent: grpcsync.NewEvent(),
  }
  // ....
  // 控制客户端向服务端询问健康情况的时间间隔
  cc.mkp = cc.dopts.copts.KeepaliveParams

  if cc.dopts.copts.Dialer == nil {
    // TCP 连接
    cc.dopts.copts.Dialer = func(ctx context.Context, addr string) (net.Conn, error) {
      network, addr := parseDialTarget(addr)
      return (&net.Dialer{}).DialContext(ctx, network, addr)
    }
    // HTTPHandShake
    if cc.dopts.withProxy {
      cc.dopts.copts.Dialer = newProxyDialer(cc.dopts.copts.Dialer)
    }
  }
  // ....
  // Determine the resolver to use.
  // Resolver: 控制如何根据传入的 "host:port" 配置获取服务端所有的 ip 地址
  // Resolver: 策略模式, 注册和根据 Schema 获取, 默认是 passthroughResolver
  if resolverBuilder == nil {
    cc.parsedTarget = resolver.Target{
      Scheme:   resolver.GetDefaultScheme(),
      Endpoint: target,
    }
    resolverBuilder = cc.getResolver(cc.parsedTarget.Scheme)
    if resolverBuilder == nil {
      return nil, fmt.Errorf("could not get resolver for default scheme: %q", cc.parsedTarget.Scheme)
    }
  }

  // Balancer: 负载均衡器, 选择发送的目标 ip
  cc.balancerBuildOpts = balancer.BuildOptions{
    DialCreds:        credsClone,
    CredsBundle:      cc.dopts.copts.CredsBundle,
    Dialer:           cc.dopts.copts.Dialer,
    ChannelzParentID: cc.channelzID,
    Target:           cc.parsedTarget,
  }

  // Wrapper: 装饰器模式, 把 Resolver 装饰成 Wrapper, 是用户对 Resolver 的使用变成透明的
  // Build the resolver.
  // 这里 newCCResolverWrapper 内部会调用 passthroughBuilder.Build(...) 方法, 会发起对服务端的异步调用
  // passthroughBuilder 是 grpc 默认的负载均衡策略, 即, 不做负载均衡, 直接返回 target
  // https://pandaychen.github.io/2019/11/11/GRPC-RESOLVER-DEEP-ANALYSIS/
  rWrapper, err := newCCResolverWrapper(cc, resolverBuilder)
  if err != nil {
    return nil, fmt.Errorf("failed to build resolver: %v", err)
  }
  cc.mu.Lock()
  cc.resolverWrapper = rWrapper
  cc.mu.Unlock()

  // A blocking dial blocks until the clientConn is ready.
  // 阻塞状态下, 阻塞地等待客户端返回, 连接处于 Ready 状态
  // 非阻塞情况下, 异步发起连接之后, 不关心客户端有没有收到
  if cc.dopts.block {
    for {
      s := cc.GetState()
      if s == connectivity.Ready {
        break
      } else if cc.dopts.copts.FailOnNonTempDialError && s == connectivity.TransientFailure {
        if err = cc.connectionError(); err != nil {
          terr, ok := err.(interface {
            Temporary() bool
          })
          if ok && !terr.Temporary() {
            return nil, err
          }
        }
      }
      if !cc.WaitForStateChange(ctx, s) {
        // ctx got timeout or canceled.
        if err = cc.connectionError(); err != nil && cc.dopts.returnLastError {
          return nil, err
        }
        return nil, ctx.Err()
      }
    }
  }

  return cc, nil
}
````



(2) 函数调用
````go
newClientStream
SendMsg
RecvMsg
````




# 服务端



