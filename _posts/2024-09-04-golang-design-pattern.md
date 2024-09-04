---
title: "Golang 设计模式之简单工厂模式"
categories:
  - Golang
tags:
  - Go Design Pattern
last_modified_at: 2024-09-04T11:28:50-05:00
---

## 1. 简介

简单工厂模式（Simple Factory Pattern）是一种创建型设计模式，它提供了一个创建对象的接口，而不需要指定具体要实例化的类。简单工厂模式通常用于创建相似类型的对象，通过使用一个单一的工厂方法，客户端可以请求工厂生成一个实例，而不必知道实际的类名称。

在Golang中，简单工厂模式可以通过函数或结构体来实现，主要目的是减少客户端代码和对象创建过程之间的耦合。

## 2. 使用场景

使用简单工厂模式的典型场景包括：

- **日志记录系统**：需要根据日志类型（如文件日志、控制台日志、远程日志等）生成不同的日志处理器对象。。
- **支付系统**：根据支付方式（如支付宝、微信支付、银行卡支付）创建对应的支付处理器对象。
- **消息通知系统**：根据通知方式（如短信通知、邮件通知、Push通知）生成不同的通知处理器对象。

## 3. 代码示例

### 3.1 简单工厂模式的基本实现

下面是一个关于简单工厂模式的Golang实现示例，其中我们实现了一个图形工厂来创建不同的几何形状对象（如圆形和正方形）。

```go
package main

import (
    "fmt"
)

// Shape 是一个接口，定义了所有形状的行为
type Shape interface {
    Draw() string
}

// Circle 实现了 Shape 接口
type Circle struct{}

func (c Circle) Draw() string {
    return "Drawing a Circle"
}

// Square 实现了 Shape 接口
type Square struct{}

func (s Square) Draw() string {
    return "Drawing a Square"
}

// ShapeFactory 是一个简单工厂，用于创建 Shape 对象
type ShapeFactory struct{}

func (f ShapeFactory) CreateShape(shapeType string) Shape {
    switch shapeType {
    case "circle":
        return Circle{}
    case "square":
        return Square{}
    default:
        return nil
    }
}

func main() {
    factory := ShapeFactory{}

    shape1 := factory.CreateShape("circle")
    fmt.Println(shape1.Draw())

    shape2 := factory.CreateShape("square")
    fmt.Println(shape2.Draw())
}
```

在这个示例中，ShapeFactory 通过 CreateShape 方法根据 shapeType 参数来生成不同的形状对象。

### 3.2 单元测试

下面是对应的单元测试代码，用于验证工厂方法的正确性：

```go
package main

import (
    "testing"
)

func TestShapeFactory(t *testing.T) {
    factory := ShapeFactory{}

    shape1 := factory.CreateShape("circle")
    if shape1 == nil || shape1.Draw() != "Drawing a Circle" {
        t.Errorf("Expected 'Drawing a Circle', got '%v'", shape1.Draw())
    }

    shape2 := factory.CreateShape("square")
    if shape2 == nil || shape2.Draw() != "Drawing a Square" {
        t.Errorf("Expected 'Drawing a Square', got '%v'", shape2.Draw())
    }

    shape3 := factory.CreateShape("triangle")
    if shape3 != nil {
        t.Errorf("Expected nil, got '%v'", shape3)
    }
}
```

在这个单元测试中，我们分别测试了 circle、square 和一个无效类型 triangle 的生成过程。测试用例确保工厂生成的对象符合预期。

## 4. 总结

简单工厂模式是创建型设计模式中的一种，通过工厂类封装了对象的创建过程，从而将对象的创建与使用分离。它适用于需要根据不同条件创建不同实例的场景，可以有效减少代码的耦合度，提高代码的可维护性和可扩展性。

在Golang中，简单工厂模式可以通过函数或结构体实现，通过条件判断创建不同的对象实例。本文中，我们以图形工厂为例，展示了如何使用简单工厂模式来创建不同的几何形状对象，并通过单元测试验证了工厂的正确性。

尽管简单工厂模式在某些场景下非常实用，但它也存在一定的局限性，如当产品种类过多时，工厂类的复杂度也会相应增加。此外，如果新增产品类型，需要修改工厂类的代码，这可能违反开闭原则。因此，在实际开发中，需要根据具体需求选择合适的设计模式。

通过学习和使用简单工厂模式，你可以更好地组织代码结构，简化对象创建过程，并提高系统的灵活性。

这样总结了简单工厂模式的优点及其适用场景，并提到它的局限性，为读者提供了全面的理解。如果有任何进一步的需求，欢迎提出！
