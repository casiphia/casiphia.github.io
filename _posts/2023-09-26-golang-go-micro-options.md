---
title: "go-micro 框架源码剖析之函数选项模式"
categories:
  - Golang
tags:
  - go-micro
last_modified_at: 2023-09-26T10:28:50-05:00
---

go-micro整个框架都采用了`函数式编程`,如果你对`函数式编程`不了解的话，很难看懂源码，以及使用框架，本文将带你简单介绍一下go-micro下的`函数式编程`，方便你入门使用。

## 创建微服务示例

在go-micro中使用micro.NewService创建一个微服务

```go
import "github.com/micro/v2/go-micro"

service := micro.NewService()
```

也可以在创建过程中设置服务选项（如服务名称，自定义服务发现等）

```go
service := micro.NewService(
        micro.Name("hellooo"),
        micro.Version("latest"),
)
```
对比两个例子可以发现，虽然go语言不支持默认参数，但是上面的代码例子却达到了同样的效果。

## 如何实现默认参数

那么，go-micro是如何实现默认参数的呢？进入micro.NewService源码看个究竟：`github.com/micro/go-micro/go-micro.go`

```go
// Option定义为一个方法，接受Options类型的指针参数
// 注意这个Option类型（函数类型），这是理解本文的关键
type Option func(*Options)
...
// 创建并返回一个服务：接收Option函数类型的不定向参数列表
func NewService(opts ...Option) Service {
    return newService(opts...)
}

...

// 内部真正创建服务的方法
func newService(opts ...Option) Service {
    // 关键步骤：利用opts函数列表初始化服务
    options := newOptions(opts...)

     ...

    // 返回初始化的服务
    return &service{
        opts: options,
    }
}
```

其中newOptions方法最为关键，继续跟踪下去：`github.com/micro/go-micro/options.go`，该文件定义了设置服务相关的所有选项：

```go
// 服务选项结构体
// 因为go-micro是可插件化的框架，其中的组件（如消息中间件）均是可以用其他类似服务替换的
type Options struct {
    Broker    broker.Broker 
    Cmd       cmd.Cmd
    Client    client.Client
    Server    server.Server
    Registry  registry.Registry
    Transport transport.Transport

    // Register loop interval
    RegisterInterval time.Duration

    // Before and After funcs
    BeforeStart []func() error
    BeforeStop  []func() error
    AfterStart  []func() error
    AfterStop   []func() error

    Context context.Context
}

// 生成服务相关选项（初始化使用go-micro框架定义的默认组件）
func newOptions(opts ...Option) Options {
    // 初始化默认值
    opt := Options{
        Broker:    broker.DefaultBroker,
        Cmd:       cmd.DefaultCmd,
        Client:    client.DefaultClient,
        Server:    server.DefaultServer,
        Registry:  registry.DefaultRegistry,
        Transport: transport.DefaultTransport,
        Context:   context.Background(),
    }

    for _, o := range opts {
        o(&opt) // 依次调用opts函数列表中的函数，为服务选项（opt变量）赋值
    }

    return opt
}

...

// 服务名字选项
// 返回一个Option类型的函数（闭包）：接受Options类型指针参数并修改之
func Name(n string) Option {
    return func(o *Options) {
        o.Server.Init(server.Name(n))
    }
}

// 服务版本选项
// 返回一个Option类型的函数（闭包）：接受Options类型指针参数并修改之
func Version(v string) Option {
    return func(o *Options) {
        o.Server.Init(server.Version(v))
    }
}

...
```

除了Name与Version选项之外，go-micro的Broker、Register、Transport等选项均是通过此方法实现的。它极大地利用了go语言支持闭包的特性，优雅地实现了函数支持默认参数
的功能。
再次回到文章开头出创建服务的代码就很好理解了：

```go
// 创建服务，同时接收对指定参数选项的设置
service := micro.NewService(
        micro.Name("hellooo"),
        micro.Version("latest"),
)
```

这样的写法处理调用代码比较简洁之外，它扩展性特别好，增加新的选项时，只需要很少量的代码。
这便是go语言的函数选项模式。

## 函数选项（Functional Options）模式

函数选项模式是由Rob Pike提出，并由Dave Cheney等推广开，它优雅地解决了go语言中默认参数问题。

虽然Functional Options并不是一个新的概念，从Rob Pike提出至今已4年有余（2014），但作为Golang新手，还是很受启发。这也是阅读源码带来的好处。

下面总结一下，函数选项模式有哪些优点：

支持默认参数：不必向结构体参数那样，不使用时仍必须参数一个空的struct值
代码简洁：即使是像go-micro这种支持如此繁多选项，代码也很美观
扩展性好：增加新的选项只需少量代码
推而广之：类似结构体中变量的赋值都可以效仿之。