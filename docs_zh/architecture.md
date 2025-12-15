# 架构

这部分主要面向希望了解 sōzu 工作原理的人。

## 主/工作进程模型

Sōzu 采用一个主进程和多个工作进程的模式。这使得当一个工作进程遇到问题并崩溃时，它能继续运行，并在必要时逐个升级工作进程。

### 单线程，无共享架构

每个工作进程运行一个单线程，并带有一个基于 epoll 的事件循环。为避免同步问题，每个工作进程都拥有整个路由配置的副本。路由的每一次修改都通过配置消息进行。日志和指标由每个工作进程单独发送，将聚合和序列化事件的工作留给外部服务。
所有监听的 TCP 套接字都使用 [SO_REUSEPORT](https://lwn.net/Articles/542629/) 选项打开，允许多个进程在同一地址上监听。

### 配置

外部工具通过一个 unix 套接字与主进程交互，配置更改消息将由主进程分发给工作进程。
配置消息是“差异”，例如“添加一个后端服务器”或“删除一个 HTTP 前端”，而不是一次性更改整个配置。这使得 sōzu 能够在有流量的情况下更智能地处理配置更改。

配置消息以 protobuf 二进制格式传输，它们定义在 [command 库](https://github.com/sozu-proxy/sozu/tree/main/command)中。有三种可能的消息回复：processing（表示消息已收到但更改尚未生效）、failure 或 ok。

主进程暴露一个用于配置的 unix 套接字，而不是在 localhost 上暴露一个 HTTP 服务器，因为 unix 套接字的访问可以通过文件系统权限来保护。

## 代理

### 使用 mio 的事件循环

每个工作进程都运行一个基于 epoll（在 Linux 上）或 kqueue（在 OSX 和 BSD 上）的事件循环，使用 [mio 库](https://github.com/tokio-rs/mio)。

Mio 提供了一个跨平台抽象，允许调用者接收事件，例如套接字变为可读（意味着它收到了一些数据）。

Sōzu 要求 mio 以[边缘触发模式](http://man7.org/linux/man-pages/man7/epoll.7.html)发送套接字的所有事件。
这样，它只接收一次事件，并将其存储在一个
[`Readiness` 结构体](https://github.com/sozu-proxy/sozu/blob/01a78be7d95ac295d30b342d3ec0be403c98e776/lib/src/lib.rs#L527)中。
然后它将使用该信息和“兴趣”（指示当前协议状态机是否想在套接字上读取或写入）。

每个套接字事件都带有一个 `Token` 返回，指示其在 `Slab` 数据结构中的索引。一个客户端会话可以有多个套接字（通常是一个前端套接字和一个后端套接字）。

### 协议

每个代理实现（HTTP、HTTPS 和 TCP）将在每个客户端会话中使用一个状态机来描述当前使用的协议。它旨在允许从一个协议升级到下一个协议。例如，你可以有以下 progression：

- 在 TLS 握手协议中启动
- 握手完成后，升级到最近协商的 TLS 流上的 HTTP 协议
- 升级到 websockets

每个协议都将与 `Readiness` 结构一起工作，以指示它是否想在每个套接字上读取或写入。例如，[基于 OpenSSL 的握手](https://github.com/sozu-proxy/sozu/blob/3111e2db420d2773b1f0404d6556f40b2f2ea85b/lib/src/network/protocol/openssl.rs) 只对前端套接字感兴趣。

它们都在 [`lib/src/network/protocol`](https://github.com/sozu-proxy/sozu/tree/3111e2db420d2773b1f0404d6556f40b2f2ea85b/lib/src/network/protocol) 中定义。

## 日志记录

[记录器](https://github.com/sozu-proxy/sozu/blob/3111e2db420d2773b1f0404d6556f40b2f2ea85b/lib/src/logging.rs) 旨在使用 Rust 的格式化系统减少分配和字符串插值。它可以在各种后端上发送日志：stdout、文件、TCP、UDP、Unix 套接字。

记录器可以通过一个线程局部存储变量从任何地方通过日志宏调用。

## 指标

[指标](https://github.com/sozu-proxy/sozu/tree/3111e2db420d2773b1f0404d6556f40b2f2ea85b/lib/src/network/metrics) 的工作方式与记录器类似，可以通过宏和 TLS 从任何地方访问。我们支持两种“drains”：一种通过 statsd 兼容协议在网络上发送指标，另一种在本地聚合指标，以通过配置套接字进行查询。

## 负载均衡

对于给定的集群，Sōzu 维护一个后端列表，连接将被重定向到这些后端。
Sōzu 检测损坏的服务器，并仅将流量重定向到健康的服务器，有多种可用的负载均衡算法：
轮询（默认）、随机、最少连接和二次幂。

## TLS

Sōzu 是一个由 rustls 支持的 TLS 端点。
它使用 TLS 密钥和证书解密流量，并将其未加密地转发到后端。

## 深入探讨

### 缓冲区

Sōzu 经过优化，内存使用非常有限。
所有流量都（短暂地）存储在一个固定大小（通常为 16 kB）的可重用缓冲区池中。

### 通道

它们是 unix 套接字之上的一个抽象层，使与 sōzu 的通信更容易。
