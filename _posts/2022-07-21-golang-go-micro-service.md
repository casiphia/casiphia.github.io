---
title: "go-micro接口调用"
categories:
  - Golang
tags:
  - go-micro
  - rabbitmq
last_modified_at: 2022-07-21T16:10:50-05:00
---

go-micro在微服务框架中提供了开箱即用的灵活接口，但是官方文档以及版本过于混乱，导致很难上手，初次使用很多调用都需要去阅读源码才能使用，微服务之间最核心的功能就是接口调用，当你使用go-micro的微服务去调用另一个go-micro的微服务时，使用起来简单，但是如果跨语言呢，grpc是没有语言限制的，假设你用go-micro实现了一个微服务，使用java作为客户端去调用呢？本教程主要记录go-micro在跨语言上调用接口的方法，希望对你有所帮助。

## 环境准备

使用`micro`创建一个微服务，提供给接口用来测试。

``` bash
$ micro new geeter
$ cd geeter
$ make proto
$ go mod tidy
$ go run main

2022-07-21 11:31:35  file=v2@v2.9.1/service.go:200 level=info Starting [service] go.micro.service.geeter
2022-07-21 11:31:35  file=grpc/grpc.go:864 level=info Server [grpc] Listening on [::]:53294
2022-07-21 11:31:35  file=grpc/grpc.go:881 level=info Broker [http] Connected to 127.0.0.1:53295
2022-07-21 11:31:35  file=grpc/grpc.go:697 level=info Registry [mdns] Registering node: go.micro.service.geeter-12413e0b-a51a-45da-954c-38b99a80d276
2022-07-21 11:31:35  file=grpc/grpc.go:730 level=info Subscribing to topic: go.micro.service.geeter
```

> 默认使用mdns作为注册中心，系统随机分配端口53294

## go-micro微服务接口调用

使用`go-micro`微服务调用必须要能从注册中心发现服务

```go
	// New Service
	service := micro.NewService(
		micro.Name("go.micro.client.geeter"),
		micro.Version("latest"),
	)

	// Initialise service
	service.Init()
```

> 这里客户端仅实现了服务发现，并没有实现服务注册，所以在注册中心时发现不了该客户端的

```go
cli := geeter.NewGeeterService("go.micro.service.geeter", client.DefaultClient)
resp, err := cli.Call(context.TODO(), &geeter.Request{
  Name: "micro call.",
})
fmt.Println(resp, err)
```

> go.micro.service.geeter:微服务名称，这里不需要关心微服务的端口，因为走注册中心通过微服务名称调用，比较灵活
>
> client.DefaultClient:实例化默认客户端

`client.DefaultClient`是go-micro默认的客户端，其内部实现如下：

``` go
// DefaultClient is a default client to use out of the box
	DefaultClient Client = newRpcClient()
	// DefaultBackoff is the default backoff function for retries
	DefaultBackoff = exponentialBackoff
	// DefaultRetry is the default check-for-retry function for retries
	DefaultRetry = RetryOnError
	// DefaultRetries is the default number of times a request is tried
	DefaultRetries = 1
	// DefaultRequestTimeout is the default request timeout
	DefaultRequestTimeout = time.Second * 5
	// DefaultPoolSize sets the connection pool size
	DefaultPoolSize = 100
	// DefaultPoolTTL sets the connection pool ttl
	DefaultPoolTTL = time.Minute

	// NewClient returns a new client
	NewClient func(...Option) Client = newRpcClient
```

支持自定义客户端，对客户端的超时、重试、线程池按需定义

## 其他框架调用go-micro服务接口

使用其他框架调用`go-micro`服务必须要重新生成`protoc`文件，这里也是使用`Go`作为演示，不同的是使用grpc标准方法调用

``` bash
protoc --go_out=plugins=grpc:. *.proto 
```

仅生成`geete.pb.go`文件

首先初始化grpc链接

```go
// 创建客户端连接
conn, err := grpc.Dial("127.0.0.1:53294", grpc.WithInsecure())
if err != nil {
  fmt.Println("grpc Dial err: ", err)
  return
}
defer conn.Close()
```

创建远程调用的客户端

``` go
// 创建远程服务的客户端
cli := geete.NewGeeterClient(conn)
resp, err := cli.Call(context.TODO(), &geete.Request{
  Name: "micro call.",
})
fmt.Println(resp, err)
```

> 127.0.0.1:53294：需要注意，使用grpc标准接口调用，必须指定ip和端口，这种情况下，要求服务端必须要指定端口启动

好了，以上就是本文的主要内容。

老规矩，代码已经上传到Github，欢迎访问： [https://github.com/casiphia/go-micro-examples](