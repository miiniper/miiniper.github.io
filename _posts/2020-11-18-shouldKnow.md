---
layout:     post
title:      随便写写
subtitle:   搬砖三年有感
date:       2020-11-18
author:     miiniper
header-img: img/bg-mysql.jpg
catalog: true
tags:
    - 总结
    - 运维
---

[TOC]

# 知识点

## linux基础

[linux命令手册](https://man7.org/linux/man-pages/dir_all_alphabetic.html)

### 关于cpu,mem，io

#### cpu



```
cat /proc/stat
user	用户态时间
nice	用户态时间(低优先级，nice>0)
system	内核态时间
idle	空闲时间
iowait	I/O等待时间（不可靠）
irq	硬中断
softirq	软中断
steal 虚拟cpu时间

//    user, nice ,system idle, iowait ,irq ,softirq,steal,guest,guest_nice 
cpu  759116721 3449445 1149196339 101780263681 288147799 0 623024 205246155 0 0

单位时间内 cpu_use = (1-idle/all) *100%
```

#### mem

```
/proc/meminfo
MemTotal:        6291456 kB
MemFree:          948132 kB
MemAvailable:     948132 kB
Buffers:               0 kB
Cached:          2117748 kB
SwapCached:            0 kB
```

#### io

[说明](https://www.kernel.org/doc/Documentation/iostats.txt)，[计算](https://www.cnblogs.com/yuyue2014/p/5602800.html)

```
cat /proc/diskstats
 253       0 vda 4321981 166 930703352 14350799 759195807 419642751 85224067346 2235130173 0 211640386 2113831178 0 0 0 0
```







### cgroup，namespace, UnionFS

[参考](https://juejin.im/post/6844904078238040078)

#### Namespace

```
/proc/[pid]/ns/
```

linux内核提供的资源隔离方案，Namespaces之间的资源相互独立。目前Linux中提供8种namespace

[官方说明](https://man7.org/linux/man-pages/man7/namespaces.7.html)

- cgroup  隔离cgroup
- IPC    隔离进程通信
- Network  隔离网络资源
- mount  隔离挂载点
- PID   隔离进程id
- Time    启动和单调的时钟
- User  隔离用户和用户组
- UTS  隔离主机名域名信息

##### IPC通信方式

1. 管道（pipe）实质缓存区，局限1半双工2亲缘关系进程3缓存区有限4无格式字节流
2. 有名管道（F IFO）
3. 信号（signal）
4. 消息队列（mq）
5. 共享内存
6. 信号量（semaphore）
7. 套接字(socket)

#### cgroup资源控制

```
groups的API以一个伪文件系统的方式实现，cpu相关(cpu,cpuacct,cpuset), 内存相关(memory)，块设备I/O相关(blkio)，网络相关(net_cls,net_prio)
/sys/fs/cgroup

查看内存使用
cat /sys/fs/cgroup/memory/docker/$container_id/memory.usage_in_bytes
内存硬限制软限制
cat /sys/fs/cgroup/memory/docker/$container_id/memory.limit_in_bytes
cat /sys/fs/cgroup/memory/docker/$container_id/memory.soft_limit_in_bytes

当前分组cpu耗时情况 单位是ns
cat cpuacct.usage
限制基于两种算法
Completely Fair Scheduler (CFS) 基于完全公平算法的调度器。
Real-Time scheduler (RT) 基于实时调度算法的调度器。

cat cpu.cfs_period_us  #调度周期 默认为100ms(100000μs）
cat cpu.cfs_quota_us  #表示在一个调度周期时间内（即cpu.cfs_period_us设定的时间），当前组内所有的进程允许在单个CPU上运行的总时长，微秒（μs）为单位。默认为-1，即不限制。

cpu.cfs_quota_us/cpu.cfs_period_us = 分配给当前组的cpu核数
eg：0.5core
cpu.cfs_quota_us = 50000
cpu.cfs_period_us= 100000

cpuset.cpus用于标明当前分组可以使用哪些CPU。
cpuset.mems用于标明当前分组可以使用哪些NUMA节点

blkio 查看
cat blkio.throttle.io_service_bytes
设备号ls -lt /dev/
echo "253:0 10485760" > blkio.throttle.write_bps_device #对253:0设备号写限制10M/s

进程数限制
cat /sys/fs/cgroup/pids/docker/$container_id/pids.max
```

#### unionFS

逻辑文件系统。

例如容器镜像目录和容器运行目录，修改运行时的目录不会对镜像目录修改

实现读写层分离：镜像层（只读）和容器层（读写）

Copy-on-write

OverlayFS

好处：

1. 节省磁盘空间
2. 共享资源

dockerfile优化：

```
1: 尽可能选择体积小From
2：尽可能合并RUN指令，清理无用的文件（yum缓存，源码包） 
3：修改dockerfile时，把需要变更的内容尽可能放在dockerfile结尾 
4：使用.dockerignore，减少不必要的文件ADD 
5: 打包镜像剥离

例：
FROM golang AS build-env
...
FROM alpine
...
COPY --from=build-env /root/pkg /

好处：
1提高分发效率
2节约资源
```

### PaaS应用
#### mysql
关系型数据库；主备模式；B+树设计模式
- 主从同步原理：日志同步原理3部分：
1. master将log写入binlog文件
2. slave主动拉去master binlog写入本地relaylog （slave_io线程）
3. sql i/o线程读取relaylog将数据写入slave数据库（slave_sql线程）
- 索引设计模式：
两种：B+和hash
#### mongodb
非关系型数据库，副本分片模式基于raft协议；B树设计模式
1. 副本数据同步oplog，主写入local库（log按顺序执行）
2. 副本分片，mongos，config集群。
#### elasticsearch
#### redis
- 4.0版本前单线程，4.0后支持多线程；
- 单线程处理能力100w/s，数据在内存，速度快；使用i/o多路复用；可维护性；单线程也可并发；瓶颈不在cpu，在网络io；
- 多线程：处理主线程以外的操作如：unlink，async， flashdb等；cpu切换线程也需要开销；
- 不开aof，单线程redis够用。
- aof是牺牲性能保存数据，数据重要开启。每次redis操作都需要记录log。
- rdb是数据快照，性能高，但是未快照期间数据没法保存。数据一般mem<50%；
#### kafka，rocketmq，rabbitmq 
- 结构：broker，topic，partition, consumer,producter,consumer group ,leader,follower。


## 网络

### tcp

#### 三次握手，四次挥手

tcp数据包头部32位长度，也就是4bytes。重要标记位

ACK —— 确认，使得确认号有效。

 RST —— 重置连接（经常看到的reset by peer）就是此字段搞的鬼。

 SYN —— 用于初如化一个连接的序列号。

 FIN —— 该报文段的发送方已经结束向对方发送数据

三次握手

<img src="https://github.com/miiniper/miiniper.github.io/raw/master/img/image-20201117171530995.png" alt="image-20201117171530995" style="zoom:60%;" />

四次挥手

<img src="https://github.com/miiniper/miiniper.github.io/raw/master/img/image-20201117171647544.png" alt="image-20201117171647544" style="zoom:60%;" />

2MSL存在原因：

```
1保证客户端发送的最后一个ACK报文段能够到达服务端。
这个ACK报文段有可能丢失，使得处于LAST-ACK状态的B收不到对已发送的FIN+ACK报文段的确认，服务端超时重传FIN+ACK报文段，而客户端能在2MSL时间内收到这个重传的FIN+ACK报文段，接着客户端重传一次确认，重新启动2MSL计时器，最后客户端和服务端都进入到CLOSED状态，若客户端在TIME-WAIT状态不等待一段时间，而是发送完ACK报文段后立即释放连接，则无法收到服务端重传的FIN+ACK报文段，所以不会再发送一次确认报文段，则服务端无法正常进入到CLOSED状态。
2防止“已失效的连接请求报文段”出现在本连接中。
客户端在发送完最后一个ACK报文段后，再经过2MSL，就可以使本连接持续的时间内所产生的所有报文段都从网络中消失，使下一个新的连接中不会出现这种旧的连接请求报文段

```

### http

#### http版本区别
- http1.1 ：长链接，断点续传，管道化
- http2.0 ：二进制分帧，多路复用，头部压缩
#### http status code

[说明](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Status)

```
1**	信息，服务器收到请求，需要请求者继续执行操作
2**	成功，操作被成功接收并处理
3**	重定向，需要进一步的操作以完成请求
4**	客户端错误，请求包含语法错误或无法完成请求
5**	服务器错误，服务器在处理请求的过程中发生了错误
```



#### restful api

[说明](https://restfulapi.net/)

```
get 获取资源 对应读
post 发布资源（提交表单） 对应写
```



### 排障思路

#### web

```
当遇到http非200时：
1 ping下地址（解决网络联通性和dns解析问题）
2 看下对应的性能监控：系统io，cpu，mem，net；服务端的qps 
3 查看对应服务log，看下具体问题。
4 看下对应时间的事件（events）多为变更事件。
5 是否位应用自身bug

解决：
1 网络问题具体排查优化
2 服务端性能问题较少，一般扩容可解决（水平或者深度扩容）；
3 重要在数据库瓶颈：使用中间件；加缓存；分库分表；磁盘ssd；

应急：
1 变更引起的：马上回滚
2 非变更引起：切流

平时：
1 监控完善
2 流量分流 高可用

```

#### 数据库

```
当数据库出现问题：
1 直接看日志报错
2 看数据库进程监控，是否io瓶颈，cpu是否高 -> log里查下那个语句引起的。

解决：
1 切库
2 kill掉io有问题的线程

平时：
1 数据备份
2 多副本
3 性能优化（系统和数据库层面，业务的语句优化）
4 缓存

```







## k8s
[官方文档](https://kubernetes.io/zh/docs/concepts/overview/components/)
### 基础
master：
kube-apiserver
etcd
kube-scheduler
kube-controller-manager
cloud-controller-manager

node：
kubelet
kube-proxy
Container Runtime 

周边：
dns
web
log
监控

#### 常见问题
1. 为什么pod是k8s最小调度单位而不是container？
- 多个容器在同一个pod中可以同时调度到多node集群的同一节点；容器通讯更快；共享挂载的文件系统。
2. 多container 的pod模式
- sidecar ：例如日志收集模式。
- Ambassador ：例如数据库代理
- Adapter ：处理一些标准化的事务，格式化log等
3. pod创建过程 [参考](https://fuckcloudnative.io/posts/what-happens-when-k8s/)
- kubectl 验证：语法合法，config获取：3个途径：env，kubeconfig参数，home/.kube/config，构造http请求发给apiserver
- apiserver 三步：认证，授权，准入控制；序列化后写入etcd
- etcd 写入相应的资源
- controller 对比当前值和期望值 deployment通过replicsets控制
- scheduler ：预选策略 打分 分高获得pod并更新etcd
- kubelet 初始化，创建puase容器（提供namespace资源） 同时cni提供网络服务提供ip，dns；启动init容器（拉去镜像，写入元数据标签）


### cni,csi,cri
相关文章：

[pod get ip](https://medium.com/cloud-belivers/how-kubernetes-pod-obtains-ip-address-3982ac9697b1)

#### flannel
1.使集群中的不同Node主机创建的Docker容器都具有全集群唯一的虚拟IP地址。

2.建立一个覆盖网络（overlay network），通过这个覆盖网络，将数据包原封不动的传递到目标容器。覆盖网络是建立在另一个网络之上并由其基础设施支持的虚拟网络。覆盖网络通过将一个分组封装在另一个分组内来将网络服务与底层基础设施分离。在将封装的数据包转发到端点后，将其解封装。

3.创建一个新的虚拟网卡flannel0接收docker网桥的数据，通过维护路由表，对接收到的数据进行封包和转发（vxlan）。

4.etcd保证了所有node上flanned所看到的配置是一致的。同时每个node上的flanned监听etcd上的数据变化，实时感知集群中node的变化

默认udp跨机器pod通信：pod-> vethh0->docker0->flannel0->eth0->flannel0->docker0->veth0->pod
##### 通讯方式
1. host gateway ：直接添加本地路由。限制：要求所有的主机都在一个子网内，即二层可达，否则就无法将目的主机当做网关，直接路由
2. udp ： pod-> vethh0->docker0->flannel0->eth0->eth0->flannel0->docker0->veth0->pod
3. vxlan ：会存主机的mac地址

#### calico
##### 通讯方式
1. ipip ：pod -> tunl0->eth0-eth0 -> tunl0->pod
2. bgp : pod -> eth0->eth0 -> pod 

#### ceph

### 监控

[参考](https://mp.weixin.qq.com/s/P3F1grbVpb7LF2hcxYNOcg)

#### api-server

 监控数据从[/metrics](https://github.com/kubernetes/apiserver/blob/master/pkg/endpoints/metrics/metrics.go)获取，三个维度监控

1. 延迟
2. 通讯量
3. 错误
4. 饱和度

### etcd
#### 监控方面
1. 延迟
2. 吞吐量

### 大集群瓶颈

### 调度

### CICD

### 源码
#### 学习go设计模式
[参考](https://github.com/Junedayday/code_reading)
1. factory模式
2. visitor 模式
### service mesh

todo

### serverless

todo

## 对sre职位思考

```
他不在是传统的运维，各个平台相对健全，人来维护平台即可，不在由人来做一些低技术含量的工作，例如重启，完全可以交给平台，在平台鉴权使应用owner可以重启。减少不必要的劳动。
一般出现故障很少是应用自身的问题，大多数是人为变更，网络问题。所以监控要到位metrics,log,trace三方面为基础监控，扩展有事件监控events。
人应该大多数时间用来学习研究新的技术，可以不用把时间浪费在一些可替代的事情上。
做到dev不用登录服务器，从开发，上线，测试，发布等。以后得目标要做到ops也尽可能的不登录服务器。排查问题完全可以在各个平台上操作。除非是本机的系统问题。
需要有扎实基础知识，排障能力，应急定位能力。
平台完善后sre甚至可以不在了解业务逻辑和链路情况。完全靠监控即可。
对容器的一些看法，不易过大，不然就失去了容器的意义，能做到单进程的为最优。其他的完全可以在容器外部做。例如监控数据，日志等。
举例，golang程序打包一个from image，RUN在一个from image。。不赞同定制化的镜像，把监控日志收集和应用打在同一个镜像中，同样是消耗集群资源，但是这样做会更多的抢占进程资源。
发现问题，系统分析能力。
下一步就可以学习了解更深层次的原理，实现良性循环。
可能存在的痛点，数据存储，底层问题尤其是涉及到内核问题，网络问题，数据一致性。
面对大流量的思考，弹性扩容迅速，中间件缓冲，数据库读写瓶颈(分库分表)。
```

