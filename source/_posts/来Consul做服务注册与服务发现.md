---
title: 用 Consul 来做服务注册与服务发现
date: 2019-03-28 15:14:38
tags: [Consul, gRPC]
---
服务注册与服务发现是在分布式服务架构中常常会涉及到的东西，业界常用的服务注册与服务发现工具有 [ZooKeeper](https://zookeeper.apache.org/)、[etcd](https://coreos.com/etcd/)、[Consul](https://www.consul.io/) 和 [Eureka](https://github.com/Netflix/eureka)。Consul 的主要功能有服务发现、健康检查、KV存储、安全服务沟通和多数据中心。Consul 与其他几个工具的区别可以在这里查看 [Consul vs. Other Software](https://www.consul.io/intro/vs/index.html)。
<!-- more -->
### 为什么需要有服务注册与服务发现？
假设在分布式系统中有两个服务 Service-A （下文以“S-A”代称）和 Service-B（下文以“S-B”代称），当 S-A 想调用 S-B 时，我们首先想到的时直接在 S-A 中请求 S-B 所在服务器的 IP 地址和监听的端口，这在服务规模很小的情况下是没有任何问题的，但是在服务规模很大每个服务不止部署一个实例的情况下是存在一些问题的，比如 S-B 部署了三个实例 S-B-1、S-B-2 和 S-B-3，这时候 S-A 想调用 S-B 该请求哪一个服务实例的 IP 呢？还是将3个服务实例的 IP 都写在 S-A 的代码里，每次调用 S-B 时选择其中一个 IP？这样做显得很不灵活，这时我们想到了 `Nginx` 刚好就能很好的解决这个问题，引入 `Nginx` 后现在的架构变成了如下图这样：
![](http://nnblog-storage.b0.upaiyun.com/img/SA-N-SB.jpeg)
引入 Nginx 后就解决了 S-B 部署多个实例的问题，还做了 S-B 实例间的负载均衡。但现在的架构又面临了新的问题，分布式系统往往要保证高可用以及能做到动态伸缩，在引入 Nginx 的架构中，假如当 S-B-1 服务实例不可用时，Nginx 仍然会向 S-B-1 分配请求，这样服务就不可用，我们想要的是 S-B-1 挂掉后 Nginx 就不再向其分配请求，以及当我们新部署了 S-B-4 和 S-B-5 后，Nginx 也能将请求分配到 S-B-4 和 S-B-5，Nginx 要做到这样就要在每次有服务实例变动时去更新配置文件再重启 Nginx。这样看似乎用了 Nginx 也很不舒服以及还需要人工去观察哪些服务有没有挂掉，Nginx 要是有对服务的健康检查以及能够动态变更服务配置就是我们想要的工具，这就是服务注册与服务发现工具的用处。下面是引入服务注册与服务发现工具后的架构图：
![](http://nnblog-storage.b0.upaiyun.com/img/service_discovery.jpg)

在这个架构中：
- 首先 S-B 的实例启动后将自身的服务信息（主要是服务所在的 IP 地址和端口号）注册到注册工具中。不同注册工具服务的注册方式各不相同，后文会讲 Consul 的具体注册方式。
- 服务将服务信息注册到注册工具后，注册工具就可以对服务做健康检查，以此来确定哪些服务实例可用哪些不可用。
- S-A 启动后就可以通过服务注册和服务发现工具获取到所有健康的 S-B 实例的 IP 和端口，并将这些信息放入自己的内存中，S-A 就可用通过这些信息来调用 S-B。
- S-A 可以通过监听（Watch）注册工具来更新存入内存中的 S-B 的服务信息。比如 S-B-1 挂了，健康检查机制就会将其标为不可用，这样的信息变动就被 S-A 监听到了，S-A 就更新自己内存中 S-B-1 的服务信息。

所以务注册与服务发现工具除了服务本身的服务注册和发现功能外至少还需要有健康检查和状态变更通知。

### Consul
Consul 作为一种分布式服务工具，为了避免单点故障常常以集群的方式进行部署，在 Consul 集群的节点中分为 Server 和 Client 两种节点（所有的节点也被称为Agent），Server 节点保存数据，Client 节点负责健康检查及转发数据请求到 Server；Server 节点有一个 Leader 节点和多个 Follower 节点，Leader 节点会将数据同步到 Follower，在 Leader 挂掉的时候会启动选举机制产生一个新的 Leader。

Client 节点很轻量且无状态，它以 RPC 的方式向 Server 节点做读写请求的转发，此外也可以直接向 Server 节点发送读写请求。下面是 Consul 的架构图：
![](http://nnblog-storage.b0.upaiyun.com/img/consul-arch-420ce04a.png)
Consule 的安装和具体使用及其他详细内容可浏览[官方文档](https://www.consul.io/docs/index.html)。
下面是我用 Docker 的方式搭建了一个有3个 Server 节点和1个 Client 节点的 Consul 集群。
```
# 这是第一个 Consul 容器，其启动后的 IP 为172.17.0.5
docker run -d --name=c1 -p 8500:8500 -e CONSUL_BIND_INTERFACE=eth0 consul agent --server=true --bootstrap-expect=3 --client=0.0.0.0 -ui

docker run -d --name=c2 -e CONSUL_BIND_INTERFACE=eth0 consul agent --server=true --client=0.0.0.0 --join 172.17.0.5

docker run -d --name=c3 -e CONSUL_BIND_INTERFACE=eth0 consul agent --server=true --client=0.0.0.0 --join 172.17.0.5

#下面是启动 Client 节点
docker run -d --name=c4 -e CONSUL_BIND_INTERFACE=eth0 consul agent --server=false --client=0.0.0.0 --join 172.17.0.5

```
启动容器时指定的环境变量 `CONSUL_BIND_INTERFACE` 其实就是相当于指定了 Consul 启动时 `--bind` 变量的参数，比如可以把启动 c1 容器的命令换成
```
docker run -d --name=c1 -p 8500:8500 -e consul agent --server=true --bootstrap-expect=3 --client=0.0.0.0 --bind='{{ GetInterfaceIP "eth0" }}' -ui
```
也是一样的效果。进入任意一个容器执行 `consul members` 都可以有如下的输出，说明 Consul 集群就已经搭建成功了。
```
Node          Address          Status  Type    Build  Protocol  DC   Segment
2dcf0c824cf0  172.17.0.7:8301  alive   server  1.4.4  2         dc1  <all>
64746cffa116  172.17.0.6:8301  alive   server  1.4.4  2         dc1  <all>
77af7d94a8ca  172.17.0.5:8301  alive   server  1.4.4  2         dc1  <all>
6c71148f0307  172.17.0.8:8301  alive   client  1.4.4  2         dc1  <default>
```
### 项目实践
