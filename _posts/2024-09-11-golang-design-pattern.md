---
title: "Golang 设计模式之代理模式"
categories:
  - Golang
tags:
  - Go Design Pattern
last_modified_at: 2024-09-011T11:28:50-05:00
---

## 简介

**代理模式（Proxy Pattern）** 是一种结构型设计模式，允许我们提供一个代理对象来控制对另一个对象的访问。代理对象通常负责控制访问权限、延迟初始化或提供额外功能，而不直接暴露实际对象的细节。代理模式在实际开发中非常有用，尤其是在需要控制对象的访问、进行缓存、延迟加载或进行安全控制的场景下。

## 应用场景

代理模式主要应用于以下场景：

1. **远程代理**：用于处理访问远程对象的请求，比如分布式系统中的RPC调用。
2. **虚拟代理**：用于延迟创建对象，以提高性能。常用于对象的惰性加载。
3. **保护代理**：用于控制对象的访问权限，确保只有合法用户可以访问对象。
4. **缓存代理**：用于缓存耗时的操作结果，避免重复计算或请求，提升系统性能。

### 实际应用场景

假设你正在开发一个图片查看应用。在这个应用中，加载高清图片是一个耗时的操作。如果每次查看图片都要重新加载高清图片，会导致用户体验下降。因此，可以使用代理模式来延迟加载图片，只有在真正需要时才加载图片。

## 示例代码

### 图片接口和实际图片类

```go
package main

import "fmt"

// Image 是图片接口
type Image interface {
    Display() // 展示图片
}

// RealImage 是实际的图片类，实现了 Image 接口
type RealImage struct {
    fileName string
}

// NewRealImage 是 RealImage 的构造函数，负责加载图片
func NewRealImage(fileName string) *RealImage {
    fmt.Println("Loading image:", fileName)
    return &RealImage{fileName: fileName}
}

// Display 是 RealImage 的展示方法
func (image *RealImage) Display() {
    fmt.Println("Displaying image:", image.fileName)
}
```

### 代理图片类

```go
// ProxyImage 是代理图片类，代理了 RealImage 的功能
type ProxyImage struct {
    realImage *RealImage
    fileName  string
}

// NewProxyImage 是 ProxyImage 的构造函数
func NewProxyImage(fileName string) *ProxyImage {
    return &ProxyImage{fileName: fileName}
}

// Display 是 ProxyImage 的展示方法，只有在需要时才创建 RealImage
func (proxy *ProxyImage) Display() {
    if proxy.realImage == nil {
        proxy.realImage = NewRealImage(proxy.fileName) // 延迟加载图片
    }
    proxy.realImage.Display()
}
```

### 使用示例

```go
func main() {
    // 不使用代理直接加载图片
    fmt.Println("Without Proxy:")
    realImage := NewRealImage("test_image.jpg")
    realImage.Display()
    fmt.Println()

    // 使用代理延迟加载图片
    fmt.Println("With Proxy:")
    proxyImage := NewProxyImage("test_image.jpg")
    proxyImage.Display() // 第一次调用时加载并展示图片
    proxyImage.Display() // 第二次调用时直接展示图片，不再加载
}
```

### 代码解释

- Image 接口：定义了一个通用的图片接口，包含 Display 方法。
- RealImage 类：实现了 Image 接口，表示实际图片类，包含 fileName 字段，负责加载并展示图片。
- ProxyImage 类：也是 Image 的实现类，但它作为 RealImage 的代理，控制 RealImage 的访问。代理类延迟加载图片，只有在需要时才加载真实的图片。
- main 函数：展示了使用代理模式进行图片加载的过程，第一次调用 Display 方法时会加载图片，之后的调用则不会再次加载。

### 图解

``` go
+-------------+        +----------------+        +-------------------+
|   Client    | -----> |   ProxyImage    | -----> |    RealImage       |
+-------------+        +----------------+        +-------------------+
```

在这个图中，客户端通过 ProxyImage 访问 RealImage，而 ProxyImage 控制了 RealImage 的创建和访问。

## 单元测试

以下是针对代理模式的单元测试，确保代理类能够正确代理实际对象。

```go
package main

import (
    "testing"
)

func TestRealImage(t *testing.T) {
    realImage := NewRealImage("test_image.jpg")
    result := realImage.fileName

    expected := "test_image.jpg"
    if result != expected {
        t.Errorf("Expected %s but got %s", expected, result)
    }
}

func TestProxyImage(t *testing.T) {
    proxyImage := NewProxyImage("test_image.jpg")

    // 第一次调用时应该创建 RealImage
    if proxyImage.realImage != nil {
        t.Error("Expected realImage to be nil before Display()")
    }

    proxyImage.Display() // 调用 Display，加载图片
    if proxyImage.realImage == nil {
        t.Error("Expected realImage to be created after Display()")
    }
}
```

## 总结

代理模式 提供了一种通过代理对象控制对目标对象访问的方法。代理模式非常适合在以下场景中使用：

- 远程对象访问：代理模式可以用于处理远程对象，减少直接访问远程服务的复杂性。
- 资源管理：可以用于延迟加载资源，避免不必要的资源占用。
- 权限控制：代理模式可以控制对象的访问权限，确保只有合法用户才能访问资源。

优点：

- 控制对象访问：代理模式允许你控制对实际对象的访问，提供额外的功能（如缓存、权限检查）。
- 延迟加载：代理模式可以延迟创建实际对象，只有在需要时才创建，节省资源。
- 增强功能：代理模式可以在不修改原始类的前提下为类添加额外的功能。

缺点：

- 增加代码复杂性：引入代理类会增加代码的复杂性，尤其是在需要管理多个代理对象时。
- 性能开销：代理模式可能会带来一定的性能开销，因为每次调用都需要经过代理对象。

代理模式是非常灵活且强大的设计模式，适用于延迟加载、权限控制和资源管理等场景。通过代理，开发者可以为对象的访问提供额外的控制和优化，同时保持客户端代码的简洁和一致性。
