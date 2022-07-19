---
title: "go-micro使用etcd作为注册中心"
categories:
  - Golang
tags:
  - go-micro
  - etcd
last_modified_at: 2022-07-19T14:28:50-05:00
---

这一篇就来讲讲，go-micro v2 如何进行配置etcd注册中心和操作配置中心

## 前言

go-micro框架为服务注册发现提供了标准的接口Registry。只要实现这个接口就可以定制自己的服务注册和发现。不过官方已经为主流注册中心提供了官方的接口实现，大多数时候我们不需要从头写起。

官方默认实现了mdns，consul，etcd等注册中心接口，提供了开箱即用的方法。

> v2以前：默认使用`consul`作为注册中心。
>
> 最新版本：默认使用`mDNS` 提供零配置的发现系统，大多数系统已经内置，程序不需要任何改动就具备服务注册和发现能力。

实际生产中，官方则推荐使用`etcd`组成更具弹性的集群方案。

## 简介

etcd作为服务发现系统，有以下的特点： 

- 简单：安装配置简单，而且提供了HTTP API进行交互，使用也很简单 
- 安全：支持SSL证书验证 
- 快速：根据官方提供的benchmark数据，单实例支持每秒2k+读操作 
- 可靠：采用raft算法，实现分布式系统数据的可用性和一致性

etcd项目地址：[https://github.com/coreos/etcd/](https://github.com/coreos/etcd/)

### 启动服务

```bash
docker run --name etcd -d -p 2379:2379 -p 2380:2380 -e ALLOW_NONE_AUTHENTICATION=yes bitnami/etcd:3.3.11 etcd
docker exec -it etcd /bin/bash
```

## 服务配置

### 代码配置

在标准接口下，server和client端诶之方式没有任何区别，这边会分开展示

#### 服务注册

``` go
...
// New Service
	service := micro.NewService(
		// 服务名称
		micro.Name("go.micro.service.demo"),
		// 服务版本
		micro.Version("latest"),
		micro.Registry(etcd.NewRegistry(
			registry.Addrs("127.0.0.1:2379"),
		)), //etcd注册
	)
...
```

这里使用go-micro已经实现的etcd包配置etcd的地址即可

```go
import (
	"github.com/micro/go-micro/v2/registry"
	"github.com/micro/go-micro/v2/registry/etcd"
)
```

我们也可以查看源码`etcd.NewRegistry`支持的配置项

```go
type Options struct {
    // 地址列表
    Addrs     []string
    // 超时时间
    Timeout   time.Duration
    // 与注册中心的安全通信
    Secure    bool
    // tls加密通信配置
    TLSConfig *tls.Config
    // Other options for implementations of the interface
    // can be stored in a context
    Context context.Context
}
```

然后正常启动程序，在日志中会发现已经注册到etcd

``` shell
2022-07-19 10:14:10  file=v2@v2.9.2-0.20201226154210-35d72660c801/service.go:192 level=info Starting [service] go.micro.service.demo
2022-07-19 10:14:10  file=grpc/grpc.go:864 level=info Server [grpc] Listening on [::]:61922
2022-07-19 10:14:10  file=grpc/grpc.go:697 level=info Registry [etcd] Registering node: go.micro.service.demo-d39516cd-a8a2-438f-bb90-fe897bf9ed7c
```

这里看到服务已经正常移动，并注册到etcd中

> 61922:这是服务的端口，使用etcd作为注册中心，服务间相互调用是通过服务名称的，服务本身的端口号不需要我们自己维护，client通过注册中心的服务名就能call服务的rpc接口，但是如果你使用grpc的官方方法来call服务的接口，必须要要自己指定端口启动，使用`micro.Address`来指定端口号
>
> go.micro.service.demo-d39516cd-a8a2-438f-bb90-fe897bf9ed7c:注册中心中的key

#### 服务发现

服务发现和服务注册配置是一致的，当你仅需要client去调用go-micro的微服务，而client不提供给外部的rpc接口时，你可以这样配置

``` go
...
service := micro.NewService(
    micro.Name("go.micro.client"),
    // 配置etcd为注册中心，配置etcd路径，默认端口是2379
    micro.Registry(etcd.NewRegistry(
        // 地址是我本地etcd服务器地址，不要照抄
        registry.Addrs("172.18.0.58:2379"),
    )),
)
service.Init()
...
```

> 这里完全不用调用service.Run(),因为你仅需要去注册中心call微服务的rpc方法，并不提供接口给其他服务调用

### 命令行参数

go-micro除了可以在代码中指定配置注册中心，还支持命令行获取注册中心配置

在启动命令中增加如下参数：

- `--registry=etcd`
- `--registry_address=172.18.0.58:2379`

或

- `MICRO_REGISTRY_ADDRESS=172.18.0.58:2379`
- `MICRO_REGISTRY=etcd`

使用上面方法也可达到相同的效果，但是生产过程中使用的很少，虽然他很灵活，但是依赖系统的环境配置，给部署增加了工作量

## 总结

我们可以用很少的代码，甚至不需要改动现有的服务，将注册中心切换到etcd来，这基于统一的接口设计方式值得我们学习使用。