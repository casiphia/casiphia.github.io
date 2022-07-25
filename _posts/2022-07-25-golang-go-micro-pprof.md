---
title: "go-micro pprof分析工具"
categories:
  - Golang
tags:
  - go-micro
  - pprof
last_modified_at: 2022-07-25T10:28:50-05:00
---

pprof是golang程序性能分析工具，go-micro基于官方pprof做了一层封装，对网络和应用封装了一套完整的分析方法。

## 源码分析

### profile

`go-micro`的pprof分析在以下包中：

```go
"github.com/micro/go-micro/v2/debug/profile"
"github.com/micro/go-micro/v2/debug/profile/pprof"
```

其中`profile`是封装好的方法。其内部提供：

``` go
type Profile interface {
	// Start the profiler
	Start() error
	// Stop the profiler
	Stop() error
	// Name of the profiler
	String() string
}
```

> Start():性能监控启动
>
> Stop():性能监控停止

还提供了自定义pprof路由名字的接口：

``` go
// Name of the profile
func Name(n string) Option {
	return func(o *Options) {
		o.Name = n
	}
}
```

提供一个开箱即用的profile

```go
var (
	DefaultProfile Profile = new(noop)
)
```

其内部实现了`runtime/pprof`和`net/http/pprof`两种场景的采样性能监控，实际使用我们提到

## 实战部分

实际项目中我们需要自己配置一些pprof的参数，方便我们标识是对那些应用作性能监控

runtime/pprof性能监控配置如下

``` go
import (
	"github.com/micro/go-micro/v2/debug/profile"
	"github.com/micro/go-micro/v2/debug/profile/pprof"
)

func PprofBoot() {
	pf := pprof.NewProfile(
		profile.Name("test"),
	)
	pf.Start()
}
```

>1.使用`pprof`的方法NewProfile(),改方法返回一个profile.Profile对象
>
>2.NewProfile()支持opts ...profile.Option配置参数，目前只有修改监控前缀这个一个配置参数
>
>3.使用Start()可直接启动

net/http/pprof采样监控

```go
import (
	"github.com/micro/go-micro/v2/debug/profile"
	"github.com/micro/go-micro/v2/debug/profile/http"
)

func PprofBoot() {
	pf := http.NewProfile(
		profile.Name("test"),
	)
	pf.Start()
}
```

这种采样模式下，官方默认端口为`6060`

``` go
var (
	DefaultAddress = ":6060"
)
```

默认监控路由如下

``` go
mux.HandleFunc("/debug/pprof/", pprof.Index)
mux.HandleFunc("/debug/pprof/cmdline", pprof.Cmdline)
mux.HandleFunc("/debug/pprof/profile", pprof.Profile)
mux.HandleFunc("/debug/pprof/symbol", pprof.Symbol)
mux.HandleFunc("/debug/pprof/trace", pprof.Trace)
```

> 暂时不支持自定义端口

主函数调用

``` go
if config.RunMode == "debug" {
		go monitor.PprofBoot()
}
```

之后就可以使用`go tool pprof`工具监控你的程序啦

## pprof使用

使用交互终端

``` bash
[root@master-20 ~]#go tool pprof http://localhost:6060/debug/pprof/profile
Fetching profile over HTTP from http://localhost:6060/debug/pprof/profile
Saved profile in /root/pprof/pprof.go-pprof-practice.samples.cpu.001.pb.gz
File: go-pprof-practice
Type: cpu
Time: Apr 23, 2021 at 11:57am (CST)
Duration: 30.14s, Total samples = 19.66s (65.22%)
Entering interactive mode (type "help" for commands, "o" for options)
(pprof)
```

输入 top 命令，查看 CPU 占用的调用：

``` bash
(pprof) top
Showing nodes accounting for 19.21s, 99.48% of 19.31s total
Dropped 22 nodes (cum <= 0.10s)
      flat  flat%   sum%        cum   cum%
    19.21s 99.48% 99.48%     19.26s 99.74%  go-pprof-practice/animal/felidae/tiger.(*Tiger).Eat
         0     0% 99.48%     19.26s 99.74%  go-pprof-practice/animal/felidae/tiger.(*Tiger).Live
         0     0% 99.48%     19.30s 99.95%  main.main
         0     0% 99.48%     19.30s 99.95%  runtime.main
```

使用`list Eat`,查看问题具体在代码的哪一个位置

``` bash
(pprof) list Eat
Total: 19.31s
ROUTINE ======================== go-pprof-practice/animal/felidae/tiger.(*Tiger).Eat in /root/go-pprof-practice/animal/felidae/tiger/tiger.go
    19.21s     19.26s (flat, cum) 99.74% of Total
         .          .     19:}
         .          .     20:
         .          .     21:func (t *Tiger) Eat() {
         .          .     22:	log.Println(t.Name(), "eat")
         .          .     23:	loop := 10000000000
    19.21s     19.26s     24:	for i := 0; i < loop; i++ {
         .          .     25:		// do nothing
         .          .     26:	}
         .          .     27:}
         .          .     28:
         .          .     29:func (t *Tiger) Drink() {
```

对于不熟悉终端的我们可以是用图形界面查看调用栈的信息

```bash
yum install graphviz
```

安装完成后，我们继续在上文的交互式终端里输入 `web`，注意，虽然这个命令的名字叫“web”，但它的实际行为是产生一个 .svg 文件，并调用你的系统里设置的默认打开 .svg 的程序打开它。如果你的系统里打开 .svg 的默认程序并不是浏览器（比如可能是你的代码编辑器），这时候你需要设置一下默认使用浏览器打开 .svg 文件

如果浏览器无法使用，可以输入 svg，然后会在当前目录生成svg文件，然后使用浏览器打开也可以看到

就可以看到整个链路调用的火焰图了