---
title: "go-micro使用etcd存储配置"
categories:
  - Golang
tags:
  - go-micro
last_modified_at: 2022-07-19T15:28:50-05:00
---

不管是单个服务还是微服务，读取文件在每个项目系统中是必不可少的部分。

大多数项目中都是静态加载项目配置文件的，有时候可能需要从各种源中读取配置数据，这让配置读取复杂化，不易于快速开发。而go-micro中，不管是从动态读取配置，还是从多元读取配置都很简单，唯一难点就是需要读取源码来了解他的工作机制。

Go-Micro支持多种源的读取，包括命令行参数、文件（json、yaml）、etcd、consul、k8s等。

## 核心概念

Go-Micro 配置核心模块有四个，分别是 Source、Encoder、Reader、Config。

对于**Source，它表示读取的配置源**，比如文件、命令行、consul、etcd等。Go-Micro官方支持的源有：

- cli：即命令行参数
- env：从环境变量中读取
- consul：从consul中读取
- etcd：从etcd中读取
- file：从文件中读取
- flag：从flags中读取
- memory：从内存中读取

除了官方支持的源，还有一些是Go-Micro社区支持的：

- configmap：从 k8s 的configmap中读取

- grpc：从grpc服务器读取

- url：从URL读取

- runtimevar：read from Go Cloud Development Kit runtime variable

- vault：read from Vault server

## 源码分析

go-micro初始化config,`config.DefaultConfig`,在config/config.go中

``` go
var (
    // Default Config Manager
    DefaultConfig, _ = NewConfig()
)

// NewConfig returns new config
func NewConfig(opts ...Option) (Config, error) {
    return newConfig(opts...)
}

func newConfig(opts ...Option) (Config, error) {
    var c config

    c.Init(opts...)
    go c.run()

    return &c, nil
}

func (c *config) Init(opts ...Option) error {
    c.opts = Options{
        Reader: json.NewReader(),
    }
    c.exit = make(chan bool)
    for _, o := range opts {
        o(&c.opts)
    }

    // default loader uses the configured reader
    if c.opts.Loader == nil {
        c.opts.Loader = memory.NewLoader(memory.WithReader(c.opts.Reader))
    }

    err := c.opts.Loader.Load(c.opts.Source...)
    if err != nil {
        return err
    }

    c.snap, err = c.opts.Loader.Snapshot()
    if err != nil {
        return err
    }

    c.vals, err = c.opts.Reader.Values(c.snap.ChangeSet)
    if err != nil {
        return err
    }

    return nil
}

func (c *config) run() {
    watch := func(w loader.Watcher) error {
        for {
            // get changeset
            snap, err := w.Next()
            if err != nil {
                return err
            }

            c.Lock()

            if c.snap.Version >= snap.Version {
                c.Unlock()
                continue
            }

            // save
            c.snap = snap

            // set values
            c.vals, _ = c.opts.Reader.Values(snap.ChangeSet)

            c.Unlock()
        }
    }

    for {
        w, err := c.opts.Loader.Watch()
        if err != nil {
            time.Sleep(time.Second)
            continue
        }

        done := make(chan bool)

        // the stop watch func
        go func() {
            select {
            case <-done:
            case <-c.exit:
            }
            w.Stop()
        }()

        // block watch
        if err := watch(w); err != nil {
            // do something better
            time.Sleep(time.Second)
        }

        // close done chan
        close(done)

        // if the config is closed exit
        select {
        case <-c.exit:
            return
        default:
        }
    }
}
```

看看`Init()`做了什么

1. 初始化并设置opts，创建exit用于监听退出信号，设置opts

2. 设置默认loader，c.opts.Loader默认是memory `memory.NewLoader()`[config/loader/memory/memory.go]

   1. 初始化并设置opts，包含Reader[默认json]
   2. 初始化memory{}
   3. 设置m.sets,并`watch()`每个options.Source，看看`watch()`做了什么
      1. 定义watch()函数
         1. 调用watcher.Next(),下面看看next()做了什么
            1. 定义update()函数，返回loader.Snapshot{}
            2. 监听watcher.exit，watcher.updates信号，有更新时且版本更新时，调用上面的update()函数，更新watcher.value并返回loader.Snapshot{}
         2. 保存m.sets[idx]，值为loader.Snapshot{}
         3. 合并所有m.sets
         4. 读取所有值到m.vals,保存快照到m.snap
         5. 调用update()
            1. 获取所有watcher，如果版本有更新，则发送watcher.updates信号
      2. 调用Watch()函数返回watcher，注意W是大写,调用的是memory.Watch()
         1. 调用Get()，返回m.vals.Get(path...)
         2. 初始化watcher，并添加到m.watchers【双向链表】
         3. 开协程，监听watcher.exit信号,收到信号从watchers中移除当前watcher
      3. 开协程，监听完成信号done和exit信号，收到信号后执行Stop(),关闭exit，updates这2个channel
      4. 调用上面定义的watch()
      5. 关闭done channel，监听m.exit信号

3. 调用c.opts.Loader.Load()

   1. 循环所有source，更新m.sources,m.sets,并watch()所有source
   2. 调用`reload()`
      1. 合并所有sets
      2. 设置m.vals,m.snap
      3. 调用m.update()

4. 调用`c.opts.Loader.Snapshot()`

   1. 如已经load，直接复制一份并返回m.snap
   2. 没载入就调用`Sync()`同步配置
   3. 复制一份m.snap返回

5. 调用`c.opts.Reader.Values()`，赋值config.vals【`reader.Values`类型】

   

go-micro支持从读取配置有以下方法：

``` go
type Config interface {
	// provide the reader.Values interface
	reader.Values
	// Init the config
	Init(opts ...Option) error
	// Options in the config
	Options() Options
	// Stop the config loader/watcher
	Close() error
	// Load config sources
	Load(source ...source.Source) error
	// Force a source changeset sync
	Sync() error
	// Watch a value for changes
	Watch(path ...string) (Watcher, error)
}
```

主要使用的是

``` go
// Load config sources
	Load(source ...source.Source) error
// Watch a value for changes
	Watch(path ...string) (Watcher, error)
```

- Load:允许从各种源获取配置信息
- Watch可以监控path中文件变更，也可监控etcd中配置改动

## 实战部分

本次展示仅使用etcd读取配置，为了能够读取到数据，我们需要先往etcd中写入数据，如果你对命令不熟悉，建议使用`etcdkeeper`来添加配置

``` shell
$ etcdctl put /micro/config/demo "{ \"network\": \"172.30.0.0/16\", \"backend\": \"vxlan\"}"
OK
$ etcdctl get /micro/config/demo
/micro/config/demo #key
{ "network": "172.30.0.0/16", "backend": "vxlan"} #value
```

### 读取配置

``` go
type Demo struct {
	NetWork string `json:"netWork"`
	Backend string `json:"backend"`
}

func getConfig() Demo {
	getJSON("demo")
	conf := Demo{}
	config.Scan(&conf)
	return conf
}

func getJSON(pr string) error {
	//配置中心使用etcd key/value 模式
	etcdSource := etcd.NewSource(
		//设置配置中心地址
		etcd.WithAddress("127.0.0.1:2379"),
		//设置前缀，不设置默认为 /micro/config
		etcd.WithPrefix("/micro/config/"+pr),
		//是否移除前缀，这里设置为true 表示可以不带前缀直接获取对应配置,StripPrefix 从语义上看是去掉前缀的意思，如果没有去掉前缀，则会保留micro、etcd这两个key
		etcd.StripPrefix(true),
	)
	//加载配置
	return config.Load(etcdSource)
}
```

### 监控配置

``` go
w, err := config.Watch("/micro/config/", "demo")
if err != nil {
  return err
}
// wait for next value
v, err := w.Next()
if err != nil {
  return err
}
var demo Demo
v.Scan(&demo)
return nil
```

监控`/micro/config/`下的`demo`节点，当节点数据发生改变时，就会触发监控