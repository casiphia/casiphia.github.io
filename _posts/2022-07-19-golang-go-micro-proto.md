---
title: "proto3默认值与可选项"
categories:
  - Golang
tags:
  - go-micro
last_modified_at: 2022-07-19T17:28:50-05:00
---

目前开发的产品架构采用微服务架构，微服务之间通信的消息格式则使用的proto3标准协议格式。

### proto介绍

全称Protocol Buffers(下面简称PB)是Google公司开发的一种数据描述语言，是一种类似XML但更灵活和高效的结构化数据存储格式，可用于结构化数据的序列化，适用于数据存储、RPC数据交换格式。它可用于通讯协议、数据存储等领域的语言无关、平台无关、可扩展的序列化结构数据格式。它支持多种语言，比如C++，Java，C#，Python，JavaScript等等。目前它的最新版本是3.18.0。

### proto优点

从上面的proto介绍不难得出其具备下面几个优点：

1. 描述简单，对开发人员友好
2. 跨平台、跨语言，不依赖于具体运行平台和编程语言
3. 高效自动化解析和生成
4. 压缩比例高
5. 可扩展、兼容性好

### proto3特性

proto3相较于proto2支持更多语言但在语法上更为简洁。去除了一些复杂的语法和特性，更强调约定而弱化语法。

1. 删除原始值字段的presence字段逻辑，删除required字段以及删除默认值。这使得proto3更容易实现如在Android Java，Objective C或Go等语言中的开放式结构化表示。
2. 移除unknown关键字.
3. 去掉extensions类型，使用Any新标准类型替换。
4. 针对未知枚举值的固定语法.
5. 增加maps(主要指代码生成支持map)
6. 添加一组用于表示时间，动态数据等的标准类型。
7. 替换二进制编码的明确JSON编码

## 问题提出

不可否认由于proto3在语法上进行了大量简化，使得proto格式无论是在友好性上、还是灵活性上都有了大幅提升。但是由于删除了presence、required及默认值这些内容，导致proto结构中的所有字段都成了optional（可选字段）类型。这在实际使用过程出现了如下问题：

1. 结构化数据缺失、显示不全，默认值都当成了不存在（not present)。对外提供的数据上报时，不便于对数据的分析和使用。对内服务调试时，不便于问题跟踪和定位;
2. 无法验证业务逻辑上数据构造的正确性，如果是默认值不清楚数据构造时到底是否赋过值。

## 问题描述

openAPI调用RPC微服务的接口返回字段应该与接口文档一致

#### 正确返回

```json
{
  "code": 200,
  "cnMsg": "操作成功",
  "enMsg": "Success",
  "data": {
    "nonce": "",
    "isBind": false
  }
}
```

> 接口文档

#### 错误返回

```json
{
  "code": 200,
  "cnMsg": "操作成功",
  "enMsg": "Success",
  "data": {}
}
```

> Openapi

api文档需要返回`nonce`和`isBind`字段，但是openApi返回缺少这两个字段

### 问题追踪

通过手动调用Grpc测试发现，grpc接口返回的空值时忽略了此字段

```json
{
  "code": 200,
  "cnMsg": "操作成功",
  "enMsg": "Success",
  "data": {}
}
```

> Grpc接口调用

## 问题解决

### 1.wrappers方案

#### 方案介绍

经过研究发现google已经意识到这个问题，采取了一些补救方法—提供wrappers包。该包位于github.com/golang/protobuf/ptypes/wrappers/wrappers.proto，proto文件中包含以下消息类型：

1. DoubleValue
2. FloatValue
3. Int64Value
4. UInt64Value
5. Int32Value
6. UInt32Value
7. BoolValue
8. StringValue
9. BytesValue

以Int32Value为例，其包装方法如下：

```protobuf
// 登陆响应
message RspLogin {
	string nonce = 1; //随机值
	bool isBind = 2; //是否绑定邮箱:true绑定,false未绑定
}
```

#### 示例

```protobuf
import "google/protobuf/wrappers.proto";

message Test {
  google.protobuf.StringValue nonce = 1;
  google.protobuf.BoolValue nonce = 2;
}
```

#### Handler处理

```go
import "google.golang.org/protobuf/types/known/wrapperspb"

rsp.Data.IsBind = wrapperpb.Bool(false)
rsp.Data.Nonce = wrapperpb.String("")
```

#### 优缺点：

**优点：** 描述简洁清晰

**缺点：** 使用该方案会使结构变大，每在一个字段使用都会增加2个字节。需要修改左右handler方法，openapi的swag不能识别`FloatValue`类型

### 2.oneof方案

#### 方案介绍

另外还可以使用oneof来达到目的，oneof与数据结构联合体（UNION）有点类似，一次最多只有一个字段有效，一般是为了节省存储空间。针对本文所遇到的问题则是将需要处理的字段通过oneof进行包装。

#### 示例

```protobuf
// 登陆响应
message RspLogin {
  oneof a_oneof {
		string nonce = 1; //随机值
		bool isBind = 2; //是否绑定邮箱:true绑定,false未绑定
	}
}
```

可以使用test.getAOneofCase()来检查a是否被设置。

#### 优缺点

**优点：** 向后兼容proto2 

**缺点：** 不能使用在repeated类型字段,需要改动handler方法

### 3.自定义nullable方案

#### 方案介绍

该方案主要通过实现一个自定义的nullable关键字来解决字段是否能够为空的问题。 修改详细可参考： https://github.com/criteo-forks/protobuf/commit/8298aff178ccffd0c7c99806e714d0f14f40faf8

#### 示例

```protobuf
// 登陆响应
message RspLogin {
		nullable nonce = 1; //随机值
		nullable isBind = 2; //是否绑定邮箱:true绑定,false未绑定
}
```

#### 优缺点

**优点：** 描述简洁清晰 **缺点：**

1. 自定义的关键字不利于版本升级更新
2. 需要修改源代码
3. 与proto3版本的本意冲突（使得proto语义复杂化）

### 4.map方案

#### 方案介绍

还有人通过map<int32,bool>来解决问题，每当一个字段被设置时，bool值则被设置为true（默认值也是）。另外如果设置的不是默认值时，还需要在每个字段的setter方法中增加hasXXX方法。详情参见：https://github.com/google/protobuf/issues/2684

#### 示例

```
message Test {
  map<string, string> data = 1;
}
```

#### 优缺点

**优点：**

1. 结构的增大会比wrapper方案小得多

2. 向后兼容proto2

**缺点：**

1. 写法不够简介

2. map的key值只能是整形和字符串

3. 需要修改setter的实现

4. 对外提供接口不明确

### 5.jsonpb方案

#### 方案介绍

使用`jsonpb`讲pb消息序列化成byte提供外部使用

##### 示例：

```go
import "github.com/gogo/protobuf/jsonpb"
...
m := jsonpb.Marshaler{EmitDefaults: true}
var buf bytes.Buffer
m.Marshal(&buf, rsp.Data)
```

返回

```
data:{\"nonce\": \"\",\"isBind\": false}
```

#### 优缺点

**优点：**

1. 统一消息返回string

**缺点：**

1. 返回[]byte不够友好
2. openapi序列化后不能解决问题
3. 前端需要序列化数据
4. 不能根本解决问题

### 6.手动修改proto

#### 方案介绍

问题的根源在与struct的tag中json为`omitempty`,导致json序列化的时候字段为空则忽略字段

##### 示例：

```protobuf
Nonce                string   `protobuf:"bytes,1,opt,name=nonce,proto3" json:"nonce, omitempty"`
IsBind               bool     `protobuf:"varint,2,opt,name=isBind,proto3" json:"isBind, omitempty"`
```

手动删除json中的`omitempty`字段

#### 优缺点

**优点：**

1. 改动小，仅需要改动proto文件

**缺点：**

1. 兼容性差
2. 不灵活，每次对proto改动都需要手动修改

### 7.手动修改json包

#### 方案介绍

最后使用的方法是复制了 `encoding/json` 库的源码到新的库 `my_json`，修改[这一行](https://github.com/golang/go/blob/release-branch.go1.9/src/encoding/json/encode.go#L1156)中的 `omitEmpty` 为 `false`。当需要忽略 `omitempty`时，使用 `my_json` 库即可

##### 示例：

```go
fields = append(fields, fillField(field{
    name:      name,
    tag:       tagged,
    index:     index,
    typ:       ft,
    omitEmpty: opts.Contains("omitempty"), // 改为 false
    quoted:    quoted,
}))
```

#### 优缺点

**优点：**

1. 兼容性好

**缺点：**

1. 不灵活
2. 对现有项目改动最大

### 8.手动修改protoc-gen-go

#### 方案介绍

既然根源在于json的tag标签，那么从proto生成方法入手，追踪protoc-gen-go代码发现如下

##### 示例：

```go
//tag := fmt.Sprintf("protobuf:%s json:%q", g.goTag(message, field, wiretype), jsonName+",omitempty")
tag := fmt.Sprintf("protobuf:%s json:%q", g.goTag(message, field, wiretype), jsonName)
```

注释生成``omitempty`标签代码,使用`go install`重新安装工具

#### 优缺点

**优点：**

1. 对现有项目不需要任何改动
2. grpc接口之间相互调用忽略空字段
3. openapi接口对前端显式提供字段为空

**缺点：**

1. 不灵活，对煸一会环境依赖大

## 总结

结合各种因素，最终采用手动修改`protoc-gen-go`工具。

## 参考文献：

1. https://github.com/google/protobuf/issues/1606
2. https://groups.google.com/forum/#!topic/protobuf/6eJKPXXoJ88

