---
title: "Golang 设计模式之创建者模式"
categories:
  - Golang
tags:
  - Go Design Pattern
last_modified_at: 2024-09-05T11:28:50-05:00
---

## 1. 简介

创建者模式（Builder Pattern） 是一种用于构建复杂对象的设计模式。通过将对象的构造过程与对象的表示分离，创建者模式使得我们能够一步一步地构建对象。创建者模式特别适合当对象有很多可选参数或者构建过程非常复杂的情况。

在 Go 语言中，虽然没有像 Java 那样的构造函数重载机制，但我们可以通过使用创建者模式来更好地管理对象的创建过程。

## 2. 使用场景

创建者模式在以下场景中非常有用：

- 需要构建复杂的对象，且该对象由多个部分组成。
- 对象的构建过程需要多步操作，而不是一蹴而就。
- 需要根据不同需求创建不同表示形式的对象。

例如，在构建复杂的配置文件、创建定制化的产品对象，或者需要逐步构建对象时，创建者模式可以有效地简化这些过程。

## 3. 代码示例

我们以构建一台电脑为例，电脑有多个组件（CPU、GPU、RAM、硬盘等），而这些组件可以自由组合或选择。这正是创建者模式的一个理想应用场景。

```go
package main

import "fmt"

// Computer 是我们要创建的复杂对象
type Computer struct {
    CPU  string
    GPU  string
    RAM  string
    Disk string
}

// Builder 是创建者接口，定义了创建过程
type Builder interface {
    SetCPU(string) Builder
    SetGPU(string) Builder
    SetRAM(string) Builder
    SetDisk(string) Builder
    Build() Computer
}

// ConcreteBuilder 是具体的创建者
type ComputerBuilder struct {
    cpu  string
    gpu  string
    ram  string
    disk string
}

func NewComputerBuilder() *ComputerBuilder {
    return &ComputerBuilder{}
}

func (b *ComputerBuilder) SetCPU(cpu string) Builder {
    b.cpu = cpu
    return b
}

func (b *ComputerBuilder) SetGPU(gpu string) Builder {
    b.gpu = gpu
    return b
}

func (b *ComputerBuilder) SetRAM(ram string) Builder {
    b.ram = ram
    return b
}

func (b *ComputerBuilder) SetDisk(disk string) Builder {
    b.disk = disk
    return b
}

func (b *ComputerBuilder) Build() Computer {
    return Computer{
        CPU:  b.cpu,
        GPU:  b.gpu,
        RAM:  b.ram,
        Disk: b.disk,
    }
}

func main() {
    // 创建电脑
    gamingPC := NewComputerBuilder().
        SetCPU("Intel i9").
        SetGPU("NVIDIA RTX 3090").
        SetRAM("32GB").
        SetDisk("1TB SSD").
        Build()

    fmt.Printf("Gaming PC: %+v\n", gamingPC)

    officePC := NewComputerBuilder().
        SetCPU("Intel i5").
        SetRAM("16GB").
        SetDisk("512GB SSD").
        Build()

    fmt.Printf("Office PC: %+v\n", officePC)
}
```

### 3.2 单元测试

在实际项目中，单元测试可以帮助我们确保创建者模式实现的正确性。我们可以测试不同配置下的电脑对象创建。

```go
package main

import (
    "reflect"
    "testing"
)

func TestComputerBuilder(t *testing.T) {
    // 测试游戏电脑配置
    expectedGamingPC := Computer{
        CPU:  "Intel i9",
        GPU:  "NVIDIA RTX 3090",
        RAM:  "32GB",
        Disk: "1TB SSD",
    }

    gamingPC := NewComputerBuilder().
        SetCPU("Intel i9").
        SetGPU("NVIDIA RTX 3090").
        SetRAM("32GB").
        SetDisk("1TB SSD").
        Build()

    if !reflect.DeepEqual(gamingPC, expectedGamingPC) {
        t.Errorf("Expected %+v, but got %+v", expectedGamingPC, gamingPC)
    }

    // 测试办公电脑配置
    expectedOfficePC := Computer{
        CPU:  "Intel i5",
        RAM:  "16GB",
        Disk: "512GB SSD",
    }

    officePC := NewComputerBuilder().
        SetCPU("Intel i5").
        SetRAM("16GB").
        SetDisk("512GB SSD").
        Build()

    if !reflect.DeepEqual(officePC, expectedOfficePC) {
        t.Errorf("Expected %+v, but got %+v", expectedOfficePC, officePC)
    }
}
```

## 4. 总结

创建者模式 提供了一种优雅的方式来构建复杂的对象，尤其是在对象有多个可选配置的情况下。它将对象的构造与表示分离，使得客户端代码更加简洁，并且更容易进行对象的定制。

优点：

- 可以更清晰地管理复杂对象的构建过程。
- 易于扩展和维护。
- 构建过程可以根据需要进行定制，而不会影响对象的最终表示。

缺点：

- 如果对象很简单，使用创建者模式可能会增加不必要的复杂性。
- 增加了额外的构建器类，可能会增加项目的代码量。

在实际应用中，创建者模式广泛用于生成复杂对象，如配置文件、UI组件，或者涉及多步构建的对象。它为我们提供了一种灵活、清晰的对象构造方式，尤其适合对象的属性较多且组合较为复杂的场景。
