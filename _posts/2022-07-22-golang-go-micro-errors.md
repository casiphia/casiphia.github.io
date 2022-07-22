---
title: "go-micro框架定义接口错误返回"
categories:
  - Golang
tags:
  - go-micro
last_modified_at: 2022-07-22T11:10:50-05:00
---

`go-micro`为分布式系统中发生的大多数事物包括错误提供了抽象和类型。通过提供一组核心错误和定义详细错误类型的能力，我们可以始终如一地了解典型 Go 错误字符串之外发生的情况.

官方默认有grpc的errors返回值类型，基本上可以满足业务需求，但是当你需要自定义grpc接口时，这边文章可能对你有所帮助。

## 阅读源码

官方`errors`返回类型：

```go
syntax = "proto3";

package errors;

message Error {
  string id = 1;
  int32 code = 2;
  string detail = 3;
  string status = 4;
};
```

>id:系统级错误码
>
>code:业务相关响应码
>
>detail:错误详情
>
>status:状态

官方默认已经支持`errors`类型的序列化以及`New`的方法

``` go
type Error struct {
    Id     string `json:"id"`
    Code   int32  `json:"code"`
    Detail string `json:"detail"`
    Status string `json:"status"`
}
```

``` go
//go:generate protoc -I. --go_out=paths=source_relative:. errors.proto

func (e *Error) Error() string {
	b, _ := json.Marshal(e)
	return string(b)
}

// New generates a custom error.
func New(id, detail string, code int32) error {
	return &Error{
		Id:     id,
		Code:   code,
		Detail: detail,
		Status: http.StatusText(int(code)),
	}
}
```

在系统中，要求您从处理程序返回错误或从客户端接收错误的任何位置，都应假定其为 go-micro 错误，或者应该生成错误。默认情况下，我们返回 errors.InternalServerError, 其中某些问题在内部出错及发现超时错误 errors.Timeout.
官方已经提供了*400*、*401*、*403*、*404*、*405*、*408*、*409*、*500*的错误定义

让我们假设处理程序中发生了一些错误。然后，您应该决定返回哪种错误，并执行以下操作.

假设提供的某些数据无效

```go
return errors.BadRequest("com.example.srv.service", "invalid field")
```

如果发生内部错误

```go
if err != nil {
    return errors.InternalServerError("com.example.srv.service", "failed to read db: %v", err.Error())
}
```

现在，假设您从客户端收到一些错误

``` go
pbClient := pb.NewGreeterService("go.micro.srv.greeter", service.Client())
rsp, err := pb.Client(context, req)
if err != nil {
    // parse out the error
    e := errors.Parse(err.Error())

    // inspect the value
    if e.Code == 401 {
        // unauthorised...
    }
}
```

## 自定义错误类型

`errors.proto`定义如下

``` go
syntax = "proto3";

package errors;

message Errors {
  int32 code = 1;
  string detail = 2;
};
```

`errors.go`定义如下：

```go
type Error errors.Errors

func (e *Error) Error() string {
	b, _ := json.Marshal(e)
	return string(b)
}

// New generates a custom error.
func New(code int32, err error) error {
	return &Error{
		Code:   code,
		Detail: err.Error(),
	}
}
```

客户端调用

```go
// FromError try to convert go error to *Error
func FromError(err error) *Error {
	if verr, ok := err.(*Error); ok && verr != nil {
		return verr
	}

	return Parse(err.Error())
}
```

通过`FromError`方法解析errors结构，通过响应的code做不同的业务判断

> 这里有个坑，自定义的错误类型只能在官方的基础下修改，不能自定义字段，不够灵活，因为go-micro官方的包之间都是使用这个错误类型做判断的，所以没有办法深度自定义类型

好了，以上就是本文的主要内容。

老规矩，代码已经上传到Github，欢迎访问： [https://github.com/casiphia/go-micro-examples](https://github.com/casiphia/go-micro-examples)


