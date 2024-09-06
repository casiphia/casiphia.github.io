---
title: "Golang 设计模式之适配器模式"
categories:
  - Golang
tags:
  - Go Design Pattern
last_modified_at: 2024-09-08T11:28:50-05:00
---

## 简介

**适配器模式（Adapter Pattern）** 是一种结构型设计模式，适用于将一个类的接口转换为客户端希望的另一种接口。适配器模式让原本接口不兼容的类可以合作无间。

适配器模式的核心思想是通过引入一个适配器类，使得原本不能一起工作的类能够协同工作。该模式非常适合在使用第三方库、遗留代码或不同系统模块之间进行集成时使用。

## 应用场景

适配器模式主要用于以下场景：

1. **兼容老旧代码**：当你需要复用现有代码（例如遗留系统），但接口不兼容时，可以使用适配器来进行接口的转换。
2. **集成第三方库**：当你需要将第三方库引入项目，而其接口与当前系统接口不一致时，可以使用适配器模式进行适配。
3. **接口统一**：当你希望对不同的类提供统一的接口时，可以通过适配器来实现。

### 实际应用场景

假设你在开发一个支付系统，其中已经有了一个旧版的支付接口 `OldPaymentSystem`，但现在你需要引入一个新的第三方支付库 `NewPaymentSystem`。你不想改变现有代码，因此可以使用适配器模式将 `NewPaymentSystem` 适配为旧的支付接口。

## 示例代码

### 旧版支付系统

```go
package main

import "fmt"

// OldPaymentSystem 是老版的支付系统接口
type OldPaymentSystem interface {
    ProcessPayment(amount float64) string
}

// OldPaySystem 是旧版支付系统的具体实现
type OldPaySystem struct{}

func (ops *OldPaySystem) ProcessPayment(amount float64) string {
    return fmt.Sprintf("Processed payment of $%.2f using the old system.", amount)
}
```

### 新版支付系统

```go
// NewPaymentSystem 是新版的支付系统
type NewPaymentSystem struct{}

func (nps *NewPaymentSystem) MakePayment(amount float64) string {
    return fmt.Sprintf("Processed payment of $%.2f using the new system.", amount)
}
```

### 适配器

```go
// PaymentAdapter 是用于将新支付系统适配为老版支付系统的适配器
type PaymentAdapter struct {
    newPaymentSystem *NewPaymentSystem
}

// ProcessPayment 实现了旧版支付系统的接口，实际上是调用新版支付系统的 MakePayment 方法
func (adapter *PaymentAdapter) ProcessPayment(amount float64) string {
    return adapter.newPaymentSystem.MakePayment(amount)
}
```

### 使用示例

```go
func main() {
    // 创建一个旧版支付系统的实例
    oldSystem := &OldPaySystem{}
    fmt.Println(oldSystem.ProcessPayment(100.0))

    // 创建一个新版支付系统的实例
    newSystem := &NewPaymentSystem{}

    // 使用适配器将新版支付系统适配为老版支付系统
    adapter := &PaymentAdapter{newPaymentSystem: newSystem}
    fmt.Println(adapter.ProcessPayment(100.0))
}
```

### 代码解释

- OldPaymentSystem 接口：代表旧版支付系统的接口，包含 ProcessPayment 方法。
- OldPaySystem 结构体：实现了旧版支付系统的接口，处理支付请求。
- NewPaymentSystem 结构体：这是新的支付系统，它的接口与旧系统不同，使用 MakePayment 方法来处理支付。
- PaymentAdapter 适配器：这是关键部分。它将新版的 NewPaymentSystem 适配为 OldPaymentSystem，通过 ProcessPayment 调用 MakePayment。
- main 函数：展示了如何使用适配器模式，在不改变旧系统代码的情况下，集成新版支付系统。

### 图解

``` go
+-------------+        +---------------------+        +-------------------+
|   Client    | -----> |    OldPaymentSystem  | -----> |  PaymentAdapter    |
+-------------+        +---------------------+        +-------------------+
                                                               |
                                                      +-------------------+
                                                      |  NewPaymentSystem  |
                                                      +-------------------+
```

在这个图中，客户端通过 OldPaymentSystem 接口与系统交互，适配器 PaymentAdapter 实现了 OldPaymentSystem 的接口，并将其请求转发给新版的 NewPaymentSystem，从而实现了兼容性

## 单元测试

以下是针对适配器模式的单元测试，确保适配器能够正确适配新版支付系统。

```go
package main

import "testing"

func TestOldSystem(t *testing.T) {
    oldSystem := &OldPaySystem{}
    result := oldSystem.ProcessPayment(100.0)
    expected := "Processed payment of $100.00 using the old system."
    
    if result != expected {
        t.Errorf("Expected %s but got %s", expected, result)
    }
}

func TestPaymentAdapter(t *testing.T) {
    newSystem := &NewPaymentSystem{}
    adapter := &PaymentAdapter{newPaymentSystem: newSystem}
    result := adapter.ProcessPayment(100.0)
    expected := "Processed payment of $100.00 using the new system."

    if result != expected {
        t.Errorf("Expected %s but got %s", expected, result)
    }
}
```

### 测试解释

- TestOldSystem：测试旧版支付系统是否能够正常处理支付请求。
- TestPaymentAdapter：测试适配器是否能够正确将旧版接口适配为新版支付系统，并且调用了正确的方法。

## 总结

适配器模式 通过转换一个类的接口，使得它能够与客户端期望的接口兼容，从而解决了接口不兼容的问题。它常用于系统集成、老旧代码复用和引入新库的场景中。

优点：

- 接口转换：适配器模式让原本不兼容的类可以协同工作。
- 复用现有代码：在不修改原有类的前提下，通过适配器可以复用现有代码。
- 解耦客户端与第三方库：适配器模式可以将客户端与第三方库解耦，使得客户端只需要与适配器交互。

缺点：

- 增加系统复杂性：适配器模式引入了额外的适配器类，可能增加系统的复杂性。
- 性能开销：每次调用时，适配器需要将请求转换为适配的接口，可能带来一些性能开销。

适配器模式在系统集成、旧代码兼容以及第三方库引入中发挥了重要作用。通过引入适配器，系统可以更加灵活、易于扩展和维护。
