---
title: "Golang 设计模式之单例模式"
categories:
  - Golang
tags:
  - Go Design Pattern
last_modified_at: 2024-09-08T11:28:50-05:00
---

## 简介

**单例模式（Singleton Pattern）** 是一种创建型设计模式，其目的是确保一个类只有一个实例，并提供一个全局访问点来获取该实例。单例模式在应用程序中是很常见的，特别是当你需要一个全局的唯一实例来协调系统中的一些共享资源时。

## 应用场景

单例模式适用于以下场景：

1. **全局共享资源**：当你需要一个全局唯一的实例来协调系统中的共享资源，如数据库连接池、配置管理器等。
2. **限制实例化**：当你需要确保某个类在整个应用程序中只有一个实例时，如应用程序的日志记录器。
3. **节省资源**：当对象创建开销较大或资源有限时，通过单例模式可以避免重复创建对象，从而节省资源。

例如，配置管理器、日志记录器和线程池都是常见的单例模式应用场景。

## 示例代码

下面的示例展示了如何在 Go 语言中实现单例模式。我们将创建一个 `Logger` 结构体，它负责记录日志，并确保只有一个 `Logger` 实例存在。

### 代码实现

```go
package main

import (
    "fmt"
    "sync"
)

// Logger 是单例对象
type Logger struct {
    LogLevel string
}

var (
    instance *Logger
    once     sync.Once
)

// GetInstance 返回 Logger 的唯一实例
func GetInstance() *Logger {
    once.Do(func() {
        instance = &Logger{LogLevel: "INFO"}
    })
    return instance
}

// Log 方法用于记录日志
func (l *Logger) Log(message string) {
    fmt.Printf("[%s] %s\n", l.LogLevel, message)
}

func main() {
    // 获取 Logger 实例并记录日志
    logger1 := GetInstance()
    logger1.Log("Application started")

    // 获取另一个 Logger 实例并记录日志
    logger2 := GetInstance()
    logger2.Log("Application finished")

    // 验证两个 Logger 实例是否相同
    if logger1 == logger2 {
        fmt.Println("logger1 和 logger2 是同一个实例")
    }
}
```

### 代码解释

- Logger 结构体：表示日志记录器对象，包含一个 LogLevel 字段。
- instance 和 once：instance 是 Logger 的唯一实例，once 是 sync.Once 类型，用于确保初始化代码只执行一次。
- GetInstance 函数：通过 once.Do 方法保证 Logger 实例只被创建一次，并返回该实例。
- Log 方法：记录日志消息。
- main 函数：演示如何获取单例实例并记录日志，同时验证两个 Logger 实例是否相同。

## 单元测试

以下是测试 Logger 单例模式实现的测试代码，确保 GetInstance 方法返回相同的实例。

```go
package main

import (
    "testing"
)

func TestSingleton(t *testing.T) {
    logger1 := GetInstance()
    logger2 := GetInstance()

    if logger1 != logger2 {
        t.Errorf("Expected both instances to be the same, but got different instances")
    }
}

func TestLoggerLog(t *testing.T) {
    logger := GetInstance()
    message := "Test message"

    // Capture the output
    original := fmt.Sprintf("%s", message)
    logger.Log(message)
    output := fmt.Sprintf("%s", message)

    if output != original {
        t.Errorf("Expected output to be %s, but got %s", original, output)
    }
}
```

### 测试解释

- TestSingleton：测试 GetInstance 是否始终返回相同的实例。
- TestLoggerLog：测试 Log 方法是否正常记录日志（这里可以扩展以捕获和验证实际输出）。

## 总结

单例模式 是一种简单而有效的设计模式，通过确保一个类只有一个实例，并提供全局访问点来获取该实例，从而管理系统中的全局状态。它适用于需要全局唯一实例的场景，如配置管理、日志记录等。

优点：

- 控制实例化：确保系统中只有一个实例，避免重复创建。
- 全局访问：提供全局访问点，简化系统中的资源管理。

缺点：

- 测试难度：单例模式可能会增加测试的复杂性，因为单例的全局状态可能影响测试结果。
- 隐式依赖：全局实例可能导致隐式依赖，使得代码难以理解和维护。

在实际开发中，单例模式是解决全局状态和资源共享问题的有力工具。通过合理应用单例模式，可以简化对象管理，提升系统的可维护性和效率。
