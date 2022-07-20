---
title: "go-micro使用配置文件"
categories:
  - Golang
tags:
  - go-micro
last_modified_at: 2022-07-20T14:07:50-05:00
---

Go-micro config作为配置库，它也是动态的可插拔的。

应用程序中的大多数配置都是静态配置的，或者包括从多个源加载的复杂逻辑。Go Config使这变得简单、可插拔和合并。您再也不用以同样的方式处理配置了。

## 特性

* **动态加载** - 动态按时按需从多资源加载配置。Go-micro config会在后台监视配置资源，动态在内存中合并、更新。

* **可插拔资源** - 可选择从任意数量的资源中加载、合并配置，后台资源在内部被抽象成标准格式并通过编码器解码。资源可以是环境变量、参数flag、文件、etcd、k8s configmap等等。

* **可合并配置** - 假设指定了多个配置源，格式不限，它们会被合并划一。 这样大大简化了配置的优先级与环境的变动。

* **观察变动** - 可以选择观测指定配置值的变动。使用Go Config观测器热加载，可以随时查看配置值的变动情况。

* **安全修复** - 某些情况如配置加载失败或者被擦除时，可以指定回退值。这可以保证在发生事故时，我们能读取完整的默认值。

## 开始

* [Source](https://github.com/asim/go-micro/tree/v2.9.1/config/source) - 后台获取加载的位置
* [Encoder](https://github.com/asim/go-micro/tree/v2.9.1/config/encoder) - 负责处理资源配置编码、解码
* [Reader](https://github.com/asim/go-micro/tree/v2.9.1/config/reader) - 将多个编码处理后的资源合并成单一的格式

## Sources

`Source` 也即后台加载的配置，同时可以使用多资源。

官方支持以下格式：

* [cli](https://github.com/asim/go-micro/tree/v2.9.1/config/source/microcli) - 即命令行参数
* [env](https://github.com/asim/go-micro/tree/v2.9.1/config/source/env) - 从环境变量中读取
* [etcd](https://github.com/asim/go-micro/tree/v2.9.1/config/source/etcd) - 从etcd v3中读取
* [file](https://github.com/asim/go-micro/tree/v2.9.1/config/source/file) - 从配置文件读取
* [flag](https://github.com/asim/go-micro/tree/v2.9.1/config/source/flag) - 从flags中读取
* [memory](https://github.com/asim/go-micro/tree/v2.9.1/config/source/memory) - 从内存中读取
* [grpc](https://github.com/asim/go-micro/tree/v2.9.1/config/source/service) - 从grpc服务器读取

社区支持一下格式：

* [configmap](https://github.com/asim/go-micro/tree/v2.9.1/config/source/configmap) - k8s configmap
* [consul](https://github.com/asim/go-micro/tree/v2.9.1/config/source/consul) - 从consul中读取


### ChangeSet变更集

Sources以变更集的方式返回配置。对于多个后台配置，变更集是单一的内部抽象。

``` go
type ChangeSet struct {
    // Raw encoded config data
    Data      []byte
    // MD5 checksum of the data
    Checksum  string
    // Encoding format e.g json, yaml, toml, xml
    Format    string
    // Source of the config e.g file, consul, etcd
    Source    string
    // Time of loading or update
    Timestamp time.Time
}
```

## Encoder

`Encoder` 负责资源配置编码、解码。后台资源可能会存在不同的格式，编码器负责处理不同的格式，默认的格式是Json。

编码器支持以下格式：

* json
* yaml
* toml
* xml
* hcl

## Reader

`Reader` 负责把多个changeset集合并成一个可查询的值集。

``` go
type Reader interface {
    // Merge multiple changeset into a single format
    Merge(...*source.ChangeSet) (*source.ChangeSet, error)
    // Return return Go assertable values
    Values(*source.ChangeSet) (Values, error)
    // Name of the reader e.g a json reader
    String() string
}
```

读取器使用编码器将更改集解码为 `map[string]interface{}`Values, 然后将它们合并到单个更改集中。它通过 Format 字段以确定编码器。然后更改集表示为一组，具有重新备份 Go 类型和无法加载值的回退功能.


```go

// Values is returned by the reader
type Values interface {
    // Return raw data
        Bytes() []byte
    // Retrieve a value
        Get(path ...string) Value
    // Return values as a map
        Map() map[string]interface{}
    // Scan config into a Go type
        Scan(v interface{}) error
}
```

`Value` 接口支持使用构建、类型断言转化成go类型的值，默认使用回退值。

``` go
type Value interface {
    Bool(def bool) bool
    Int(def int) int
    String(def string) string
    Float64(def float64) float64
    Duration(def time.Duration) time.Duration
    StringSlice(def []string) []string
    StringMap(def map[string]string) map[string]string
    Scan(val interface{}) error
    Bytes() []byte
}
```

## Config

`Config` 管理所有配置、抽象后的资源、编码器及reader。

读取、同步、监视多个后台资源，把资源合并成单一集合以供查询。

``` go
// Config is an interface abstraction for dynamic configuration
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

## 使用

- 文件中读取配置
- 新配置
- 加载文件
- 读取配置
- 读取值
- 路径监控
- 多源
- 设置源编码器
- 添加读取器编码器

### 文件中读取配置

文件源从文件中读取配置。

它使用文件扩展名来确定格式，例如`config.yaml`具有yaml格式。它不使用编码器或插入文件数据。如果不存在文件扩展名，源格式将默认为选项中的编码器。

`json`配置示例:

``` json
{
    "database": {
        "address": "10.0.0.1",
        "port": 3306
    }
}
```

### 新增配置

指定文件源和文件路径。路径是可选的，默认为`config.json`

新增配置（直接使用默认的配置对象也可）

``` go
fileSource := file.NewSource(
	file.WithPath("/tmp/config.json"),
)
```

### 文件格式

要加载不同的文件格式，例如yaml、toml、xml，只需用它们的扩展名指定它们

``` go
fileSource := file.NewSource(
        file.WithPath("/tmp/config.yaml"),
)
```
如果您想指定一个没有扩展名的文件，请确保将编码器设置为相同的格式

``` go
e := toml.NewEncoder()

fileSource := file.NewSource(
        file.WithPath("/tmp/config"),
	source.WithEncoder(e),
)
```

### 加载配置文件

将源加载到配置中

``` go
// Create new config
conf := config.NewConfig()

// Load file source
conf.Load(fileSource)
```

### 读取多个值

如果将配置写入结构

```go
type Host struct {
    Address string `json:"address"`
    Port int `json:"port"`
}

var host Host

config.Get("hosts", "database").Scan(&host)

// 10.0.0.1 3306
fmt.Println(host.Address, host.Port)
```

读取独立的值

```go
// Get address. Set default to localhost as fallback
address := config.Get("hosts", "database", "address").String("localhost")

// Get port. Set default to 3000 as fallback
port := config.Get("hosts", "database", "port").Int(3000)
```

### 监控目录

观测目录的变化。当文件有改动时，新值便可生效。

```go
w, err := config.Watch("hosts", "database")
if err != nil {
    // do something
}

// wait for next value
v, err := w.Next()
if err != nil {
    // do something
}

var host Host

v.Scan(&host)
```

### 多资源

可以加载和合并多个源。合并优先级顺序相反.

```go
config.Load(
    // base config from env
    env.NewSource(),
    // override env with flags
    flag.NewSource(),
    // override flags with file
    file.NewSource(
        file.WithPath("/tmp/config.json"),
    ),
)
```

### 设置资源编码器

资源需要编码器才能将配置编码与解码成所需的changeset格式。

默认编码器是JSON格式，也可以使用yaml、xml、toml等选项。

```go
e := yaml.NewEncoder()

s := consul.NewSource(
    source.WithEncoder(e),
)
```

### 新增Reader编码器

Reader使用各种编码器来解码不同格式源的数据。

默认的Reader支持json、yaml、xml、toml、hcl，合并配置后会把其转成json格式。

也可指定给其特定的编码器：

```go
e := yaml.NewEncoder()

r := json.NewReader(
    reader.WithEncoder(e),
)
```

