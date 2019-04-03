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

所以务注册与服务发现工具除了服务本身的服务注册和发现功能外至少还需要有健康检查和状态变更通知的功能。

### Consul
Consul 作为一种分布式服务工具，为了避免单点故障常常以集群的方式进行部署，在 Consul 集群的节点中分为 Server 和 Client 两种节点（所有的节点也被称为Agent），Server 节点保存数据，Client 节点负责健康检查及转发数据请求到 Server；Server 节点有一个 Leader 节点和多个 Follower 节点，Leader 节点会将数据同步到 Follower 节点，在 Leader 节点挂掉的时候会启动选举机制产生一个新的 Leader。

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
启动容器时指定的环境变量 `CONSUL_BIND_INTERFACE` 其实就是相当于指定了 Consul 启动时 `--bind` 变量的参数，比如可以把启动 c1 容器的命令换成下面这样，也是一样的效果。
```
docker run -d --name=c1 -p 8500:8500 -e consul agent --server=true --bootstrap-expect=3 --client=0.0.0.0 --bind='{{ GetInterfaceIP "eth0" }}' -ui
```
操作 Consul 有 [Commands](https://www.consul.io/docs/commands/index.html) 和 [HTTP API](https://www.consul.io/api/index.html) 两种方式，进入任意一个容器执行 `consul members` 都可以有如下的输出，说明 Consul 集群就已经搭建成功了。
```
Node          Address          Status  Type    Build  Protocol  DC   Segment
2dcf0c824cf0  172.17.0.7:8301  alive   server  1.4.4  2         dc1  <all>
64746cffa116  172.17.0.6:8301  alive   server  1.4.4  2         dc1  <all>
77af7d94a8ca  172.17.0.5:8301  alive   server  1.4.4  2         dc1  <all>
6c71148f0307  172.17.0.8:8301  alive   client  1.4.4  2         dc1  <default>
```
### 代码实践
假设现在有一个用 Node.js 写的服务 node-server 需要通过 [gRPC](https://grpc.io/) 的方式调用一个用 Go 写的服务 go-server。
下面是用 Protobuf 定义的服务和数据类型文件 `hello.proto`。
```
syntax = "proto3";

package hello;
option go_package = "hello";

// The greeter service definition.
service Greeter {
  // Sends a greeting
  rpc SayHello (stream HelloRequest) returns (stream HelloReply) {}
}

// The request message containing the user's name.
message HelloRequest {
  string name = 1;
}

// The response message containing the greetings
message HelloReply {
  string message = 1;
}

```
用命令通过 Protobuf 的定义生成 Go 语言的代码：`protoc  --go_out=plugins=grpc:./hello ./*.proto` 会在 hello 目录下得到 hello.pb.go 文件，然后在 hello.go 文件中实现我们定义的 RPC 服务。
```go
// hello.go
package hello

import "context"

type GreeterServerImpl struct {}

func (g *GreeterServerImpl) SayHello(c context.Context, h *HelloRequest) (*HelloReply, error)  {
	result := &HelloReply{
		Message: "hello" + h.GetName(),
	}
	return result, nil
}
```
下面是入口文件 `main.go`，主要是将我们定义的服务注册到 gRPC 中，并建了一个 `/ping` 接口用于之后 Consul 的健康检查。
```go
package main

import (
	"go-server/hello"
	"google.golang.org/grpc"
	"net"
	"net/http"
)

func main() {
	lis1, _ := net.Listen("tcp", ":8888")
	lis2, _ := net.Listen("tcp", ":8889")
	grpcServer := grpc.NewServer()
	hello.RegisterGreeterServer(grpcServer, &hello.GreeterServerImpl{})
	go grpcServer.Serve(lis1)
	go grpcServer.Serve(lis2)
	
	http.HandleFunc("/ping", func(res http.ResponseWriter, req *http.Request){
		res.Write([]byte("pong"))
	})
	http.ListenAndServe(":8080", nil)
}
```
至此 go-server 端的代码就全部编写完了，可以看出代码里面没有任何涉及到 Consul 的地方，用 Consul 做服务注册是可以做到对项目代码没有任何侵入性的。下面要做的是将 go-server 注册到 Consul 中。将服务注册到 Consul 可以通过直接调用 Consul 提供的 REST API 进行注册，还有一种对项目没有侵入的配置文件进行注册。Consul 服务配置文件的详细内容可以[在此查看](https://www.consul.io/docs/agent/services.html)。下面是我们通过配置文件进行服务注册的配置文件 `services.json`：
```json
{
  "services": [
    {
      "id": "hello1",
      "name": "hello",
      "tags": [
        "primary"
      ],
      "address": "172.17.0.9",
      "port": 8888,
      "checks": [
        {
          "http": "http://172.17.0.9:8080/ping",
          "tls_skip_verify": false,
          "method": "GET",
          "interval": "10s",
          "timeout": "1s"
        }
      ]
    },{
      "id": "hello2",
      "name": "hello",
      "tags": [
        "second"
      ],
      "address": "172.17.0.9",
      "port": 8889,
      "checks": [
        {
          "http": "http://172.17.0.9:8080/ping",
          "tls_skip_verify": false,
          "method": "GET",
          "interval": "10s",
          "timeout": "1s"
        }
      ]
    }
  ]
}
```
配置文件中的 `172.17.0.9` 代表的是 go-server 所在服务器的 IP 地址，`port` 就是服务监听的不同端口，`check` 部分定义的就是健康检查，Consul 会每隔 10秒钟请求一下 `/ping` 接口以此来判断服务是否健康。将这个配置文件复制到 c4 容器的 /consul/config 目录，然后执行`consul reload` 命令后配置文件中的 hello 服务就注册到 Consul 中去了。通过在宿主机执行`curl http://localhost:8500/v1/catalog/services\?pretty`就能看到我们注册的 hello 服务。
下面是 node-server 服务的代码:
```javascript
const grpc = require('grpc');
const axios = require('axios');
const protoLoader = require('@grpc/proto-loader');
const packageDefinition = protoLoader.loadSync(
  './hello.proto',
  {
    keepCase: true,
    longs: String,
    enums: String,
    defaults: true,
    oneofs: true
  });
const hello_proto = grpc.loadPackageDefinition(packageDefinition).hello;

function getRandNum (min, max) {
  min = Math.ceil(min);
  max = Math.floor(max);
  return Math.floor(Math.random() * (max - min + 1)) + min;
};

const urls = []
async function getUrl() {
  if (urls.length) return urls[getRandNum(0, urls.length-1)];
  const { data } = await axios.get('http://172.17.0.5:8500/v1/health/service/hello');
  for (const item of data) {
    for (const check of item.Checks) {
      if (check.ServiceName === 'hello' && check.Status === 'passing') {
        urls.push(`${item.Service.Address}:${item.Service.Port}`)
      }
    }
  }
  return urls[getRandNum(0, urls.length - 1)];
}

async function main() {
  const url = await getUrl();
  const client = new hello_proto.Greeter(url, grpc.credentials.createInsecure());
   
  client.sayHello({name: 'jack'}, function (err, response) {
    console.log('Greeting:', response.message);
  }); 
}

main()
```
代码中 172.17.0.5 地址为 c1 容器的 IP 地址，node-server 项目中直接通过 Consul 提供的 API 获得了 hello 服务的地址，拿到服务后我们需要过滤出健康的服务的地址，再随机从所有获得的地址中选择一个进行调用。代码中没有做对 Consul 的监听，监听的实现可以通过不断的轮询上面的那个 API 过滤出健康服务的地址去更新 `urls` 数组来做到。现在启动 node-server 就可以调用到 go-server 服务。
服务注册与发现给服务带来了动态伸缩的能力，也给架构增加了一定的复杂度。Consul 除了服务发现与注册外，在配置中心、分布式锁方面也有着很多的应用。
