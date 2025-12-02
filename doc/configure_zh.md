# 配置 Sōzu

> 在深入了解代理的配置部分之前，如果您还没有阅读过 [入门文档](./getting_started.md)，您应该先看一下。

## 配置文件

> 配置文件使用 [.toml](https://github.com/toml-lang/toml) 格式。

Sōzu 配置过程涉及 3 个主要参数来源：

- `global` 部分，用于设置进程范围的参数。
- 协议的定义，如 `https`、`http`、`tcp`。
- `[clusters]` 下的集群部分。

### 全局参数

全局部分中的参数允许您定义主进程和工作进程共享的全局设置（如日志级别）：

| 参数 | 描述 | 可能的值 |
|---|---|---|
| `saved_state` | sozu 启动时尝试从中加载其状态的路径 | |
| `log_level` | 可能的值是 | `debug`、`trace`、`error`、`warn`、`info` |
| `log_target` | 可能的值是 | `stdout, tcp 或 udp 地址` |
| `access_logs_target` | 可能的值是（如果激活，则将访问日志发送到单独的目标） | `stdout`、`tcp 或 `udp 地址` |
| `command_socket` | unix 套接字命令的路径 | |
| `command_buffer_size` | 主进程用于处理命令的缓冲区大小（以字节为单位）。 | |
| `max_command_buffer_size` | 主进程用于处理命令的缓冲区的最大大小。 | |
| `worker_count` | 工作进程数 | |
| `worker_automatic_restart` | 如果激活，出现 panic 或崩溃的工作进程将重新启动（默认激活） | |
| `handle_process_affinity` | 将工作进程绑定到 cpu 核心。 | |
| `max_connections` | 最大同时/打开连接数 | |
| `max_buffers` | 用于代理的最大缓冲区数 | |
| `min_buffers` | 为代理预分配的最小缓冲区数 | |
| `buffer_size` | 工作进程使用的请求缓冲区大小（以字节为单位） | |
| `ctl_command_timeout` | 命令行等待命令完成的最长时间 | |
| `pid_file_path` | 将 pid 存储在特定文件位置 | |
| `front_timeout` | 前端套接字的最大不活动时间 | |
| `connect_timeout` | 连接请求的最大不活动时间 | |
| `request_timeout` | 请求的最大不活动时间 | |
| `zombie_check_interval` | 僵尸会话检查之间的持续时间 | |
| `activate_listeners` | 自动启动监听器 | |

_示例：_

```toml
command_socket = "./command_folder/sock"
saved_state = "./state.json"
log_level = "info"
log_target = "stdout"
command_buffer_size = 16384
worker_count = 2
handle_process_affinity = false
max_connections = 500
max_buffers = 500
buffer_size = 16384
activate_listeners = true
```

### 监听器

_listener_ 部分描述了一组接受客户端连接的监听套接字。
您可以定义任意数量的监听器。
它们遵循以下格式：

_通用参数：_

```toml
[[listeners]]
# 可能的值是 http、https 或 tcp
protocol = "http"
# 监听地址
address = "0.0.0.0:8080"
# address = "[::]:8080"

# 为日志和转发的标头指定一个不同于套接字所见的 IP
# public_address = "1.2.3.4:80

# 配置客户端套接字以接收 PROXY 协议头
# expect_proxy = false
```

#### HTTP 和 HTTPS 监听器特有选项

自 1.0.0 版本以来，Sōzu 允许为 HTTP 和 HTTPS 监听器定义自定义 HTTP 应答。

这些应答是可定制的：

  - 301 Moved Permanently
  - 400 Bad Request
  - 401 Unauthorized
  - 404 Not Found
  - 408 Request Timeout
  - 413 Payload Too Large
  - 502 Bad Gateway
  - 503 Service Unavailable
  - 504 Gateway Timeout
  - 507 Insufficient Storage

这些应答应以任何扩展名的纯文本文件提供（为清晰起见，我们建议使用 `.http`
），并且可能如下所示：

```html
HTTP/1.1 404 Not Found
Cache-Control: no-cache
Connection: close
Sozu-Id: %REQUEST_ID

<style>pre{background:#EEE;padding:10px;border:1px solid #AAA;border-radius: 5px;}</style>
<h1>404 Not Found</h1>

<p>在此处插入您的自定义文本，实际上，所有 HTML 都是可更改的，包括 CSS。</p>

<pre>
{
    "route": "%ROUTE",
    "request_id": "%REQUEST_ID",
}
</pre>
<footer>这是 Sozu 的自动应答。</footer>",
```

有许多可用的模板变量，如 `REQUEST_ID` 或 `CLUSTER_ID`，
在生成错误时，代理逻辑将替换它们。

要创建您自己的自定义 HTTP 应答，我们强烈建议您首先复制 `lib/src/protocol/kawa_h1/answers.rs`
中存在的默认应答，然后根据自己的喜好进行更改。为清晰起见，可以随意删除
默认字符串中的 `\r` 换行符。
Sōzu 将解析您的文件并替换您使用的任何换行符。

然后，为每个监听器提供每个自定义应答的绝对路径。

```toml
# 当 sozu 不知道请求的域或路径时，将发送 404 响应
answer_404 = "/path/to/my-404-answer.http"
# 如果没有可用的后端服务器，将发送 503 响应
answer_503 = "/path/to/my-503-answer.http"
# answer_507 = ...
```

如果前端具有 `sticky_session`，则粘性名称在监听器级别定义。

```toml
# 如果集群激活了 `sticky_session`，则定义粘性会话 cookie 的名称
# 默认为 "SOZUBALANCEID"
sticky_name = "SOZUBALANCEID"
```

#### HTTPS 监听器特有选项

```toml
# 支持的 TLS 版本。可能的值是 "SSL_V2"、"SSL_V3"、
# "TLS_V12"、"TLS_V13"。默认为 "TLS_V12" 和 "TLS_V13"
tls_versions = ["TLS_V12", "TLS_V13"]
```

#### 基于 Rustls 的 HTTPS 监听器特有选项

```toml
# 基于 rustls 的 HTTPS 监听器特有选项
cipher_list = [
    # TLS 1.3 密码套件
    "TLS13_AES_256_GCM_SHA384",
    "TLS13_AES_128_GCM_SHA256",
    "TLS13_CHACHA20_POLY1305_SHA256",
    # TLS 1.2 密码套件
    "TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384",
    "TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256",
    "TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305_SHA256",
    "TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384",
    "TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256",
    "TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305_SHA256",
]
```

### 集群

您可以在 `[clusters]` 部分下声明您的 _集群_ 列表。
它们遵循以下格式：

_强制参数：_

```toml
[clusters]

[clusters.NameOfYourCluster]
# 可能的值是 http 或 tcp
# https 代理将在此处使用 http
protocol = "http"

# 每个集群的负载均衡算法。可能的值是
# "roundrobin" 和 "random"。默认为 "roundrobin"
# load_balancing_policy="roundrobin"

# 强制集群将 http 流量重定向到 https
# https_redirect = true

frontends = [
  { address = "0.0.0.0:8080", hostname = "lolcatho.st" },
  { address = "0.0.0.0:8443", hostname = "lolcatho.st", certificate = "../lib/assets/certificate.pem", key = "../lib/assets/key.pem", certificate_chain = "../lib/assets/certificate_chain.pem" }
]
# frontends 的附加选项：sticky_session (布尔值)

backends  = [
  { address = "127.0.0.1:1026" }
]
```

## 指标

Sōzu 通过 `UDP` 套接字将其自身状态报告给另一个网络组件。
主进程和工作进程负责发送其状态。
我们实现 [statsd](https://github.com/b/statsd_spec) 协议来发送统计信息。
任何理解 `statsd` 协议的服务都可以从 Sōzu 收集指标。

### 配置指标

在您的 `config.toml` 中，您可以通过添加以下内容来定义外部服务的地址和端口：

```toml
[metrics]
address = "127.0.0.1:8125"
# 使用 InfluxDB 的 statsd 协议风格添加标签
# tagged_metrics = false
# 指标键前缀
# prefix = "sozu"
```

目前，我们无法更改发送消息的频率。

### 外部服务示例

- [statsd](https://github.com/etsy/statsd)
- [grad](https://github.com/geal/grad)

## PROXY 协议

当网络流通过代理时，后端服务器将仅看到代理用作客户端地址的 IP 地址和端口。
真实的源 IP 地址和端口将仅由代理看到。
由于此信息对于日志记录、安全等很有用，
因此开发了 [PROXY 协议](https://www.haproxy.org/download/1.8/doc/proxy-protocol.txt) 以将其传输到后端服务器。
使用此协议，在连接到后端服务器后，代理将首先发送一个指示客户端 IP 地址和端口
以及代理的接收 IP 地址和端口的小标头，然后将发送来自客户端的流。

Sōzu 支持 `PROXY 协议` 的 _版本 2_，有三种配置：

- “发送”协议：Sōzu 在 TCP 代理模式下，将向后端服务器发送标头
- “期望”协议：Sōzu 从代理接收标头，为其自己的日志记录和指标解释它，并在 HTTP 转发标头中使用它
- “中继”协议：Sōzu 在 TCP 代理模式下，可以接收标头，并将其传输到后端服务器

更多信息请参见：[proxy-protocol spec](https://www.haproxy.org/download/1.8/doc/proxy-protocol.txt)

### 配置 Sōzu _期望_ PROXY 协议头

配置面向客户端的连接以在从套接字读取客户端发送的任何字节之前接收 PROXY 协议头。

```txt
                           发送 PROXY                   期望 PROXY
                           协议头                       协议头
    +--------+
    | 客户端 |             +---------+                   +------------+      +-----------+
    |        |             | 代理    |                   | Sozu       |      | 上游      |
    +--------+  ---------> | 服务器  |  ---------------> |            |------| 服务器    |
   /        /              |         |                   |            |      |           |
  /________/               +---------+                   +------------+      +-----------+
```

它受 HTTP、HTTPS 和 TCP 代理支持。

_配置：_

```toml
[[listeners]]
address = "0.0.0.0:80"
expect_proxy = true
```

### 配置 Sōzu 向 上游后端 _发送_ PROXY 协议头

在与集群中声明的后端建立的任何连接上发送 PROXY 协议头。

```txt
                           发送 PROXY
    +--------+             协议头
    | 客户端 |             +---------+
    |        |             | Sozu    |
    +--------+  ---------> |         |  ------------> +-------------------+ 
   /        /              |         |                | 代理/上游       |
  /________/               +---------+                | 服务器          |
                                                      |                 |
                                                      +-----------------+
```

_配置：_

```toml
[[listeners]]
address = "0.0.0.0:81"

[clusters]
[clusters.NameOfYourTcpCluster]
send_proxy = true
frontends = [
  { address = "0.0.0.0:81" }
]
```

注意：仅适用于 TCP 集群（HTTP 和 HTTPS 代理将使用转发标头）。

### 配置 Sōzu 向 上游 _中继_ PROXY 协议头

Sōzu 将从客户端连接接收 PROXY 协议头，检查其有效性，然后将其转发到上游后端。这允许反向代理链在不丢失客户端连接信息的情况下工作。

```txt
                           发送 PROXY                       期望 PROXY               发送 PROXY
                           协议头                           协议头                   协议头
    +--------+
    | 客户端 |             +---------+                      +------------+             +-------------------+
    |        |             | 代理    |                      | Sozu       |             | 代理/上游         |
    +--------+  +--------> | 服务器  |  +-----------------> |            | +---------> | 服务器            |
   /        /              |         |                      |            |             |                   |
  /________/               +---------+                      +------------+             +-------------------+ 
```

_配置：_

这仅涉及 TCP 集群（HTTP 和 HTTPS 代理可以直接在期望模式下工作，并将使用转发标头）。

```toml
[[listeners]]
address = "0.0.0.0:80"
expect_proxy = true

[clusters]

[clusters.NameOfYourCluster]
send_proxy = true
frontends = [
  { address = "0.0.0.0:80" }
]
```
