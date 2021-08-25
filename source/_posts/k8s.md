---
title: k8s
date: 2021-08-23 14:10:00
tags: [k8s]
categories: k8s
comments: true
---

https://juejin.cn/post/6844904004774805512
# 组件
一个集群内有一个 master 节点(node), 和若干个 worker 节点(node);
master 节点负责管理集群, 包括的组件有: APIServer, Controller Manager, Coredns, Scheduler, etcd;
worker 节点里面负责真正地跑应用程序, 主要包括的组件有: Kubelet, kube-proxy, NodePort, Pod, Docker, Fluented;
pod 是在服务器上的一个隔离的命名空间, 类似于虚拟机, 把一些资源圈起来, 供自己使用, 与其他 pod 隔离, 一个 pod 中可以运行多个 contianer;
pod 的概念, 在我看来, 更像是 container 的更高层次的抽象, 同一个 pod 内的 container 可以通过 localhost 通信;

container 具有以下特点:
(1) 它被宿主机操作系统隔离了，它看不到宿主机上的其他进程，以为自己就是pid为1的进程（namespace）
(2) 它被宿主机操作系统限制了硬件资源，它能使用的cpu、内存等硬件资源只是宿主机的一部分（cgroup）
(3) 在被宿主机操作系统限制了存储空间，让这个进程认为宿主机的某个目录就是系统根目录（rootfs）
这几个欺骗进程的技术，就是是容器技术的三板斧，用于资源隔离的namespace，用于资源限制的cgroup,以及用于伪装进程根目录的rootfs。这三种技术，都是是早已存在于linux中的，只是docker公司比较创新的把这三种技术整合在一起，并且，提出了容器镜像及镜像仓库等概念，把这个进程及相关的依赖环境都一起打包成可分发及重复使用的镜像文件，方便容器能在不同机器之间移植。这样，在任何地方，只要能运行docker，就能把容器镜像跑成一个包含这应用进程及相关依赖环境的容器实例。



# 集群内访问
在集群内, client_pod 需要访问 server_pod, 会怎么做呢?
client_pod 拿着它持有的 service_name(可能是应用中配置的) 去访问 master 节点的 Coredns; 
Coredns 在 etcd 中维护了一个 (service_name <-> clusterIP) 的映射关系, 根据 service_name 返回给 client_pod 一个 clusterIP;
clusterIP 是 k8s 分配给 service 的虚拟 ip ; 如下面的 “192.168.244.170”
```
~ kdev describe service arist-rpc
Name:              arist-rpc
Namespace:         backend
Labels:            app=arist
                   app.kubernetes.io/instance=dev-c0-clingo-arist
                   cloud-infra/yugong-migrate-taskID=b5e88abf-9285-4179-8177-489d03927ff0
                   cloud-infra/yugong-need-migrate=true
                   component=rpc
                   team=clingo
                   version=default
Annotations:       <none>
Selector:          app=arist,component=rpc,version=default
Type:              ClusterIP
IP:                192.168.244.170
Port:              grpc  50051/TCP
TargetPort:        50051/TCP
Endpoints:         10.98.219.186:50051
Session Affinity:  None
Events:            <none>
```

client_pod 根据 clusterIP 访问到宿主机, 也就是一个 node;
node 通过 kube-proxy 组件, 去内核调用 iptables 或者 IPVS 来获取一个合适的 pod_ip, iptables 或者 IPVS 维护了一个 (clusterIP <-> [podIP0, podIP1...]) 的映射关系;
通过 kube-proxy 把请求打到这个 podIP:TargetPort 上去;


# 集群外访问
外部请求访问集群中的某个 service, 请求被 LoadBalancer 拦截, 然后发送到 node 中的 NodePort, 然后经过 iptables/IPVS 进行负载均衡和服务发现;














