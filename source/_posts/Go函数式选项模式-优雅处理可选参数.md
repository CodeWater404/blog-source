---
title: "Go 函数式选项模式：优雅处理构造函数的可选参数"
date: 2026-07-21 00:28:39
updated: 2026-07-22 11:43:58
cover: /img/p58.jpg
categories: golang
tags:
  - Go
  - 设计模式
  - 函数式选项模式
description: "Go 没有默认参数和函数重载，构造函数一多可选项就容易参数爆炸。函数式选项模式怎么解决，以及什么时候不该用它。"
---

写一个 `Server` 结构体，一开始只要 `addr`，后来陆续加上超时时间、最大连接数、TLS 开关、日志器……不知不觉构造函数变成了这样：

```go
func NewServer(addr string, timeout time.Duration, maxConns int, useTLS bool, logger *log.Logger) *Server
```

调用的地方是这样的：

```go
s := NewServer("localhost:8080", 30*time.Second, 100, false, nil)
```

第三个参数是不是超时？第四个 `false` 是不是 TLS？光看调用点完全猜不出来，新加一个可选参数还得改遍所有调用方。这是几乎每个 Go 项目迟早会撞上的问题，也是函数式选项模式（Functional Options Pattern）要解决的。

<!-- more -->

## 为什么朴素解法都不够用

Go 没有函数重载，也没有默认参数——这两条路在 Java、Python、C++ 里都能用来处理可选参数，Go 里直接堵死了。剩下常见的朴素解法有两种，各有各的毛病。

**全部做成必填参数**（就是开头那个例子）：参数一多，调用点全是没有名字的字面量堆在一起，顺序错了编译器还不会报错——`bool` 和 `bool` 换个位置照样能编译通过，只是语义全反了。

**用一个 Config 结构体传参**：

```go
type Config struct {
    Timeout  time.Duration
    MaxConns int
    UseTLS   bool
    Logger   *log.Logger
}

func NewServer(addr string, cfg Config) *Server
```

这个好一些，调用点能带字段名：

```go
s := NewServer("localhost:8080", Config{Timeout: 30 * time.Second})
```

但零值问题还在——`Config{}` 里没显式写的字段全部是零值，`MaxConns: 0` 到底是"用户就是要 0"还是"用户没设置，请用默认值"，函数内部区分不出来。而且 `Config` 是公开的可变结构体，谁都能在调用之后偷偷改字段，构造函数的"配置只在创建时生效一次"这层保证也就没了。

## 函数式选项模式：用函数取代参数

核心思路很简单：**把每一个可选配置做成一个"修改内部状态"的函数**，构造函数只接收一个可变数量的这类函数，依次执行它们。

```go
type Server struct {
    addr     string
    timeout  time.Duration
    maxConns int
    useTLS   bool
    logger   *log.Logger
}

type Option func(*Server)
//    Option 就是一个"接收 *Server、修改它某个字段"的函数类型

func WithTimeout(d time.Duration) Option {
    return func(s *Server) { s.timeout = d }
}

func WithMaxConns(n int) Option {
    return func(s *Server) { s.maxConns = n }
}

func WithTLS() Option {
    return func(s *Server) { s.useTLS = true }
}

func NewServer(addr string, opts ...Option) *Server {
    s := &Server{
        addr:     addr,
        timeout:  30 * time.Second, // 默认值先设好
        maxConns: 100,
    }
    for _, opt := range opts {
        opt(s) // 依次应用每一个传进来的 Option
    }
    return s
}
```

调用的地方变成这样，不用的选项完全不用管，用到的选项名字自解释：

```go
s := NewServer("localhost:8080", WithTimeout(5*time.Second), WithTLS())
```

这一下解决了朴素解法的两个问题：调用点每个参数都有名字，不会猜错；默认值在构造函数内部统一设置一次，`Option` 只负责覆盖用户显式要改的字段，零值和"没设置"不再混淆——因为没传对应的 `Option`，那个字段就压根没被碰过，走的是构造函数里设的默认值。

## 为什么这个模式在 Go 里特别自然

`Option` 只是一个普通的函数类型，不需要接口、不需要反射，这依赖两个 Go 的语言特性：函数是一等公民（可以当值传递、当返回值返回），以及变长参数 `...Option` 天然支持"传 0 个、1 个还是 10 个都可以"。这也是为什么其他语言里更常见的是 Builder 模式（链式调用 `.setTimeout().setMaxConns()`），而 Go 项目里选项模式出现得更多——变长参数 + 一等函数刚好把这条路铺平了。

标准库外的知名库里能直接看到这个模式，比如 gRPC-Go 建连接：

```go
conn, err := grpc.NewClient(
    "localhost:9090",
    grpc.WithTransportCredentials(insecure.NewCredentials()),
    grpc.WithDefaultCallOptions(grpc.MaxCallRecvMsgSize(10 << 20)),
)
```

`grpc.WithTransportCredentials(...)`、`grpc.WithDefaultCallOptions(...)` 都是返回 `grpc.DialOption` 的函数，和上面 `WithTimeout` 是同一个套路。Uber 的日志库 zap 也是同一个模式，`zap.NewProduction(zap.AddCaller())` 里的 `AddCaller()` 就是一个 `zap.Option`。

## 必填参数不要塞进 Option 里

`addr` 放在 `NewServer` 的固定参数位置，而不是做成 `WithAddr(...)` 这样的 Option，是有意为之——**必填的东西就该是不能省略的固定参数，可选的东西才用 Option**。如果把必填参数也塞进 Option 变长参数里，调用方漏传会在运行时才发现（甚至默默用了零值），而不是编译期报错。一个实用的判断标准：这个字段没有合理默认值、缺了它这个对象就没法正常工作，就应该是固定参数；反之才是 Option 的候选。

## 什么时候不需要这个模式

选项模式不是没有代价——每加一个字段就要多写一个 `WithXxx` 函数，构造函数内部还要维护默认值。如果一个结构体只有一两个可选字段，而且未来大概率不会再增加，直接用朴素的 Config 结构体或者干脆做成必填参数，比引入一整套 Option 类型更省事。这个模式真正划算的场景是：**可选参数数量会持续增长、调用方大多数时候只需要默认值、少数场景需要精确控制某几项**——库的公开 API（像 gRPC、zap 那样面向大量外部调用方）是最典型的场景。

## 该不该用，看两个信号

选项模式解决的是"可选参数一多就退化成没有名字的位置参数"这个具体问题，代价是每个可选项要多写一个小函数。判断要不要用它，看两点：可选字段是不是会持续变多，以及这个构造函数是不是会被很多不同调用方以不同组合调用。日常写业务代码里两三个字段的内部 struct，不需要照搬这套；写对外的库或者配置项经常变的基础设施代码，这个模式几乎是标配。
