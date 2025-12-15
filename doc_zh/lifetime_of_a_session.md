# 会话的生命周期

会话是 Sōzu 业务逻辑的核心单元：将流量从
前端转发到后端，反之亦然。

在本文档中，我们将探讨客户端会话从套接字的创建到关闭所发生的一切，
以及描述 HTTP 请求和响应如何发生的所有步骤。

在我们深入探讨会话的创建、生命周期和消亡之前，我们需要
理解 Sōzu 中完全隔离的两个概念：

- 使用 [mio](https://docs.rs/mio/0.7.13/mio/) crate 进行套接字处理
- 跟踪所有会话的 `SessionManager`

### mio 是做什么的？

Mio 允许我们监听套接字事件。例如，我们可以使用 TCP 套接字来
等待连接，当该套接字可读或套接字有
数据要读取，或被对等方关闭，或计时器触发时，mio 会通知我们...

Mio 提供了对 linux 系统调用 [epoll](https://man7.org/linux/man-pages/man7/epoll.7.html) 的抽象。
这个 epoll 系统调用允许我们注册文件描述符，以便内核在这些文件描述符发生任何事情时通知我们。

说到底，套接字只是原始文件描述符。我们使用 mio
`TcpListener`、`TcpStream` 包装器来包装这些文件描述符。`TcpListener`
在特定端口上侦听连接。对于每个新连接，它都会创建一个
`TcpStream`，后续流量将重定向到该 `TcpStream`（从客户端和到客户端）。

这就是我们使用 mio 的全部目的。“订阅”文件描述符事件。

### 使用 SessionManager 跟踪会话

mio 中的每个订阅都与一个 Token（一个 u64 标识符）相关联。SessionManager 的
工作是将一个 Token 链接到一个将使用该订阅的 `ProxySession`。
这是通过一个 [Slab]() 完成的，这是一个优化内存使用的键值数据结构：

- 作为键：mio 提供的 Token
- 作为值：对 `ProxySession` 的引用

话虽如此，让我们深入探讨会-话的生命周期。

### 什么是代理？

Sōzu 工作进程内部有 3 个代理，每个支持的协议一个：

- TCP
- HTTP
- HTTPS

代理管理侦听器、前端、后端以及与每个协议（缓冲、解析、错误处理...）相关的业务逻辑。

## 接受连接

Sōzu 使用 TCP 侦听器套接字来获取新连接。它通常会侦听
端口 80 (HTTP) 和 443 (HTTPS)，但也可以有其他用于 TCP 代理的端口，
甚至在其他端口上有 HTTP/HTTPS 代理。

对于每个前端，Sōzu：

1. 生成一个新的 Token *token*
2. 使用 mio 将 `TcpListener` 注册为 *token*
3. 在 `SessionManager` 中添加一个匹配的 `ListenSession`，键为 *token*
4. 将 `TcpListener` 存储在适当的代理（TCP/HTTP/HTTPS）中，键为 *token*

### TCP 套接字上的事件循环

Sōzu 是热可重构的。我们可以在运行时添加侦听器和前端。对于每个添加的侦听器，
SessionManager 将在其 Slab 中存储一个 `ListenSession`。

[事件循环]() 使用 mio 检查所有套接字上的任何活动。
每当 mio 在套接字上检测到活动时，它都会返回一个传递给 `SessionManager` 的事件。

每当客户端连接到前端时：
1. 它到达一个侦听器
2. Sōzu 收到 mio 的通知，称在特定的 Token 上收到了一个 `readable` 事件
3. 使用 SessionManager，Sōzu 获取相应的 `ListenSession`
4. Sōzu 确定所使用的协议
5. Token 被传递到适当的代理（TCP/HTTP/HTTPS）
6. 使用 Token，代理确定哪个 `TcpListener` 触发了事件
7. 代理开始以循环方式从中接受新连接（因为可能不止一个）

接受连接意味着将其作为 `TcpStream` 存储在接受队列中，直到：

- 没有更多连接要接受
- 或者接受队列已满：https://github.com/sozu-proxy/sozu/blob/e4e7488232ad6523791b94ad201239bcf7eb9b30/lib/src/server.rs#L1204-L1258

### 消费接受队列

我们从接受队列创建会话，从
最新的会话开始，并丢弃太旧的套接字。

当我们接受一个新的连接（`TcpStream`）时，它可能已经在
侦听器队列中等待了一段时间。内核甚至可能已经有一些可用的数据，
比如一个 HTTP 请求。如果我们处理该请求太慢，客户端可能
在我们将会话转发到后端并且后端响应之前就已经断开了连接（超时）。

如果达到了客户端连接的最大数量（由配置中的 `max_connections` 提供），
新的连接将保留在队列中。
如果队列已满，我们将丢弃新接受的连接。
通过指定最大并发连接数，我们确保
服务器不会过载，并为现有连接保持可管理的延迟。

### 创建会话

会话的目标是将流量从前端转发到后端，反之亦然。
`Session` 结构体保存与
[会话](https://github.com/sozu-proxy/sozu/blob/e4e7488232ad6523791b94ad201239bcf7eb9b30/lib/src/https_openssl.rs#L65-L83)关联的数据：
令牌、当前超时状态、协议状态、客户端地址...
创建的会话被包装在一个
[`Rc<RefCell<...>>`](https://github.com/sozu-proxy/sozu/blob/e4e7488232ad6523791b94ad201239bcf7eb9b30/lib/src/https_openssl.rs#L1544) 中

代理为接受队列的每个项目创建一个会话，
使用项目中提供的 `TcpStream`。
`TcpStream` 在 mio 中注册了一个新的 Token（称为 frontend_token）。
会话以相同的 Token 添加到 SessionManager 中。

### 检查僵尸会话

因为在创建和删除会话时可能会发生错误，并且其中一些
可能会被“遗忘”，所以有一个名为
[“僵尸检查器”](https://github.com/sozu-proxy/sozu/blob/e4e7488232ad6523791b94ad201239bcf7eb9b30/lib/src/server.rs#L446-L496) 的常规任务，
它会检查列表中的会话并终止那些卡住或太旧的会话。

## 会话如何从前端套接字读取数据

当数据从网络到达 `TcpStream` 时，它会存储在内核
缓冲区中，内核会通知事件循环套接字是可读的。

与侦听套接字一样，与 TCP 套接字关联的令牌将获得一个
“可读”事件，我们将使用该令牌来查找与哪个会话
关联。然后我们调用
[`Session::update_readiness`](https://github.com/sozu-proxy/sozu/blob/e4e7488232ad6523791b94ad201239bcf7eb9b30/lib/src/server.rs#L1431)
来通知它新的
[套接字状态](https://github.com/sozu-proxy/sozu/blob/e4e7488232ad6523791b94ad201239bcf7eb9b30/lib/src/https_openssl.rs#L810-L820)。

然后我们调用
[`Session::ready`](https://github.com/sozu-proxy/sozu/blob/e4e7488232ad6523791b94ad201239bcf7eb9b30/lib/src/https_openssl.rs#L822-L827)
让它读取数据、解析等。
该方法将在一个循环中运行 (https://github.com/sozu-proxy/sozu/blob/e4e7488232ad6523791b94ad201239bcf7eb9b30/lib/src/https_openssl.rs#L548-L692)，

### 引入状态机

会话的 [`Session::readable` 方法](https://github.com/sozu-proxy/sozu/blob/e4e7488232ad6523791b94ad201239bcf7eb9b30/lib/src/https_openssl.rs#L309-L339)
被调用。然后它将调用底层
[状态机](https://github.com/sozu-proxy/sozu/blob/e4e7488232ad6523791b94ad201239bcf7eb9b30/lib/src/https_openssl.rs#L58-L63)的相同方法。

状态机是实现协议的地方。会话可能需要
在其生命周期内识别不同的协议，具体取决于其配置，
并在它们之间进行 [升级](https://github.com/sozu-proxy/sozu/blob/e4e7488232ad6523791b94ad201239bcf7eb9b30/lib/src/https_openssl.rs#L165)。它们都在 [protocol 目录](https://github.com/sozu-proxy/sozu/tree/e4e7488232ad6523791b94ad201239bcf7eb9b30/lib/src/protocol)中。

示例：
- 一个 HTTPS 会话可以从一个名为 `ExpectProxyProtocol` 的状态开始
- 一旦 expect 协议运行完毕，会话将升级到 TLS 握手状态：`HandShake`
- 握手完成后，我们有一个 TLS 流，会话升级到 `HTTP` 状态
- 如果客户端需要，会话可以切换到 WebSocket 连接：`WebSocket`

现在，假设我们当前正在使用 HTTP 1 协议。会话调用了
[`readable()` 方法](https://github.com/sozu-proxy/sozu/blob/e4e7488232ad6523791b94ad201239bcf7eb9b30/lib/src/protocol/http/mod.rs#L609)。

### 查找后端

我们需要解析 HTTP 请求以找出其：

1. 主机名
2. 路径
3. HTTP 动词

我们将首先尝试
[从套接字读取数据](https://github.com/sozu-proxy/sozu/blob/e4e7488232ad6523791b94ad201239bcf7eb9b30/lib/src/protocol/http/mod.rs#L646-L667)
到前端缓冲区。如果没有错误（关闭的套接字等），我们将在
[`readable_parse()`](https://github.com/sozu-proxy/sozu/blob/e4e7488232ad6523791b94ad201239bcf7eb9b30/lib/src/protocol/http/mod.rs#L728) 中处理该数据。

HTTP 实现拥有
[两个较小的状态机](https://github.com/sozu-proxy/sozu/blob/e4e7488232ad6523791b94ad201239bcf7eb9b30/lib/src/protocol/http/mod.rs#L86-L87)，
`RequestState` 和 `ResponseState`，它们指示我们在解析
请求或响应中的位置，并存储有关它们的数据。

当我们从客户端接收到第一个字节时，两者都处于 `Initial` 状态。
我们 [从前端缓冲区解析数据](https://github.com/sozu-proxy/sozu/blob/e4e7488232ad6523791b94ad201239bcf7eb9b30/lib/src/protocol/http/mod.rs#L734-L737)
直到我们达到一个请求状态，其中标头被完全解析。如果
没有足够的数据，我们将等待更多数据到达套接字并重新开始
解析。

一旦我们完成了解析标头，并找到了我们正在寻找的内容，我们将
[返回 SessionResult::ConnectBackend](https://github.com/sozu-proxy/sozu/blob/e4e7488232ad6523791b94ad201239bcf7eb9b30/lib/src/protocol/http/mod.rs#L751-L759)
以通知会话它应该找到一个后端服务器来发送数据。

## 连接到后端服务器

会话：
1. 查找要连接到哪个集群
2. 向 SessionManager 请求一个名为 `back_token` 的新有效 Token
3. 请求连接到集群
   - 适当的代理查找后端（添加详细信息）
4. 使用 `back_token` 在 mio 中注册新的 `TcpStream`
5. 使用 `back_token` 将自身插入 SessionManager

同一个会话现在在 SessionManager 中存储了两次：

1. 一次使用前端令牌作为键
2. 其次使用后端令牌作为键

如果 Sōzu 找不到集群，它会向客户端响应默认的 HTTP 404 Not Found 响应。
一个会话可以尝试连接到后端 3 次。如果所有尝试都失败，Sōzu 会响应默认的
HTTP 503 Service Unavailable 响应。如果 Sōzu 找到了一个集群，但所有相应的
后端都已关闭（或没有响应），则会发生这种情况。

### 保持会话活动

如果 HTTP 请求的 Connection 标头设置为 Keep-Alive，
则在接收到响应后可以保持底层 TCP 连接，
以发送更多请求。由于 Sōzu 支持在 URL 和
主机名上进行路由，因此下一个请求可能会转到不同的集群。
因此，当我们从请求中获取集群 ID 时，我们会检查它是否与
前一个相同，如果相同，我们会测试后端套接字是否仍然
有效。如果是，我们可以重用它。否则，我们将用
一个新的后端套接字替换它。

### 粘性会话：将一个前端只固定到一个后端

这是一种路由机制，我们查看请求中的 cookie。所有
具有相同 id 的请求都将发送到同一个后端服务器。

该查找将返回一个结果，具体取决于哪些后端服务器被
认为是有效的（如果它们正在正确响应）以及为集群配置的负载平衡
策略。
如果找到了后端，我们将打开一个到后端服务器的 TCP 连接，
否则我们将返回一个 HTTP 503 响应。

## 将数据发送到后端套接字

然后我们等待来自后端连接的可写事件，然后我们可以开始
将挂起的请求转发给它。
如果出现错误，我们会重试连接到同一集群中的另一个后端服务器。

## 大转弯：从后端转发到前端

如上所述，我们有一个名为 `ResponseState` 的小型状态机，用于
解析来自后端的流量。整个逻辑基本相同。

我们监视后端的可读性和前端的可写性，并将流量从
一个转移到另一个。

## 会话结束

一旦 `ResponseState` 达到“已完成”状态并且每个字节都已发送
回客户端，请求的完整生命周期就结束了。会话
达到 `CloseSession` 状态并从 `SessionManager` 的 slab 中移除，
其套接字也从 mio 中注销。

但是，如果请求具有 Keep-Alive 标头，则会话将被重用
并等待新请求。这是会话的“重置”。
