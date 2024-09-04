---
title: "Golang 设计模式之工厂方法模式"
categories:
  - Golang
tags:
  - Go Design Pattern
last_modified_at: 2024-09-03T17:28:50-05:00
---

## 1. 简介

工厂方法模式（Factory Method Pattern）是一种创建型设计模式，它提供了一个创建对象的接口，但由子类决定要实例化的类是哪一个。工厂方法让一个类的实例化延迟到其子类。

在 Golang 中，工厂方法模式通常用于解耦代码与特定实现之间的依赖关系，从而使代码更具可扩展性和灵活性。

## 2. 使用场景

使用工厂方法模式的典型场景包括：

- **日志记录框架**：不同的日志级别可能需要不同的日志记录方式。
- **数据库连接池**：根据不同的数据库类型（如 MySQL、PostgreSQL）创建相应的连接对象。
- **消息处理**：不同类型的消息需要不同的处理器来处理。

## 3. 代码示例

### 3.1 工厂方法模式的基本实现

我们将以创建不同类型的 **汽车** 为例，演示如何在 Go 中使用工厂方法模式。

首先，我们定义一个 `Car` 接口，它具有 `Drive` 方法：

```go
// Car 是所有汽车类型都要实现的接口
type Car interface {
    Drive() string
}
```

然后，我们定义两个具体的 `Car` 实现：`Sedan` 和 `SUV`。

```go
// Sedan 代表轿车类型
type Sedan struct{}

func (s *Sedan) Drive() string {
    return "Driving a sedan"
}

// SUV 代表运动型多用途车
type SUV struct{}

func (s *SUV) Drive() string {
    return "Driving an SUV"
}
```

接下来，我们创建一个工厂接口 `CarFactory` 和两个具体的工厂 `SedanFactory` 和 `SUVFactory`：

```go
// CarFactory 是所有工厂类型要实现的接口
type CarFactory interface {
    CreateCar() Car
}

// SedanFactory 是用于创建 Sedan 的工厂
type SedanFactory struct{}

func (f *SedanFactory) CreateCar() Car {
    return &Sedan{}
}

// SUVFactory 是用于创建 SUV 的工厂
type SUVFactory struct{}

func (f *SUVFactory) CreateCar() Car {
    return &SUV{}
}
```

在实际使用中，我们可以根据需要使用具体的工厂来创建汽车：

```go
func main() {
    // 使用 Sedan 工厂创建 Sedan
    sedanFactory := &SedanFactory{}
    sedan := sedanFactory.CreateCar()
    fmt.Println(sedan.Drive()) // Output: Driving a sedan

    // 使用 SUV 工厂创建 SUV
    suvFactory := &SUVFactory{}
    suv := suvFactory.CreateCar()
    fmt.Println(suv.Drive()) // Output: Driving an SUV
}
```

### 3.2 单元测试

为了验证工厂方法的正确性，我们可以编写一些单元测试：

```go
package main

import (
    "testing"
)

func TestSedanFactory_CreateCar(t *testing.T) {
    factory := &SedanFactory{}
    car := factory.CreateCar()

    if car.Drive() != "Driving a sedan" {
        t.Errorf("Expected 'Driving a sedan', but got '%s'", car.Drive())
    }
}

func TestSUVFactory_CreateCar(t *testing.T) {
    factory := &SUVFactory{}
    car := factory.CreateCar()

    if car.Drive() != "Driving an SUV" {
        t.Errorf("Expected 'Driving an SUV', but got '%s'", car.Drive())
    }
}
```

## 4. 总结

工厂方法模式在 Go 语言中可以用于多种场景，包括依赖注入、测试代码以及需要根据具体情况生成不同对象的场合。通过使用工厂方法模式，我们可以更好地管理代码的扩展性和可维护性。

希望这篇文章对你理解工厂方法模式有所帮助！如果你有任何问题或想法，欢迎留言交流。
