# Sozu 中文使用教程

## 简介

Sozu 是一个快速、可靠且可编程的 HTTP 和 TCP 反向代理，由 Rust 编写。它支持高级路由、负载均衡、TLS 终止和动态配置。

## 1. Rust 环境设置

在安装 Sozu 之前，请确保您的系统上安装了最新稳定版的 Rust。我们推荐使用 `rustup` 进行安装：

*   **安装 `rustup`**: 访问 [rustup 官方网站](https://rustup.rs) 或在终端运行以下命令：

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

*   **配置环境**: 按照安装程序的指示操作，通常需要执行 `source $HOME/.cargo/env` 来将 `cargo` (Rust 的包管理器) 添加到您的 PATH 中。

*   **验证安装**: 运行 `rustc --version` 和 `cargo --version` 来验证 Rust 是否成功安装。

## 2. Sozu 安装

您可以选择通过 `cargo install` 从 crates.io 安装 Sozu，或者从源代码构建。

### 2.1 通过 `cargo install` 安装

Sozu 发布在 [crates.io](https://crates.io/) 上。安装非常简单：

```bash
cargo install sozu
```

安装完成后，`sozu` 可执行文件将在您的 `~/.cargo/bin` 目录中可用。

### 2.2 从源代码构建

如果您想从源代码构建 Sozu (例如，进行开发或使用最新版本)，请遵循以下步骤：

*   **克隆仓库**: 如果您尚未克隆 Sozu 仓库，请先克隆它：

```bash
git clone https://github.com/sozu-proxy/sozu.git
cd sozu
```

*   **构建**: 导航到 `bin` 目录并使用 `cargo build` 命令构建。为了获得生产版本，请务必使用 `--release` 标志以启用优化：

```bash
cd bin
cargo build --release --locked
```

-   `--release` 参数会告知 Cargo 启用编译优化，生成性能更好的二进制文件。仅在构建生产版本时使用。
-   `--locked` 标志会强制 Cargo 遵守 `Cargo.lock` 中指定的依赖版本，从而防止依赖中断。

构建完成后，可执行文件将在 `target/release/sozu` 路径下。

## 3. 如何使用 Sozu

Sozu 既可以作为独立进程运行，也可以集成到 Docker 容器或 Systemd 服务中。

### 3.1 运行 Sozu

-   **通过 `cargo install` 方式安装**: 如果您通过 `cargo install` 安装了 Sozu，那么 `sozu` 命令应该已经在您的 `$PATH` 中。

```bash
sozu start -c <您的/config.toml/路径>
```

-   **从源代码构建**: 如果您从源代码构建，Sozu 可执行文件位于 `target/release` 目录中。

```bash
./target/release/sozu start -c <您的/config.toml/路径>
```

您可以在 `config.toml` 文件中编辑反向代理的配置，声明新的集群、前端和后端。

**提示**: 您可以使用 `sozu` 二进制文件作为 CLI 与反向代理进行交互。更多信息请参阅[命令行文档](https://github.com/sozu-proxy/sozu/blob/main/doc/configure_cli_zh.md)。

### 3.2 使用 Docker 运行

Sozu 仓库提供了一个基于 `alpine:edge` 的多阶段 `Dockerfile` 镜像。

*   **构建 Docker 镜像**:

```bash
docker build -t sozu .
```

    您也可以构建特定 Alpine 版本的镜像：

```bash
docker build --build-arg ALPINE_VERSION=3.14 -t sozu:main-alpine-3.14 .
```

*   **运行 Docker 容器**:

```bash
docker run \
  --ulimit nofile=262144:262144 \
  --name sozu-proxy \
  -v /run/sozu:/run/sozu \
  -v /path/to/your/config.toml:/etc/sozu/config.toml \
  -v /my/state/:/var/lib/sozu \
  -p 8080:80 \
  -p 8443:443 \
  sozu start -c /etc/sozu/config.toml
```

-   `-v /path/to/your/config.toml:/etc/sozu/config.toml`: 将您的自定义 `config.toml` 文件挂载到容器中。
-   `-v /my/state/:/var/lib/sozu`: 如果您有初始配置状态 JSON 文件，请将其挂载到容器中。

### 3.3 Systemd 集成

Sozu 仓库提供了 Systemd 单元文件，可以轻松地将其作为服务运行。

*   **复制单元文件**: 将 `sozu/os-build/systemd/sozu.service` 复制到 `/etc/systemd/system/` 目录：

```bash
sudo cp sozu/os-build/systemd/sozu.service /etc/systemd/system/
```

*   **重新加载 Systemd**:

```bash
sudo systemctl daemon-reload
```

*   **启动服务**:

```bash
sudo systemctl start sozu.service
```

*   **开机自启**:

```bash
sudo systemctl enable sozu.service
```

## 4. Sozu 配置

Sozu 的核心配置通过 `config.toml` 文件进行。它包含三个主要部分：`global` 参数、协议定义 (如 `https`, `http`, `tcp`) 和 `clusters` 部分。

### 4.1 配置文件 (`config.toml`) 结构概述

Sozu 配置文件的示例如下：

```toml
# 全局参数
command_socket = "./command_folder/sock"
saved_state = "./state.json"
log_level = "info"
log_target = "stdout"
worker_count = 2
handle_process_affinity = false
max_connections = 500
buffer_size = 16384
activate_listeners = true

# 监听器配置
[[listeners]]
protocol = "http"
address = "0.0.0.0:8080"
# public_address = "1.2.3.4:80" # 用于日志和转发头
# expect_proxy = false # 期望 PROXY 协议头

# HTTPS 监听器示例
[[listeners]]
protocol = "https"
address = "0.0.0.0:8443"
certificate = "/path/to/certificate.pem"
key = "/path/to/key.pem"
certificate_chain = "/path/to/certificate_chain.pem"
# tls_versions = ["TLS_V12", "TLS_V13"]
# answer_404 = "/path/to/my-404-answer.http" # 自定义错误页面

# 集群配置
[clusters]
[clusters.MyWebsiteCluster]
protocol = "http" # https 代理也使用 http 协议
# load_balancing_policy="roundrobin" # 负载均衡策略: "roundrobin" (默认) 或 "random"
# https_redirect = true # 强制将 http 流量重定向到 https

frontends = [
  { address = "0.0.0.0:8080", hostname = "your_domain.com" },
  # 对于 HTTPS 前端，还需要指定证书和密钥
  { address = "0.0.0.0:8443", hostname = "your_domain.com", certificate = "/path/to/certificate.pem", key = "/path/to/key.pem", certificate_chain = "/path/to/certificate_chain.pem" }
]
backends  = [
  { address = "127.0.0.1:8000" }, # 后端服务器地址
  { address = "127.0.0.1:8001" }
]

# 指标配置
[metrics]
address = "127.0.0.1:8125"
# tagged_metrics = false
# prefix = "sozu"
```

### 4.2 全局参数

全局参数在 `[global]` 部分设置，影响主进程和工作进程：

| 参数                     | 描述                                       | 可选值                                  |
|--------------------------|--------------------------------------------|-----------------------------------------|
| `saved_state`            | Sozu 启动时尝试加载状态的路径              |                                         |
| `log_level`              | 日志级别                                   | `debug`, `trace`, `error`, `warn`, `info` |
| `log_target`             | 日志输出目标                               | `stdout`, `tcp` 或 `udp` 地址             |
| `access_logs_target`     | 访问日志输出目标 (如果激活)                  | `stdout`, `tcp` 或 `udp` 地址             |
| `command_socket`         | Unix 命令套接字路径                        |                                         |
| `worker_count`           | 工作进程数量                               |                                         |
| `handle_process_affinity`| 将工作进程绑定到 CPU 核心                  | `true`, `false`                         |
| `max_connections`        | 最大并发连接数                             |                                         |
| `buffer_size`            | 工作进程使用的请求缓冲区大小 (字节)        |                                         |
| `activate_listeners`     | 自动启动监听器                             | `true`, `false`                         |
| `front_timeout`          | 前端 socket 最大非活动时间                 |                                         |
| `connect_timeout`        | 连接请求的最大非活动时间                   |                                         |
| `request_timeout`        | 请求的最大非活动时间                       |                                         |
| `zombie_check_interval`  | 检查僵尸会话的间隔                         |                                         |
| `pid_file_path`          | 存储 PID 的文件路径                        |                                         |

### 4.3 监听器 (Listeners)

`[[listeners]]` 部分定义了一组接受客户端连接的监听套接字。您可以定义任意数量的监听器。

-   **通用参数**:
```toml
[[listeners]]
protocol = "http" # 或 "https", "tcp"
address = "0.0.0.0:8080"
# public_address = "1.2.3.4:80" # 可选，用于日志和转发头
# expect_proxy = true # 可选，配置客户端套接字接收 PROXY 协议头
```
-   **HTTP 和 HTTPS 监听器特定选项 (自定义错误页面)**:
        您可以为 HTTP 和 HTTPS 监听器定义自定义响应，如 404 Not Found, 503 Service Unavailable 等。这些响应可以是纯文本文件，其中包含 HTML 和一些模板变量 (如 `%REQUEST_ID%`)。

```toml
# 当 Sozu 不知晓所请求的域名或路径时发送 404 响应
answer_404 = "/path/to/my-404-answer.http"
# 当没有可用的后端服务器时发送 503 响应
answer_503 = "/path/to/my-503-answer.http"
```
-   **HTTPS 监听器特定选项**:
```toml
# 支持的 TLS 版本。可选值:"SSL_V2", "SSL_V3", "TLS_V12", "TLS_V13"。默认为 "TLS_V12" 和 "TLS_V13"。
 tls_versions = ["TLS_V12", "TLS_V13"]
# 定义 sticky session cookie 的名称，如果集群激活了 sticky_session。默认为 "SOZUBALANCEID"。
sticky_name = "SOZUBALANCEID"
```
基于 Rustls 的 HTTPS 监听器特定选项：
```toml
cipher_list = [
    # TLS 1.3 密码套件
    "TLS13_AES_256_GCM_SHA384",
    # ... 其他密码套件
]
```

### 4.4 集群 (Clusters)

在 `[clusters]` 部分声明您的集群列表：

```toml
[clusters]
[clusters.MyWebsiteCluster]
protocol = "http" # 或 "tcp"。HTTPS 代理也使用 http 协议
# load_balancing_policy="roundrobin" # 负载均衡策略: "roundrobin" (默认) 或 "random"
# https_redirect = true # 强制将 http 流量重定向到 https
# sticky_session = true # 可选，启用 sticky session

frontends = [
  { address = "0.0.0.0:8080", hostname = "your_domain.com" },
  # 对于 HTTPS 前端，还需要指定证书和密钥
  { address = "0.0.0.0:8443", hostname = "your_domain.com", certificate = "/path/to/certificate.pem", key = "/path/to/key.pem", certificate_chain = "/path/to/certificate_chain.pem" }
]
backs  = [
  { address = "127.0.0.1:8000" }, # 后端服务器地址
  { address = "127.0.0.1:8001" }
]
```

### 4.5 指标 (Metrics)

Sozu 通过 UDP 套接字向其他网络组件报告其状态，并实现了 `statsd` 协议。

配置方式：
```toml
[metrics]
address = "127.0.0.1:8125" # statsd 服务的地址和端口
# tagged_metrics = false # 使用 InfluxDB 的 statsd 协议以添加标签
# prefix = "sozu" # 指标键前缀
```

### 4.6 PROXY 协议

PROXY 协议用于在代理链中传递客户端的真实 IP 地址和端口信息。Sozu 支持 PROXY 协议版本 2。

-   **配置 Sozu 期望 PROXY 协议头**:
```toml
[[listeners]]
address = "0.0.0.0:80"
expect_proxy = true
```
这使得客户端连接在读取任何数据之前先接收 PROXY 协议头。

-   **配置 Sozu 向后端发送 PROXY 协议头**:
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
**注意**: 仅适用于 TCP 集群 (HTTP 和 HTTPS 代理将使用转发头)。

-   **配置 Sozu 转发 PROXY 协议头**:
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
Sozu 将接收客户端连接的 PROXY 协议头，验证其有效性，然后将其转发给上游后端。这允许代理链在不丢失客户端连接信息的情况下工作。

**注意**: 仅适用于 TCP 集群。

## 5. 动态后端管理（服务发现）

在 Docker 等动态环境中，后端服务的 IP 地址可能会频繁变化。尽管 `config.toml` 在 Sozu 启动时解析 IP 地址，但您可以使用 Sozu 提供的命令行工具（CLI）来**动态地添加、更新或移除后端**，而无需重启 Sozu。

这个过程需要一个外部机制（例如一个简单的脚本、服务发现工具或自定义程序）来：

*   **监控**您的后端服务（例如，通过 Docker API 监听容器的启动/停止事件，或者查询 Docker 的内置 DNS 服务）。
*   **获取**后端服务的当前 IP 地址和端口。
*   **使用 `sozu` CLI 工具**向运行中的 Sozu 实例发送命令。

以下是使用 `sozu` CLI 管理后端的示例：

### 5.1 启动 Sozu 以接受命令

确保您的 `config.toml` 中 `command_socket` 配置正确，以便 CLI 工具可以连接到运行中的 Sozu 实例。例如：

```toml
command_socket = "/var/lib/sozu/command.sock"
```

### 5.2 查找 Docker 容器的 IP 地址

假设您的 Docker 网络名为 `my-network`，后端服务名为 `my-backend-app`，您可以使用 `docker inspect` 命令来获取其 IP 地址。

```bash
# 获取单个运行中容器的 IP 地址
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' my-backend-app

# 假设您的后端服务暴露在 8080 端口
export BACKEND_IP=$(docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' my-backend-app)
export BACKEND_PORT=8080
export BACKEND_ADDRESS="${BACKEND_IP}:${BACKEND_PORT}"
```
在 `docker-compose` 环境中，如果服务在同一个 `user-defined bridge network` 下，服务名通常可以直接作为主机名解析。但在 sozu 启动时解析 IP 的情况下，您仍然需要先获取 IP。

### 5.3 动态添加后端

使用 `sozu backend add` 命令将新的后端添加到指定的集群。

```bash
# 假设您的 Sozu 配置文件路径是 /etc/sozu/config.toml
# 并且该配置文件中已经设置了 command_socket
# 假设您的集群 ID 是 'komo'
CONFIG_PATH="/etc/sozu/config.toml"
CLUSTER_ID="komo"
NEW_BACKEND_ID="backend-1" # 为新后端提供一个唯一的ID
NEW_BACKEND_ADDRESS="172.17.0.5:9120" # 替换为您的后端实际 IP:端口

sozu --config "${CONFIG_PATH}" backend add --id "${CLUSTER_ID}" --backend-id "${NEW_BACKEND_ID}" --address "${NEW_BACKEND_ADDRESS}"
```
-   `--config`: 指定 Sozu 配置文件的路径，其中应包含 `command_socket` 的路径。
-   `backend add`: 表示要执行添加后端的操作。
-   `--id`: 指定要添加后端的集群 ID。
-   `--backend-id`: 为此后端实例提供一个唯一的标识符。
-   `--address`: 指定后端服务器的 IP 地址和端口。

### 5.4 动态移除后端

使用 `sozu backend remove` 命令从指定的集群中移除后端。

```bash
# 假设您的 Sozu 配置文件路径是 /etc/sozu/config.toml
CONFIG_PATH="/etc/sozu/config.toml"
CLUSTER_ID="komo"
OLD_BACKEND_ID="backend-1" # 要移除的后端的ID
OLD_BACKEND_ADDRESS="172.17.0.5:9120" # 替换为要移除的后端实际 IP:端口

sozu --config "${CONFIG_PATH}" backend remove --id "${CLUSTER_ID}" --backend-id "${OLD_BACKEND_ID}" --address "${OLD_BACKEND_ADDRESS}"
```
-   `--config`: 指定 Sozu 配置文件的路径。
-   `backend remove`: 表示要执行移除后端的操作。
-   `--id`: 指定要移除后端的集群 ID。
-   `--backend-id`: 要移除的后端的ID。
-   `--address`: 要移除的后端服务器的 IP 地址和端口。

### 5.5 查看 Sōzu 工作进程状态

您可以使用 `sozu status` 命令来查看 Sōzu 工作进程（workers）的运行状态。

```bash
# 假设您的 Sozu 配置文件路径是 /etc/sozu/config.toml
CONFIG_PATH="/etc/sozu/config.toml"
sozu --config "${CONFIG_PATH}" status --json
```
这将返回一个 JSON 格式的当前工作进程状态。要查看详细的集群、前端和后端配置，请参阅下一节。

### 5.6 自动化脚本（概念）

要实现真正的服务发现，您需要编写一个持续运行的脚本（例如使用 Python、Bash 或 Go），该脚本：

*   **周期性**地或通过 **事件监听** Docker API 来检测后端容器的变化。
*   **比较**当前 Sozu 中的后端列表与 Docker 中实际运行的后端列表。
*   **执行** `sozu backend add` 或 `sozu backend remove` 命令来同步这两个列表。

这个脚本将作为您服务发现方案中的“控制平面”。

### 5.7 查看动态添加的前端和后端

当您通过命令行动态添加了 `frontend` 或 `backend` 后，您可能想验证它们是否已成功加载。Sōzu 的路由结构是 `frontend` -> `cluster` -> `backend`。您可以通过以下两步来追踪和查看这些动态添加的数据：

*   **查找 `frontend` 对应的 `cluster ID`**

    使用 `sozu frontend list --json` 命令可以列出所有已配置的前端。在返回的 JSON 数据中，找到您关心的 `frontend`，并记下其所属的 `cluster_id`。

```bash
# 将 <您的/config.toml/路径> 替换为您的配置文件路径
sozu --config <您的/config.toml/路径> frontend list --json
```

*   **根据 `cluster ID` 查找其包含的 `backend`**

    获取到 `cluster ID` 后，使用 `sozu cluster list --id <CLUSTER_ID> --json` 命令来查看该特定集群的详细信息，其中包括了它所包含的所有 `backend` 列表。

```bash
# 替换 <您的/config.toml/路径> 和 <您的_CLUSTER_ID>
sozu --config <您的/config.toml/路径> cluster list --id <您的_CLUSTER_ID> --json
```

通过这个两步流程，您可以清晰地看到从 `frontend` 到 `backend` 的完整映射关系，并确认您的动态配置已生效。

希望这份教程能帮助您更好地理解和使用 Sozu！
